# Authentification fournie par le kit v7 (session + Lucid)

Le kit `--kit=vue` fournit déjà tout : il s'agit surtout de **vérifier** et de **comprendre**, pas de recréer.

## 1. Fichiers fournis

| Rôle | Fichier |
|---|---|
| Modèle | `app/models/user.ts` |
| Inscription | `app/controllers/new_account_controller.ts` (`create`, `store`) |
| Connexion / déconnexion | `app/controllers/session_controller.ts` (`create`, `store`, `destroy`) |
| Validateurs | `app/validators/user.ts` (`signupValidator`, `loginValidator`) |
| Middlewares | `app/middleware/auth_middleware.ts`, `guest_middleware.ts`, `silent_auth_middleware.ts` |
| Config guard | `config/auth.ts` (guard `web`, session) |
| Pages Vue | `inertia/pages/auth/login.vue`, `signup.vue` |
| Migration | `database/migrations/*_create_users_table.ts` |
| Routes | `start/routes.ts` (groupes `guest()` et `auth()`) |

## 2. Modèle `User` : `compose` + schéma généré

```ts
import { UserSchema } from '#database/schema'
import hash from '@adonisjs/core/services/hash'
import { compose } from '@adonisjs/core/helpers'
import { withAuthFinder } from '@adonisjs/auth/mixins/lucid'

export default class User extends compose(UserSchema, withAuthFinder(hash)) {
  get initials() { /* ... */ }
}
```

- `UserSchema` (dans `database/schema.ts`) est **généré** depuis la base par `node ace migration:run` : colonnes `id`, `fullName`, `email`, `password`, `createdAt`, `updatedAt`. Ne pas l'éditer. Sans ce fichier, `import '#database/schema'` échoue.
- `compose(Base, ...mixins)` : le premier argument est la classe de base, les suivants sont des mixins.
- `withAuthFinder(hash)` ajoute `User.verifyCredentials(email, password)` (lève une exception si identifiants faux), `findForAuth` et le **hachage automatique** du mot de passe à la sauvegarde. Le kit v7 lui passe directement `hash` (pas de `() => hash.use('scrypt')` ni d'options `uids`/`passwordColumnName` comme en v6).
- On n'ajoute que des mixins, des getters (comme `initials`) et des relations.

## 3. Contrôleurs

```ts
// new_account_controller.ts
async store({ request, response, auth }: HttpContext) {
  const { fullName, email, password } = await request.validateUsing(signupValidator)
  const user = await User.create({ fullName, email, password })
  await auth.use('web').login(user)
  response.redirect().toRoute('dashboard')
}

// session_controller.ts
async store({ request, auth, response }: HttpContext) {
  const { email, password } = await request.validateUsing(loginValidator)
  const user = await User.verifyCredentials(email, password)
  await auth.use('web').login(user)
  response.redirect().toRoute('dashboard')
}
async destroy({ auth, response }: HttpContext) {
  await auth.use('web').logout()
  response.redirect().toRoute('session.create')
}
```

- Les redirections utilisent des **noms de routes** (`dashboard`, `session.create`). Pour pointer ailleurs (ex. `todos.index`), changer ces lignes et `GuestMiddleware.redirectTo` (voir `feature-guide.md`, étape 7).
- Les noms de routes viennent des contrôleurs (`controllers.Session` + `create` → `session.create`) ou de `.as('...')`.

## 4. Validateurs (VineJS)

```ts
const email = () => vine.string().email().maxLength(254)
const password = () => vine.string().minLength(8).maxLength(32)

export const signupValidator = vine.create({
  fullName: vine.string().nullable(),
  email: email().unique({ table: 'users', column: 'email' }),
  password: password(),
  passwordConfirmation: password().sameAs('password'),
})

export const loginValidator = vine.create({ email: email(), password: vine.string() })
```

- Forme `vine.create({...})` (le kit n'utilise plus `vine.compile(vine.object({...}))`).
- La confirmation du mot de passe se fait avec `passwordConfirmation: ...sameAs('password')` : le champ du formulaire s'appelle donc `passwordConfirmation` (le tutoriel utilise encore `.confirmed()`, qui attend `password_confirmation`).
- L'unicité de l'email est vérifiée par `unique({ table, column })` : cela interroge la base, donc la table doit exister.

## 5. Kernel et middlewares

`start/kernel.ts` : `router.use([...])` enregistre `bodyparser`, `session`, `shield` (CSRF), `initialize_auth` et `silent_auth` ; `server.use([...])` enregistre notamment `vite` et `inertia_middleware`. Middlewares nommés : `middleware.guest()` (réservé aux visiteurs) et `middleware.auth()` (réservé aux connectés, redirige vers `/login`).

Routes du kit :

```ts
router.on('/').renderInertia('home', {}).as('home')

router.group(() => {
  router.get('signup', [controllers.NewAccount, 'create'])
  router.post('signup', [controllers.NewAccount, 'store'])
  router.get('login', [controllers.Session, 'create'])
  router.post('login', [controllers.Session, 'store'])
}).use(middleware.guest())

router.group(() => {
  router.on('/dashboard').renderInertia('dashboard', {}).as('dashboard')
  router.post('logout', [controllers.Session, 'destroy'])
}).use(middleware.auth())
```

Vérifier : `node ace list:routes`.

## 6. Remember me (optionnel)

Le guard `web` a `useRememberMeTokens: false` par défaut.

1. `node ace make:migration remember_me_tokens` :

```ts
this.schema.createTable('remember_me_tokens', (table) => {
  table.increments('id')
  table.integer('tokenable_id').notNullable().unsigned()
    .references('id').inTable('users').onDelete('CASCADE')
  table.string('hash').notNullable().unique()
  table.timestamp('created_at').notNullable()
  table.timestamp('updated_at').notNullable()
  table.timestamp('expires_at').notNullable()
})
```

   puis `node ace migration:run`.
2. Modèle `User` : `static rememberMeTokens = DbRememberMeTokensProvider.forModel(User)` (import depuis `@adonisjs/auth/session`).
3. `config/auth.ts` : `useRememberMeTokens: true`.
4. Validateur : ajouter `remember: vine.boolean().optional()` à `loginValidator`.
5. Contrôleur : `const { email, password, remember } = await request.validateUsing(loginValidator)` puis `auth.use('web').login(user, !!remember)`.
6. Vue : `<input type="checkbox" name="remember" />` dans le `<Form>` (ou `v-model` si formulaire manuel).

## 7. Vérification

```bash
npm run dev
# /signup : créer un compte      -> redirection vers la route configurée (dashboard par défaut)
# POST /logout                   -> redirection vers /login
# /login                         -> redirection vers la route configurée
```

- Les mots de passe sont hachés automatiquement (jamais en clair en base).
- La session est maintenue par cookie ; en test, `.env.test` force `SESSION_DRIVER=memory`.
- Un utilisateur déjà connecté qui visite `/login` ou `/signup` est redirigé par `guest()` vers `GuestMiddleware.redirectTo`.
