# Autorisation avec Bouncer (abilities et policies)

Authentification = « qui es-tu ? » (voir `auth-starter.md`). Autorisation = « que peux-tu faire ? ». Validé sur le kit Vue v7 : installation, policy, contrôleur et tests (13 tests verts).

Documentation officielle : `official-docs.md` (Auth > Authorization).

## 1. Installation

```bash
node ace add @adonisjs/bouncer
```

Ce que la commande a réellement modifié :

- `app/abilities/main.ts` créé (contient une ability d'exemple `editUser`) ;
- `app/middleware/initialize_bouncer_middleware.ts` créé et enregistré dans `start/kernel.ts` (`router.use`) ;
- `adonisrc.ts` : `bouncer_provider`, `@adonisjs/bouncer/commands` et le hook `indexPolicies()` dans `hooks.init` ;
- alias `#policies/*` et `#abilities/*` déjà présents dans `package.json` (`imports`).

Le dossier `app/policies/` n'existe pas avant le premier `make:policy`.

## 2. Abilities (peu de règles, simples)

```ts
// app/abilities/main.ts
import User from '#models/user'
import Todo from '#models/todo'
import { Bouncer } from '@adonisjs/bouncer'

export const editTodo = Bouncer.ability((user: User, todo: Todo) => user.id === todo.userId)
```

Dans un contrôleur, `bouncer` est déjà lié à l'utilisateur connecté : **ne pas passer l'utilisateur**.

```ts
import { editTodo } from '#abilities/main'

async update({ bouncer, params }: HttpContext) {
  const todo = await Todo.findOrFail(params.id)
  await bouncer.authorize(editTodo, todo)
  // ...
}
```

## 3. Policies (une classe par ressource)

```bash
node ace make:policy todo                       # app/policies/todo_policy.ts (classe vide, imports pré-remplis)
node ace make:policy todo view edit delete      # pré-crée les méthodes
```

```ts
import User from '#models/user'
import Todo from '#models/todo'
import { BasePolicy } from '@adonisjs/bouncer'
import type { AuthorizerResponse } from '@adonisjs/bouncer/types'

export default class TodoPolicy extends BasePolicy {
  edit(user: User, todo: Todo): AuthorizerResponse {
    return user.id === todo.userId
  }

  delete(user: User, todo: Todo): AuthorizerResponse {
    return user.id === todo.userId
  }
}
```

Conseil officiel : une méthode par action même si la logique est identique (`edit` / `delete`), pour pouvoir les faire évoluer séparément.

Utilisation (version validée) :

```ts
import TodoPolicy from '#policies/todo_policy'

async update({ bouncer, params, request, response }: HttpContext) {
  const payload = await request.validateUsing(updateTodoValidator)
  const todo = await Todo.findOrFail(params.id)
  await bouncer.with(TodoPolicy).authorize('edit', todo)
  todo.merge(payload)
  await todo.save()
  return response.redirect().back()
}
```

Les policies profitent de l'injection de dépendances (constructeur avec `@inject()`).

Référence par nom (barrel `.adonisjs/server/policies.ts`, vérifié : `TodoPolicy: () => import('#policies/todo_policy')`) :

```ts
await bouncer.with('TodoPolicy').authorize('edit', todo)
```

Le barrel est généré par `indexPolicies()` au démarrage de `node ace serve` ou du build, et sert aussi à `initialize_bouncer_middleware.ts` (`import { policies } from '#generated/policies'`).

## 4. Les quatre méthodes de vérification

| Méthode | Comportement |
|---|---|
| `bouncer.allows(ability, ...args)` | `true` si autorisé |
| `bouncer.denies(ability, ...args)` | `true` si refusé |
| `bouncer.authorize(ability, ...args)` | lève `E_AUTHORIZATION_FAILURE` si refusé |
| `bouncer.inspect(ability, ...args)` | renvoie un `AuthorizationResponse` (`authorized`, `message`, `status`) |

Pour renvoyer 404 au lieu de 403 (masquer l'existence de la ressource), retourner `AuthorizationResponse.deny('Todo not found', 404)` depuis l'ability ou la méthode de policy.

## 5. Ce que devient la réponse HTTP (vérifié par test)

`authorize` lève une exception qui se convertit selon la négociation de contenu :

| Requête | Réponse observée |
|---|---|
| Soumission de formulaire HTML (`POST/PATCH/DELETE`) avec session | **302**, retour à la page précédente (message flashé) |
| `Accept: application/json` | **403** avec un tableau `errors` |
| Navigation HTML `GET` | texte brut + statut **403** (doc) |

Un test qui attend `403` sur une soumission de formulaire échoue (`expected 302 to equal 403`). Voir `testing.md`, section 7.

## 6. Policy vs filtre dans la requête

Deux manières valables d'isoler les données d'un utilisateur :

```ts
// A. filtre (renvoie 404 pour une ressource d'autrui, aucune fuite d'existence)
Todo.query().where('userId', user.id).where('id', params.id).firstOrFail()

// B. policy (403 explicite, centralisé, réutilisable dans Edge/Inertia)
const todo = await Todo.findOrFail(params.id)
await bouncer.with(TodoPolicy).authorize('edit', todo)
```

Le tutoriel utilise A. La B est préférable quand la règle dépasse « le propriétaire » (rôles, états, partage). Dans les deux cas, **tester le refus** : un test vérifie le code de réponse **et** que la donnée n'a pas changé (la mutation du filtre `userId` a bien fait échouer le test, preuve qu'il protège).

## 7. Partager les permissions avec le frontend

Bouncer n'est pas visible côté Vue par défaut. Exposer un booléen depuis le contrôleur (ou un transformer) :

```ts
return inertia.render('todos/index', {
  todos: TodoTransformer.transform(todos),
  can: { create: await bouncer.allows(createTodo) },
})
```

Ne jamais se contenter de masquer un bouton : la vérification serveur reste obligatoire.

## 8. Tests

```ts
const me = await UserFactory.create()
const other = await UserFactory.create()
const todo = await TodoFactory.merge({ userId: other.id }).create()

const response = await client
  .visit('todos.update', { id: todo.id })
  .loginAs(me)
  .withCsrfToken()
  .header('accept', 'application/json')
  .redirects(0)
  .json({ status: 'done' })

response.assertStatus(403)
await todo.refresh()
assert.equal(todo.status, 'pending')
```

## 9. Vérification

```bash
node ace list | grep make:policy
node ace serve --hmr          # régénère .adonisjs/server/policies.ts (barrel des policies)
npm run typecheck
node ace test functional
```

Ne pas supprimer `indexPolicies()` de `adonisrc.ts` : il génère le barrel `#generated/policies` qui permet de référencer une policy par son nom.
