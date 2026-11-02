# Tuyau (routes typées) et Transformers (données typées)

Deux mécanismes du kit v7, tous deux **générés** par les hooks de `adonisrc.ts`. Ils sont absents du tutoriel mais le kit les utilise partout.

## 1. Hooks de génération (`adonisrc.ts`)

```ts
hooks: {
  init: [
    indexEntities({ transformers: { enabled: true, withSharedProps: true } }),
    indexPages({ framework: 'vue3' }),
    generateRegistry(),
  ],
  buildStarting: [() => import('@adonisjs/vite/build_hook')],
},
```

| Hook | Génère |
|---|---|
| `indexEntities` | `.adonisjs/server/controllers.ts` (objet `controllers`), `events.ts`, `listeners.ts`, et les types `Data` (transformers) |
| `indexPages` | `.adonisjs/server/pages.d.ts` : liste des pages `inertia/pages/**` pour typer `inertia.render('...')` |
| `generateRegistry` | `.adonisjs/client/registry/*` : registre des routes pour Tuyau |

Les fichiers de `.adonisjs/` sont régénérés au démarrage de `node ace serve` ; ne pas les éditer. Ils sont versionnés dans le kit (laisser ainsi ou les ajouter à `.gitignore` selon la politique de l'équipe).

## 2. Routes : `#generated/controllers`

```ts
import { controllers } from '#generated/controllers'

router.get('signup', [controllers.NewAccount, 'create'])
router.resource('todos', controllers.Todos).only(['index', 'store', 'update', 'destroy'])
```

- `controllers.<Nom>` est un import paresseux généré depuis `app/controllers/*_controller.ts` (`todos_controller.ts` → `controllers.Todos`). Un nouveau contrôleur apparaît automatiquement après `node ace make:controller` ou au prochain démarrage.
- Le nom de route est déduit du contrôleur et de la méthode (`controllers.Session` + `create` → `session.create`) ; sinon `.as('nom')`.
- Vérifié : dans un groupe `router.group(...).prefix('todos')` avec `router.get('/', [controllers.Todos, 'index'])`, les noms `todos.index` et `todos.store` sont bien générés (`node ace list:routes`). Le prefix du tutoriel est donc compatible avec `toRoute('todos.index')` et `<Link route="todos.index">`.
- `controllers.Todos` est ajouté à `.adonisjs/server/controllers.ts` dès qu'un `todos_controller.ts` existe et que `list:routes` ou `serve` est lancé.

## 3. Client Tuyau côté frontend

`inertia/client.ts` :

```ts
import { registry } from '@generated/registry'
import { createTuyau } from '@tuyau/core/client'

export const client = createTuyau({ baseUrl: '/', registry })
export const urlFor = client.urlFor
```

- L'alias `@generated` (voir `vite.config.ts`) pointe vers `.adonisjs/client/`.
- `inertia/app.ts` enveloppe l'application dans `<TuyauProvider :client="client">` (`@adonisjs/inertia/vue`) : indispensable pour que `<Link route>` et `<Form route>` fonctionnent. Ne pas le retirer en simplifiant `app.ts`.
- `urlFor('todos.update', { id })` construit une URL typée en dehors des composants. Il est préférable à `` `/todos/${id}` `` écrit en dur.
- `<Link>`, `<Form>`, `useRouter()` : voir `inertia-layouts.md`.

## 4. Transformers : contrat serveur → frontend

Un transformer décrit **exactement** ce qui part vers le navigateur (et génère le type TypeScript correspondant). Exemple du kit :

```ts
// app/transformers/user_transformer.ts
import type User from '#models/user'
import { BaseTransformer } from '@adonisjs/core/transformers'

export default class UserTransformer extends BaseTransformer<User> {
  toObject() {
    return this.pick(this.resource, ['id', 'fullName', 'email', 'createdAt', 'updatedAt', 'initials'])
  }
}
```

Créer :

```bash
node ace make:transformer todo          # app/transformers/todo_transformer.ts
node ace make:model Todo --transformer  # génère aussi le transformer
```

API (vérifiée dans `@adonisjs/http-transformers`) :

- `this.resource` : le modèle ; `this.pick(obj, [...])` / `this.omit(obj, [...])` : sélection de champs.
- `Transformer.transform(model)` → `Item` ; `Transformer.transform(models)` → `Collection` ; `.depth(n)` limite la profondeur des relations ; `.useVariant('nom')` choisit une variante.
- `this.whenLoaded(this.resource.category)` : n'inclut la relation que si elle a été `preload`ée (sinon champ omis).
- Variantes : méthodes supplémentaires (`forDetailedView()`, éventuellement `async`) à côté de `toObject()` (variante par défaut).

Exemple pour les todos :

```ts
import type Todo from '#models/todo'
import { BaseTransformer } from '@adonisjs/core/transformers'
import CategoryTransformer from '#transformers/category_transformer'

export default class TodoTransformer extends BaseTransformer<Todo> {
  toObject() {
    return {
      ...this.pick(this.resource, ['id', 'title', 'description', 'status', 'createdAt']),
      category: CategoryTransformer.transform(this.whenLoaded(this.resource.category)),
    }
  }
}
```

Dans le contrôleur, passer le résultat directement à Inertia (l'adaptateur sait sérialiser `Item`, `Collection` et `Paginator`, pas besoin de `serialize()`) :

```ts
async index({ auth, inertia }: HttpContext) {
  const todos = await Todo.query().where('userId', auth.user!.id).preload('category')
  return inertia.render('todos/index', { todos: TodoTransformer.transform(todos) })
}
```

> Le tutoriel construit à la main des objets `{ id, title, ... createdAt: t.createdAt?.toISO() }` dans chaque méthode puis les recopie pour `transmit.broadcast`. Un transformer évite cette duplication et garantit la même forme partout.

## 5. Types côté frontend : `Data`

Le kit génère `.adonisjs/client/data.d.ts` :

```ts
export namespace Data {
  export type User = InferData<UserTransformer>
  export type SharedProps = InferSharedProps<InertiaMiddleware>
  export type FlashMessages = InferFlashData<InertiaMiddleware>
}
```

Un `TodoTransformer` ajoute automatiquement `Data.Todo` (et `Data.Todo.Variants['nom']`). Utilisation :

```vue
<script setup lang="ts">
import type { Data } from '@generated/data'
defineProps<{ todos: Data.Todo[] }>()
</script>
```

- `inertia/types.ts` branche `Data.SharedProps` et `Data.FlashMessages` sur `@inertiajs/core` : `usePage().props.user` et `usePage().flash` sont donc typés sans générique.
- Pour ajouter une prop **partagée** (ex. `todoCount`), la retourner dans `share()` de `app/middleware/inertia_middleware.ts`, avec `ctx.inertia.always(...)` si elle doit être présente à chaque visite ; le type est inféré automatiquement.
- Ne jamais renvoyer un modèle Lucid brut : mots de passe et colonnes internes fuiraient vers le navigateur. Toujours passer par un transformer (comme `UserTransformer` qui exclut `password`).

## 6. Transformers et `transmit.broadcast`

`broadcast` attend un objet JSON simple. `ctx.serialize` du kit (`providers/api_provider.ts`) enveloppe sous `data` ; pour un payload brut :

```ts
const payload = await serialize.withoutWrapping(TodoTransformer.transform(todo))
transmit.broadcast(`todos/${user.id}`, { action: 'created', todo: payload })
```

(avec `{ auth, serialize }: HttpContext`). **À tester sur le projet** : la signature exacte de `serialize.withoutWrapping` et la forme du résultat n'ont été lues que dans les types ; vérifier que le payload reçu par le client correspond au type `Data.Todo` attendu. Alternative sûre : un helper `toTodoDto(todo)` partagé par le contrôleur et le broadcast.

## 7. `providers/api_provider.ts`

Provider du kit qui ajoute `ctx.serialize(...)` : sérialiseur `ApiSerializer` qui enveloppe la réponse sous `data` et gère la pagination Lucid. Utile pour des routes JSON (`return serialize(Transformer.transform(x))`) ; inutile pour `inertia.render`.

## 8. Vérification

```bash
node ace serve --hmr           # régénère .adonisjs/*
node ace list:routes           # noms de routes disponibles
npm run typecheck              # tsc + vue-tsc : détecte un nom de route ou un type Data invalide
```
