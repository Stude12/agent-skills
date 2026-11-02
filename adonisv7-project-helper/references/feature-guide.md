# Guide de création de fonctionnalité

Ce guide aide l utilisateur à concevoir et implémenter une fonctionnalité dans une application AdonisJS v7, en prenant comme exemple la création d un module de gestion de todos.

## Objectif

Réaliser une fonctionnalité structurée avec un modèle de données, des validateurs, un contrôleur CRUD, des routes protégées et un frontend Inertia.

## Prérequis

- Application AdonisJS v7 installée
- Authentification configurée si la fonctionnalité est liée à un utilisateur
- Connaissance des routes, contrôleurs et modèles AdonisJS

## Étapes de création

### 1. Définir le périmètre de la fonctionnalité

- Quelle donnée représente la fonctionnalité ?
- Quelles actions l utilisateur peut-il effectuer ?
- Quel type d interface est requis ?
- Faut-il stocker des données en base ?
- Y a-t-il des relations entre entités (par exemple un todo appartient à un utilisateur, ou une commande appartient à un client) ?
- La feature doit-elle gérer plusieurs langues ou du contenu traduit ?

### 1.1. Ajouter i18n si nécessaire

Si l utilisateur souhaite gérer plusieurs langues, vérifier si `@adonisjs/i18n` est déjà installé et configuré.

- Demander à l utilisateur s il souhaite activer la gestion des langues.
- Si ce n est pas déjà en place, installer et configurer i18n selon la documentation :
  <https://docs.adonisjs.com/guides/digging-deeper/i18n>
- Proposer des entrées de traduction pour les libellés, les messages d erreur et les textes de l interface liés à la feature.
- Ajouter des fichiers de localisation dans `public/lang/<locale>.json` ou le format utilisé par le projet.

### 2. Migration, seeders et modèle

Tout le détail est dans **`database-migrations.md`** : migration de la table `todos` (sections 2), schéma auto-généré et modèle `Todo` qui n'ajoute que ses relations (section 3), seeders (section 5).

Résumé de l'ordre à respecter :

```bash
node ace make:migration todos     # écrire la migration
node ace migration:run            # applique ET génère database/schema.ts (TodoSchema)
node ace make:model Todo          # SEULEMENT ensuite : le modèle importe #database/schema
```

### 3. Ajouter des validateurs

Le kit utilise `vine.create({...})` (voir `app/validators/user.ts`) ; l'ancienne forme `vine.compile(vine.object({...}))` reste valide mais n'est plus celle du starter. Créer un validateur pour la création. `description` doit être `.nullable().optional()` : avec `.nullable()` seul, un formulaire qui n'envoie que `title` est refusé (`The description field must be defined`, vérifié) :

```bash
node ace make:validator todos/create
```

```ts
import vine from '@vinejs/vine'

export const createTodoValidator = vine.create({
  title: vine.string().trim().minLength(1).maxLength(255),
  description: vine.string().trim().nullable().optional(),
})
```

Puis un validateur pour la mise à jour :

```bash
node ace make:validator todos/update
```

```ts
import vine from '@vinejs/vine'

export const updateTodoValidator = vine.create({
  title: vine.string().trim().minLength(1).maxLength(255).optional(),
  description: vine.string().trim().nullable().optional(),
  status: vine.enum(['pending', 'in_progress', 'done'] as const).optional(),
})
```

### 4. Implémenter le contrôleur CRUD

> Variante recommandée avec le kit v7 : ne pas construire à la main les objets renvoyés à `inertia.render`. Créer `node ace make:transformer todo` et renvoyer `TodoTransformer.transform(todos)` (voir `tuyau-transformers.md`). L'exemple ci-dessous garde la forme explicite du tutoriel.

Générez un contrôleur resource :

```bash
node ace make:controller Todos --resource
```

Implémentez au minimum `index`, `store`, `update` et `destroy` :

```ts
import type { HttpContext } from '@adonisjs/core/http'
import Todo from '#models/todo'
import { createTodoValidator } from '#validators/todos/create'
import { updateTodoValidator } from '#validators/todos/update'

export default class TodosController {
  async index({ auth, inertia }: HttpContext) {
    const user = auth.use('web').user!
    const todos = await Todo.query().where('userId', user.id).orderBy('createdAt', 'desc')
    return inertia.render('todos/index', {
      todos: todos.map((t) => ({
        id: t.id,
        title: t.title,
        description: t.description,
        status: t.status,
        createdAt: t.createdAt?.toISO(),
      })),
    })
  }

  async store({ auth, request, response }: HttpContext) {
    const user = auth.use('web').user!
    const payload = await request.validateUsing(createTodoValidator)
    await Todo.create({
      userId: user.id,
      title: payload.title,
      description: payload.description ?? null,
    })
    return response.redirect().back()
  }

  async update({ auth, params, request, response }: HttpContext) {
    const user = auth.use('web').user!
    const payload = await request.validateUsing(updateTodoValidator)
    const todo = await Todo.query().where('userId', user.id).where('id', params.id).firstOrFail()
    todo.merge(payload)
    await todo.save()
    return response.redirect().back()
  }

  async destroy({ auth, params, response }: HttpContext) {
    const user = auth.use('web').user!
    const todo = await Todo.query().where('userId', user.id).where('id', params.id).firstOrFail()
    await todo.delete()
    return response.redirect().back()
  }
}
```

### 5. Définir les routes protégées

Ajoutez un groupe de routes dans `start/routes.ts` :

```ts
router
  .group(() => {
    router.get('/', [controllers.Todos, 'index'])
    router.post('/', [controllers.Todos, 'store'])
    router.patch('/:id', [controllers.Todos, 'update'])
    router.delete('/:id', [controllers.Todos, 'destroy'])
  })
  .prefix('todos')
  .use(middleware.auth())
```

Vérifiez les routes avec :

```bash
node ace list:routes
```

### 6. Créer le frontend Inertia

Pour une page feature, utilisez une page Vue qui reçoit les données en props :

- `inertia/pages/feature/index.vue`

Elle peut contenir :

- visualisation des données relatives à la feature
- création de nouveaux éléments de la feature
- modification des éléments existants
- suppression des éléments
- filtres ou recherche spécifiques à la feature

Cette structure s applique bien à Vue/Inertia, mais le même schéma peut être utilisé pour React ou Svelte avec des hooks ou stores équivalents.

Structure de dossiers recommandée :

- `start/routes.ts` — définition des routes protégées
- `app/controllers/FeatureController.ts` — logique serveur CRUD
- `app/models/Feature.ts` — modèle de données et relations
- `app/validators/feature/create.ts` — validation de création
- `app/validators/feature/update.ts` — validation de mise à jour
- `inertia/pages/feature/index.vue` — page Vue/Inertia

Exemple de structure de page :

```vue
<script setup lang="ts">
import { useForm, router } from '@inertiajs/vue3'
import { computed, reactive, ref } from 'vue'

interface TodoItem {
  id: number
  title: string
  description: string | null
  status: 'pending' | 'in_progress' | 'done'
  createdAt?: string
}

const props = defineProps<{ todos: TodoItem[] }>()
const form = useForm({ title: '', description: '' })
const showCreate = ref(false)
const query = ref('')
const sort = reactive<{ column: 'title' | 'status' | 'createdAt'; descending: boolean }>({
  column: 'createdAt',
  descending: true,
})

const submit = () => {
  form.post('/todos', {
    onSuccess: () => {
      form.reset('title', 'description')
      showCreate.value = false
    },
  })
}

const onStatusChange = (id: number, e: Event) => {
  const value = (e.target as HTMLSelectElement).value as TodoItem['status']
  router.patch(`/todos/${id}`, { status: value })
}

const remove = (id: number) => {
  router.delete(`/todos/${id}`)
}

const filtered = computed(() => {
  const q = query.value.trim().toLowerCase()
  return props.todos.filter(
    (t) =>
      !q || t.title.toLowerCase().includes(q) || (t.description ?? '').toLowerCase().includes(q)
  )
})

const displayedTodos = computed(() => {
  const arr = [...filtered.value]
  const dir = sort.descending ? -1 : 1
  arr.sort((a, b) => {
    if (sort.column === 'title') return a.title.localeCompare(b.title) * dir
    if (sort.column === 'status') return a.status.localeCompare(b.status) * dir
    const da = a.createdAt ? Date.parse(a.createdAt) : 0
    const db = b.createdAt ? Date.parse(b.createdAt) : 0
    return (da - db) * dir
  })
  return arr
})

const toggleSort = (col: 'title' | 'status' | 'createdAt') => {
  if (sort.column === col) sort.descending = !sort.descending
  else {
    sort.column = col
    sort.descending = false
  }
}
</script>
```

### 7. Ajuster les redirections et le flux utilisateur

Si la fonctionnalité est la page principale après connexion, mettre à jour (dans le kit v7, `SessionController.store` et `NewAccountController.store` redirigent vers `dashboard`, et `GuestMiddleware.redirectTo` vaut `'/'`) :

- `app/middleware/guest_middleware.ts` → `redirectTo = '/todos'`
- `app/controllers/session_controller.ts` et `new_account_controller.ts` → `response.redirect().toRoute('todos.index')`
- supprimer ou rediriger la route/page `dashboard` si elle devient inutile, ainsi que l'entrée correspondante dans la navigation (`inertia/layouts/app.vue`).

> Les noms de routes (`todos.index`) sont générés automatiquement à partir du contrôleur ; vérifier avec `node ace list:routes`.

### 7.1. Évolutions courantes (références dédiées)

- **Remember me** : voir `auth-starter.md`, section 6.
- **Catégories (migrations + seeders + relation)** : voir `database-migrations.md`, sections 4 et 5.
- **Sérialisation propre vers le frontend et le temps réel** : voir `tuyau-transformers.md` (un `TodoTransformer` remplace les objets construits à la main dans `index`, `store`, `update` et dans les `transmit.broadcast()`).
- **Temps réel entre onglets** : voir `sse-guide.md`.
- **Layouts, Header, Footer** : voir `inertia-layouts.md`.

### 8. Vérification

- Vérifiez que la migration est appliquée
- Contrôlez les routes via `node ace list:routes`
- Testez manuellement la création, la mise à jour et la suppression
- Vérifiez la validation côté serveur
- Assurez-vous que les données sont liées à l utilisateur connecté

### 9. Tests

La mise en place (plugins, base de test, factories), les exemples validés, les pièges (302 vs 403, CSRF, Inertia, Playwright) et les commandes sont dans **`testing.md`**. Ici, seulement ce que la fonctionnalité Todos doit couvrir : pour **chaque** endpoint, **un cas passant et un cas d'erreur**.

- `GET /todos` : ne liste que les todos de l'utilisateur connecté (créer deux utilisateurs) ; un visiteur est redirigé vers `/login` ;
- `POST /todos` : création, titre vide refusé, requête sans jeton CSRF refusée ;
- `PATCH /todos/:id` : changement de statut, **et** refus pour la ressource d'un autre utilisateur (vérifier le code de réponse **et** que la donnée n'a pas changé) ;
- `DELETE /todos/:id` : suppression, refus pour un autre utilisateur ;
- navigateur : connexion réussie et refusée, création valide puis invalide, accès non autorisé (`testing.md`, section 8).

```bash
node ace make:test todos --suite=functional
node ace test unit functional
```

## Exemple de structure pour la fonctionnalité

- `database/migrations/..._create_todos_table.ts`
- `database/schema.ts` (généré après migration)
- `app/models/todo.ts`
- `app/validators/todos/create.ts`
- `app/validators/todos/update.ts`
- `app/controllers/todos_controller.ts`
- `start/routes.ts`
- `inertia/pages/todos/index.vue`
- `app/middleware/guest_middleware.ts`

## Conseils

- Gardez la logique métier simple et testable
- Utilisez des validateurs distincts pour chaque action
- Protégez toutes les routes sensibles avec `auth`
- Distinguez bien la donnée (`model`), la logique (`controller`) et la présentation (`frontend`)
- Documentez la fonctionnalité dans `SKILL.md` et dans une référence dédiée si nécessaire
