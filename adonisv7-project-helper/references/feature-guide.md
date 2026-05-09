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

### 2. Créer la migration

Pour le module Todos, générez une migration :

```bash
node ace make:migration todos
```

Exemple de migration :

```ts
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'todos'

  async up() {
    this.schema.createTable(this.tableName, (table) => {
      table.increments('id').notNullable()
      table.integer('user_id').unsigned().notNullable().references('users.id').onDelete('CASCADE')
      table.string('title').notNullable()
      table.text('description').nullable()
      table.enum('status', ['pending', 'in_progress', 'done']).notNullable().defaultTo('pending')
      table.timestamp('created_at').notNullable()
      table.timestamp('updated_at').nullable()
    })
  }

  async down() {
    this.schema.dropTable(this.tableName)
  }
}
```

Appliquer la migration :

```bash
node ace migration:run
node ace migration:status
```

### 2.1. Seeders (optionnel)

Si la fonctionnalité nécessite des données initiales ou des exemples de test, créez un seeder :

```bash
node ace make:seeder TodoSeeder
```

Ensuite, ajoutez les enregistrements dans `database/seeders/TodoSeeder.ts` et exécutez :

```bash
node ace db:seed
```

Cela est particulièrement utile pour les fonctionnalités liées aux todos, aux rôles, aux permissions ou aux données de configuration.

### 3. Créer le modèle

Générez le modèle Todo :

```bash
node ace make:model Todo
```

En AdonisJS v7, les colonnes sont définies automatiquement dans le schéma généré, donc le modèle ne contient souvent que les relations :

```ts
import { belongsTo } from '@adonisjs/lucid/orm'
import type { BelongsTo } from '@adonisjs/lucid/types/relations'
import { TodoSchema } from '#database/schema'
import User from '#models/user'

export default class Todo extends TodoSchema {
  @belongsTo(() => User)
  declare user: BelongsTo<typeof User>
}
```

### 4. Ajouter des validateurs

Créer un validateur pour la création :

```bash
node ace make:validator todos/create
```

```ts
import vine from '@vinejs/vine'

export const createTodoValidator = vine.compile(
  vine.object({
    title: vine.string().trim().minLength(1).maxLength(255),
    description: vine.string().trim().nullable(),
  })
)
```

Puis un validateur pour la mise à jour :

```bash
node ace make:validator todos/update
```

```ts
import vine from '@vinejs/vine'

export const updateTodoValidator = vine.compile(
  vine.object({
    title: vine.string().trim().minLength(1).maxLength(255).optional(),
    description: vine.string().trim().nullable().optional(),
    status: vine.enum(['pending', 'in_progress', 'done'] as const).optional(),
  })
)
```

### 5. Implémenter le contrôleur CRUD

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

### 6. Définir les routes protégées

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

### 7. Créer le frontend Inertia

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

### 8. Ajuster les redirections et le flux utilisateur

Si la fonctionnalité est la page principale après connexion, mettez à jour les redirections :

- `app/middleware/guest_middleware.ts` → `redirectTo = '/todos'`
- contrôleurs de session et d inscription → `response.redirect().toRoute('todos.index')`

### 9. Vérification

- Vérifiez que la migration est appliquée
- Contrôlez les routes via `node ace list:routes`
- Testez manuellement la création, la mise à jour et la suppression
- Vérifiez la validation côté serveur
- Assurez-vous que les données sont liées à l utilisateur connecté

### 10. Créer des tests unitaires / API

AdonisJS propose des tests API et unitaires pour valider les fonctionnalités. Utilisez la documentation suivante :

<https://docs.adonisjs.com/guides/testing/api-tests>

1. Générez un fichier de test :

```bash
node ace make:test todos
```

2. Écrivez une suite de tests pour :

- création de ressource via POST (cas passant ET cas d'erreur de validation)
- récupération de la liste via GET (vérifier que seules les données de l'utilisateur sont retournées)
- mise à jour via PATCH (cas passant ET cas d'erreur)
- suppression via DELETE (cas passant ET cas d'erreur)
- autorisation et validation des données

**Obligation** : Pour chaque endpoint, ajoutez au minimum **un cas passant** et **un cas d'erreur**.

3. Exemple simple :

```ts
import test from '@japa/runner'
import Todo from 'App/Models/Todo'

test.group('Todos', (group) => {
  group.setup(async () => {
    await Todo.query().delete()
  })

  test('create a todo', async ({ client }) => {
    const response = await client.post('/todos').json({
      title: 'New todo',
      description: 'Test todo',
    })

    response.assertStatus(302)
  })
})
```

4. Lancez les tests :

```bash
node ace test
```

### 11. Créer des tests d interface (browser)

Pour tester l interface utilisateur, proposez à l utilisateur de choisir entre :

- Japa Browser (tests intégrés AdonisJS) : <https://docs.adonisjs.com/guides/testing/browser-tests>
- Playwright (tests de bout en bout indépendants) : <https://playwright.dev/docs/intro>

Si vous préférez rester dans l écosystème AdonisJS, utilisez Japa Browser pour automatiser les interactions avec les pages, les formulaires et les composants.

**Obligation** : Pour chaque scénario utilisateur important, ajoutez au minimum **un cas passant** (chemin normal) et **un cas d'erreur** (erreur de validation, accès refusé, etc.).

Exemple de workflow :

1. Définir les scénarios importants (connexion, création avec données valides, création avec données invalides, modification, suppression, accès non autorisé)
2. Vérifier les éléments visibles et les messages d erreur
3. Simuler les actions utilisateur sur le frontend (clic, saisie, validation)
4. Exécuter les tests en continu pendant le développement

Ensuite, ajoutez une note pour demander au client quel outil il souhaite utiliser :

- `japa-browser` si vous voulez un support natif AdonisJS
- `playwright` si vous voulez des tests E2E plus larges et multi-plateforme

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
