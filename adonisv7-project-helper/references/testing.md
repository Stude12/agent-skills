# Tests (Japa) : configuration, fonctionnels, navigateur, console, doubles

Tout ce qui suit a été exécuté sur le kit Vue v7 avec MariaDB (23 tests verts + `npm run typecheck`). Documentation officielle : voir `official-docs.md` (section Testing).

## 1. Ce que le kit fournit (et ne fournit pas)

- Fournis : `tests/bootstrap.ts` (plugins `assert`, `pluginAdonisJS`, `dbAssertions`), `bin/test.ts`, suites `unit`, `functional`, `browser` dans `adonisrc.ts` (globs `tests/<suite>/**/*.spec.{ts,js}`), `.env.test` avec `SESSION_DRIVER=memory`.
- **Non fournis** : le client HTTP (`@japa/api-client`), `playwright`, les plugins session/auth/shield/inertia, et aucun test d'exemple (pas de dossier `tests/unit`, `tests/functional`).
- Conséquence : `node ace test unit` sans fichier répond `NO TESTS EXECUTED`.

## 2. Installation

```bash
npm i -D @japa/api-client playwright      # playwright : requis dès que le plugin browser est importé
npx playwright install chromium            # navigateur pour la suite browser
```

**Piège** : `@japa/browser-client` importe `playwright` au chargement. Si le plugin est enregistré mais que `playwright` n'est pas installé, **toute** commande `node ace test` (même `unit`) plante ou se fige sur `Cannot find package 'playwright'`. Installer `playwright` ou retirer les 3 plugins browser.

## 3. `tests/bootstrap.ts` validé

```ts
import { assert } from '@japa/assert'
import { apiClient } from '@japa/api-client'
import app from '@adonisjs/core/services/app'
import type { Config } from '@japa/runner/types'
import { pluginAdonisJS } from '@japa/plugin-adonisjs'
import { dbAssertions } from '@adonisjs/lucid/plugins/db'
import testUtils from '@adonisjs/core/services/test_utils'
import { browserClient } from '@japa/browser-client'
import { authApiClient } from '@adonisjs/auth/plugins/api_client'
import { authBrowserClient } from '@adonisjs/auth/plugins/browser_client'
import { sessionApiClient } from '@adonisjs/session/plugins/api_client'
import { sessionBrowserClient } from '@adonisjs/session/plugins/browser_client'
import { shieldApiClient } from '@adonisjs/shield/plugins/api_client'
import { inertiaApiClient } from '@adonisjs/inertia/plugins/api_client'
import type { Registry } from '../.adonisjs/client/registry/schema.d.ts'

declare module '@japa/api-client/types' {
  interface RoutesRegistry extends Registry {}
}

export const plugins: Config['plugins'] = [
  assert(),
  pluginAdonisJS(app),
  dbAssertions(app),
  apiClient(),
  sessionApiClient(app),
  authApiClient(app),
  shieldApiClient(),          // fournit .withCsrfToken()
  inertiaApiClient(app),      // fournit .withInertia() et assertInertia*
  browserClient({ runInSuites: ['browser'] }),
  sessionBrowserClient(app),
  authBrowserClient(app),
]

export const runnerHooks: Required<Pick<Config, 'setup' | 'teardown'>> = {
  setup: [() => testUtils.db().migrate()],   // applique les migrations avant la suite
  teardown: [],
}

export const configureSuite: Config['configureSuite'] = (suite) => {
  if (['browser', 'functional', 'e2e'].includes(suite.name)) {
    return suite.setup(() => testUtils.httpServer().start())
  }
}
```

Chemins d'import vérifiés (champ `exports` des paquets) : `@adonisjs/auth/plugins/{api_client,browser_client}`, `@adonisjs/session/plugins/{api_client,browser_client}`, `@adonisjs/inertia/plugins/api_client`, `@adonisjs/shield/plugins/api_client`. Le typage des noms de routes de `client.visit()` vient de l'ajout `RoutesRegistry extends Registry`.

**`.withCsrfToken()` n'existe pas sans `shieldApiClient()`** : l'erreur est `TypeError: client.visit(...).withCsrfToken is not a function`. Il manque aussi dans la doc de démarrage du kit.

## 4. Base de test

- Utiliser une base **distincte** : `.env.test` doit surcharger `DB_DATABASE` (sinon les tests vident la base de développement).
- MariaDB en Docker : créer la base et donner les droits :

```bash
docker compose exec -T db mariadb -uroot -p<root> -e \
  "CREATE DATABASE IF NOT EXISTS \`<projet>-db-test\`; GRANT ALL ON \`<projet>-db-test\`.* TO 'adonis'@'%'; FLUSH PRIVILEGES;"
```

- `.env.test` : `SESSION_DRIVER=memory` et `DB_DATABASE=<projet>-db-test`.
- `runnerHooks.setup: [() => testUtils.db().migrate()]` migre la base de test au démarrage (le log indique `Executed N migrations` puis `Reverted N migrations` à la fin).

## 5. Isoler l'état entre les tests

Dans chaque groupe :

```ts
group.each.setup(() => testUtils.db().truncate())               // vide toutes les tables après chaque test
// ou, plus rapide :
group.each.setup(() => testUtils.db().wrapInGlobalTransaction()) // rollback automatique
```

Les deux ont été validés, y compris sur la suite `browser` (le test de connexion passe avec `wrapInGlobalTransaction`). `withGlobalTransaction()` est déprécié. Par prudence, préférer `truncate()` pour les tests navigateur : le serveur de test peut ouvrir d'autres connexions que celle qui porte la transaction, ce que je n'ai pas cherché à provoquer.

## 6. Factories

```bash
node ace make:factory User     # database/factories/user_factory.ts (vide : à remplir)
node ace make:factory Todo
```

```ts
import factory from '@adonisjs/lucid/factories'
import User from '#models/user'

export const UserFactory = factory
  .define(User, async ({ faker }) => ({
    fullName: faker.person.fullName(),
    email: faker.internet.email().toLowerCase(),
    password: 'secret1234',                  // haché par withAuthFinder
  }))
  .build()

export const TodoFactory = factory
  .define(Todo, async ({ faker }) => ({
    title: faker.lorem.sentence(4),
    status: 'pending' as const,
  }))
  .relation('user', () => UserFactory)
  .build()
```

Usage : `await UserFactory.create()`, `await TodoFactory.merge({ userId: me.id }).createMany(3)`. Le générateur produit un `define` qui renvoie `{}` : sans le remplir, la création échoue sur les colonnes `NOT NULL`.

## 7. Tests fonctionnels (HTTP)

```ts
import { test } from '@japa/runner'
import testUtils from '@adonisjs/core/services/test_utils'

test.group('Todos', (group) => {
  group.each.setup(() => testUtils.db().truncate())

  test('guests are redirected to login', async ({ client }) => {
    const response = await client.visit('todos.index').redirects(0)
    response.assertStatus(302)
    response.assertHeader('location', '/login')
  })

  test('index renders the Inertia page with my todos only', async ({ client }) => {
    const me = await UserFactory.create()
    const other = await UserFactory.create()
    await TodoFactory.merge({ userId: me.id, title: 'Mine' }).create()
    await TodoFactory.merge({ userId: other.id, title: 'Theirs' }).create()

    const response = await client.visit('todos.index').loginAs(me).withInertia()

    response.assertStatus(200)
    response.assertInertiaComponent('todos/index')
    response.assertInertiaPropsContains({ todos: [{ title: 'Mine' }] })
  })

  test('store creates a todo', async ({ client, assert }) => {
    const me = await UserFactory.create()
    const response = await client
      .visit('todos.store')
      .loginAs(me)
      .withCsrfToken()
      .redirects(0)
      .form({ title: 'Buy milk' })

    response.assertStatus(302)
    assert.equal((await Todo.findByOrFail('title', 'Buy milk')).userId, me.id)
  })
})
```

Points validés :

- `client.visit('<nom.de.route>', { id })` résout méthode et URL depuis le routeur (typé via le registre).
- `.loginAs(user)` utilise le guard par défaut ; `.withCsrfToken()` pour tout `POST/PATCH/DELETE`.
- **Une soumission de formulaire redirige (302)** au lieu de répondre 200/204 : utiliser `.redirects(0)` pour asserter le code. Sans cela le client suit la redirection.
- Ajouter `.header('accept', 'application/json')` pour obtenir une réponse JSON (voir comportements ci-dessous).
- `.form({...})` envoie `application/x-www-form-urlencoded`, `.json({...})` envoie du JSON.

### Comportements observés (à ne pas supposer)

| Cas | Résultat réel |
|---|---|
| POST sans jeton CSRF (requête HTML) | **302** (redirection avec erreur flashée), pas 403 |
| Titre vide (`'   '`) à la validation | **302** (retour au formulaire), aucun enregistrement |
| `bouncer.authorize` refusé, requête HTML de formulaire | **302** (retour arrière) |
| `bouncer.authorize` refusé, `accept: application/json` | **403** |
| Exception personnalisée `static status = 422`, JSON | **422** |
| Route protégée sans session | **302** vers `/login` |

Pour tester un refus d'accès, vérifier **aussi** l'état en base (le test `cannot update someone else's todo` recharge le modèle et contrôle que `status` est resté `pending`), pas seulement le code HTTP.

## 8. Tests navigateur (Playwright)

```ts
import { test } from '@japa/runner'
import testUtils from '@adonisjs/core/services/test_utils'
import { UserFactory } from '#database/factories/user_factory'

test.group('Auth (browser)', (group) => {
  group.each.setup(() => testUtils.db().truncate())

  test('a user can log in through the form', async ({ visit, assert }) => {
    const user = await UserFactory.create()
    const page = await visit('/login')

    await page.locator('input[name="email"]').fill(user.email)
    await page.locator('input[name="password"]').fill('secret1234')
    await page.locator('button[type="submit"]').click()

    // Inertia navigue en SPA : attendre un élément de la page cible
    await page.getByText('Your dashboard is ready').waitFor()
    assert.match(page.url(), /\/dashboard$/)
  })
})
```

- `visit('/chemin')` ouvre une page sur le serveur de test (port aléatoire).
- **Ne pas utiliser `page.waitForURL('**/dashboard')`** : la navigation Inertia ne déclenche pas d'événement `load`, l'attente expire (30 s, `TimeoutError`). Attendre plutôt un texte ou un élément de la page cible.
- Les sélecteurs (`input[name="email"]`, `button[type="submit"]`) viennent des pages du kit.
- Lancement : `node ace test browser`. Pour sauter le navigateur : `node ace test unit functional`.
- Un test navigateur bloqué peut laisser des processus (`chrom`, `bin/test.ts`) : toujours lancer avec `timeout 120 node ace test browser`.

## 9. Isolation d'un limiteur de débit partagé

Un limiteur en mémoire (`LIMITER_STORE=memory`) est **partagé par toute la suite** : un test qui épuise la limite de l'IP `127.0.0.1` fait échouer les tests suivants (le test navigateur de connexion a expiré pour cette raison). Vider le store avant **et** après :

```ts
import limiter from '@adonisjs/limiter/services/main'

group.each.setup(async () => {
  await limiter.clear()
  return async () => { await limiter.clear() }
})
```

Voir `security-middleware.md`.

## 10. Tests unitaires et doubles

```ts
import { test } from '@japa/runner'
import hash from '@adonisjs/core/services/hash'
import emitter from '@adonisjs/core/services/emitter'

test('hash.fake() skips real hashing', async ({ assert }) => {
  using _hash = hash.fake()        // restauré automatiquement en fin de test
  const hashed = await hash.make('secret')
  assert.isTrue(await hash.verify(hashed, 'secret'))
})

test('emitter.fake() captures events', async () => {
  using fakeEmitter = emitter.fake()
  await emitter.emit(events.UserRegistered, { user })
  fakeEmitter.assertEmitted(events.UserRegistered)
})
```

- Les fakes intégrés : `emitter`, `hash`, `mail`, `drive` (doc officielle « Test doubles »). Le mot-clé `using` restaure le service à la sortie du test.
- Recommandation officielle : **ne pas mocker la base** ; utiliser la vraie base de test. Les fakes servent pour les services externes (mail, paiement, stockage).
- Le plugin `@japa/mock` n'est **pas** installé par le kit ; `spy.on`, `stub`, `mock.route` ne sont pas disponibles sans lui.

Modèle (unitaire, sans HTTP) :

```ts
test('initials fall back to the email', ({ assert }) => {
  const user = new User()
  user.fullName = null
  user.email = 'zoe@example.com'
  assert.equal(user.initials, 'ZE')   // le getter découpe l'email sur '@' : 'zoe' + 'example.com'
})
```

## 11. Tests de commandes Ace

```bash
node ace make:test commands/todos_count --suite=unit
```

```ts
import { test } from '@japa/runner'
import ace from '@adonisjs/core/services/ace'
import testUtils from '@adonisjs/core/services/test_utils'
import TodosCount from '../../../commands/todos_count.js'

test.group('Command todos:count', (group) => {
  group.each.setup(() => testUtils.db().truncate())
  group.each.setup(() => {
    ace.ui.switchMode('raw')                       // capture les sorties en mémoire
    return () => ace.ui.switchMode('normal')
  })

  test('prints the number of todos', async ({ assert }) => {
    const user = await UserFactory.create()
    const command = await ace.create(TodosCount, [user.email])
    await command.exec()

    command.assertSucceeded()
    assert.deepEqual(
      command.logger.getLogs().map((l) => l.message),
      [`[ blue(info) ] ${user.email}: 1 todo(s)`]    // préfixe de niveau en mode raw
    )
  })

  test('fails for an unknown user', async () => {
    const command = await ace.create(TodosCount, ['ghost@example.com'])
    await command.exec()
    command.assertFailed()
  })
})
```

- Les commandes du kit vivent dans `commands/` (racine), pas `app/commands/` : l'import dans un test est relatif (`../../../commands/x.js`) car il n'existe pas d'alias `#commands/*` par défaut.
- Mode `raw` : chaque message est préfixé par le niveau, par exemple `[ blue(info) ] ...`. Une assertion sur le texte seul échoue.
- La commande doit déclarer `static options = { startApp: true }` pour utiliser la base ou le conteneur.

## 12. Commandes utiles

```bash
node ace test                          # toutes les suites
node ace test unit functional          # sans navigateur
node ace test --files=todos            # fichiers dont le nom contient 'todos'
node ace test --tests="store"          # tests par titre
timeout 120 node ace test browser      # navigateur, avec garde-fou
npm run typecheck                      # tsc + vue-tsc : les tests sont typés aussi
```

## 13. Checklist de dépannage

| Symptôme | Cause |
|---|---|
| `NO TESTS EXECUTED` | aucun fichier dans la suite ciblée |
| `Cannot find package 'playwright'`, commande figée | plugin browser enregistré sans `playwright` installé |
| `withCsrfToken is not a function` | `shieldApiClient()` absent des plugins |
| `loginAs is not a function` | `authApiClient(app)` (et `sessionApiClient(app)`) absents |
| `withInertia is not a function` | `inertiaApiClient(app)` absent |
| `Table '...-test.users' doesn't exist` (commande lancée hors test) | seule la base de test est configurée dans `.env.test`, elle n'est migrée que par `node ace test` ; une commande lancée à la main utilise `.env` |
| `waitForURL` expire | navigation Inertia sans événement `load` : attendre un élément |
| un test échoue seulement dans la suite complète | état partagé (limiteur mémoire, sessions) : nettoyer en `each.setup` |
| `TypeError` sur `status` attendu 403 | la requête HTML redirige (302) : ajouter `accept: application/json` si besoin d'un 403 |
