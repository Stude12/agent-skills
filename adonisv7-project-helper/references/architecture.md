# Architecture d'une application AdonisJS v7 : quand structurer, comment

Principe directeur : **partir du kit, n'ajouter une couche que lorsqu'un besoin précis l'exige.** Le kit v7 donne déjà `controllers`, `models`, `validators`, `middleware`, `transformers`, `exceptions`. Chaque couche supplémentaire (service, repository, événement, contrat) a un coût de lecture ; ce document dit quand il se justifie et comment l'écrire correctement en v7.

Tout ce qui suit a été exécuté sur le kit Vue v7 avec MariaDB (28 tests verts, `tsc` propre). Documentation officielle : `official-docs.md` (Concepts > Dependency injection, Digging deeper > Emitter, Database > Lucid).

Source d'inspiration : le skill communautaire `omakei/adonisjs-architecture-skill`, écrit pour **AdonisJS v6** (API). Ses idées sont reprises quand elles se vérifient en v7, corrigées sinon (section 11).

## 1. Où mettre quoi

| Besoin | Où | Créé par |
|---|---|---|
| Valider une entrée HTTP | `app/validators/` (`vine.create`) | kit |
| Orchestration HTTP (valider, appeler, répondre) | `app/controllers/` | `make:controller` |
| Forme des données envoyées au frontend | `app/transformers/` | `make:transformer` |
| Règle métier réutilisée par HTTP **et** commande/listener | `app/services/` | `make:service` |
| Requêtes Lucid non triviales et réutilisées | `app/repositories/` (à créer) | à la main |
| Erreur métier | `app/exceptions/` | `make:exception` |
| Autorisation d'une ressource | `app/policies/` | `make:policy` (Bouncer) |
| Effet de bord découplé (notification, audit) | `app/events/` + `app/listeners/` | `make:event`, `make:listener` |
| Dépendance interchangeable (paiement, e-mail) | `app/contracts/` (classe abstraite) + provider | à la main |
| Calcul pur, sans base ni HTTP | `app/domain/` ou fonction dans le service | à la main |

Les alias `#xxx/*` du `package.json` existent pour `controllers`, `services`, `events`, `listeners`, `exceptions`, `policies`, `abilities`, `transformers`, `validators`, `providers`, `database`, `start`, `mails`, `middleware`, `models`. Pour un nouveau dossier, **ajouter l'alias** soi-même :

```json
"imports": {
  "#contracts/*": "./app/contracts/*.js",
  "#repositories/*": "./app/repositories/*.js"
}
```

Sans cela : `TypeError: Package import specifier "#contracts/notifier" is not defined in package ...` à l'exécution et `TS2307` au typecheck (rencontré pendant la validation).

## 2. Échelle de décision : quand ajouter une couche

Ne pas appliquer toutes les couches à toute l'application. Monter d'un cran seulement quand le symptôme apparaît.

| Niveau | Quand | Structure |
|---|---|---|
| **0 : kit seul** | CRUD simple, logique de 3 lignes | Controller → Model, validateur, transformer |
| **1 : service** | la même règle sert deux points d'entrée (contrôleur **et** commande Ace, listener, job), ou doit être testée sans HTTP | Controller → Service → Model |
| **2 : repository** | la même requête complexe est copiée dans plusieurs services, ou une requête exige une transaction transmise | Service → Repository → Model |
| **3 : événements** | un effet de bord ne fait pas partie du contrat de l'opération (notification, audit) | Service → `dispatch` → Listener |
| **4 : contrat** | une dépendance externe doit être remplaçable ou fausse en test | Service → classe abstraite ← implémentation liée par provider |

Exemple du tutoriel Todos : niveau 0 suffit (le contrôleur filtre par `userId`). Le passage au niveau 1 est justifié par la limite de 3 todos, appelée par le contrôleur **et** par la commande `todos:count` (voir `ace-services.md`).

Règles de bon sens :

- **Ne pas créer un repository par modèle « par principe »** : `Todo.query().where('userId', id)` dans un service est lisible. Un repository est utile quand la requête porte une règle (`findOverdue`) ou doit recevoir une transaction (section 7).
- **Un modèle Lucid reste mince** : relations, getters, mixins. Les colonnes viennent de `database/schema.ts` (généré). La logique d'orchestration va dans un service.
- **Un contrôleur** : valider, appeler **une** méthode de service, répondre. Pas de requête SQL inline sauf CRUD trivial.

## 3. Contrôleur mince avec service injecté

Vérifié : `@inject()` sur la méthode, `HttpContext` **toujours en premier paramètre**, dépendances ensuite.

```ts
import { inject } from '@adonisjs/core'
import type { HttpContext } from '@adonisjs/core/http'
import TodoWorkflowService from '#services/todo_workflow_service'
import { createTodoValidator } from '#validators/todos/create'

export default class TodosController {
  @inject()
  async store({ auth, request, response }: HttpContext, workflow: TodoWorkflowService) {
    const payload = await request.validateUsing(createTodoValidator)
    await workflow.createMany(auth.use('web').user!.id, [payload.title])
    return response.redirect().back()
  }
}
```

Le service n'a jamais accès à `HttpContext` : lui passer des **valeurs** (`userId`, données validées), pas `ctx`. C'est ce qui permet de l'appeler depuis une commande ou un test.

### Types d'entrée sans DTO manuel

Pas besoin de dossier `dtos/` pour la plupart des cas : VineJS infère le type du validateur (vérifié).

```ts
// app/validators/todos/create.ts
import vine from '@vinejs/vine'
import type { Infer } from '@vinejs/vine/types'

export const createTodoValidator = vine.create({
  title: vine.string().trim().minLength(1).maxLength(255),
  description: vine.string().trim().nullable().optional(),
})
export type CreateTodoInput = Infer<typeof createTodoValidator>
```

```ts
// le service reçoit un type précis ; une propriété inconnue ou un mauvais type échoue au typecheck
create(input: CreateTodoInput, userId: number) { /* ... */ }
```

Vérifié : `describe({ title: 'x', nope: true }, 1)` → `TS2353: Object literal may only specify known properties`. Un objet d'entrée explicite (`interface`) n'est utile que si l'on doit enrichir avec un contexte absent de la requête (`userId`, résultat d'une autre lecture). Pour la sortie vers le frontend, utiliser un **transformer** (voir `tuyau-transformers.md`), pas un DTO écrit à la main.

## 4. Services

```bash
node ace make:service TodoWorkflow    # app/services/todo_workflow_service.ts  (export nommé, classe vide)
```

Choisir un style d'export et le garder (`export default class` ou `export class`). Les services ne portent **pas d'état par requête** : tout passe par les paramètres.

Règles vérifiées :

- **Injection par constructeur** : `@inject()` sur la classe, les types du constructeur servent de clés. Fonctionne pour une classe concrète (`TodoRepository`) **et** pour une classe abstraite liée par provider (`Notifier`, section 6).
- Un service résolu par `app.container.make(Service)` est **recréé à chaque appel** (vérifié : `service_same=false service_new=2` pour deux `make`) ; ses dépendances liées en singleton, elles, sont partagées (`same=true`).
- **Jamais de modèle Lucid ni de `Map` non borné en propriété d'instance** d'un service durable : la référence ne serait jamais libérée.
- Une règle de refus lève une **exception métier** (voir `errors-logging.md`) ; ne pas renvoyer `null` pour signaler une erreur.

## 5. Événements (v7)

À la différence de la v6 et du skill d'omakei, la v7 propose des événements **de classe** avec `dispatch` statique.

```bash
node ace make:event TodoCreated
node ace make:listener NotifyOwner
node ace make:preload events --register    # crée start/events.ts ET l'enregistre dans adonisrc.ts
```

**Le kit n'a pas de `start/events.ts`** : sans `make:preload events`, aucun listener n'est enregistré. Sans `--register`, la commande pose une question interactive et se fige dans un script.

```ts
// app/events/todo_created.ts
import { BaseEvent } from '@adonisjs/core/events'
import type Todo from '#models/todo'

export default class TodoCreated extends BaseEvent {
  constructor(public todo: Todo) {
    super()
  }
}
```

```ts
// app/listeners/notify_owner.ts
import { inject } from '@adonisjs/core'
import type TodoCreated from '#events/todo_created'

@inject()
export default class NotifyOwner {
  async handle(event: TodoCreated) { /* travail rapide, ou dispatch d'un job */ }
}
```

```ts
// start/events.ts : tout est enregistré ICI, une seule fois, au démarrage
import emitter from '@adonisjs/core/services/emitter'
import logger from '@adonisjs/core/services/logger'
import { events } from '#generated/events'
import { listeners } from '#generated/listeners'

emitter.on(events.TodoCreated, [listeners.NotifyOwner])    // TABLEAU de listeners

emitter.onError((eventName, error) => {
  logger.error({ event: String(eventName), err: error }, 'listener failed')
})
```

```ts
// émission
await TodoCreated.dispatch(todo)        // ou events.TodoCreated.dispatch(todo)
```

Faits vérifiés par test :

| Point | Constat |
|---|---|
| Forme d'enregistrement | `emitter.on(evt, listeners.X)` (sans tableau) **n'appelle jamais** le listener, sans erreur ; il faut `[listeners.X]` (conforme à la doc) |
| Barrels `#generated/events` et `#generated/listeners` | **vides** jusqu'au démarrage de `node ace serve` (ou du build) ; `list:routes` ne les régénère pas |
| `dispatch(...)` | retourne `Promise<void>` ; les arguments sont ceux du constructeur |
| Listeners d'un même événement avec `emit` | **concurrents** : `a:start,b:start,b:end,a:end` |
| `emitSerial` | **séquentiels, dans l'ordre d'enregistrement** : `a:start,a:end,b:start,b:end` |
| Listener qui lève une exception | passe par `onError` ; `emit` **ne rejette pas** (`rejected=false`) |
| `await` sur `dispatch`/`emit` | attend la fin de **tous** les listeners ; sans `await`, ils tournent en arrière-plan |

Ce dernier point contredit le skill d'omakei (« listeners exécutés séquentiellement ») : la doc officielle v7 dit « concurrently » et le test le confirme.

**Fuite de listeners** (vérifié) : 25 appels à `emitter.on(...)` dans une boucle donnent `listenerCount = 25`; la fonction retournée par `on` les retire (`0`). `emitter.once` se retire seul après l'émission. Aucun avertissement `MaxListenersExceededWarning` n'a été observé (l'émetteur est `emittery` 1.x, pas `EventEmitter` de Node). La règle reste valable : **ne jamais appeler `emitter.on` dans un handler HTTP ou un constructeur** ; conserver la fonction de désinscription si l'on enregistre dynamiquement.

**Quand utiliser un événement plutôt qu'un appel direct** : quand l'effet est un « bonus » (notification, audit) et ne doit pas faire échouer l'opération. Si l'effet **fait partie du contrat** (écrire la ligne de journal obligatoire), l'exécuter dans la transaction du service.

**Travail lent dans un listener** (e-mail, webhook, PDF) : y dispatcher un **job** (`@adonisjs/queue`, voir `optional-packages.md`) ; `await dispatch` bloque la réponse tant que tous les listeners n'ont pas fini.

### Tester les événements (vérifié)

```ts
import emitter from '@adonisjs/core/services/emitter'
import { events } from '#generated/events'

test('emits TodoCreated', async () => {
  using fakeEmitter = emitter.fake()          // restauré automatiquement à la sortie du bloc
  await workflow.createMany(user.id, ['a'])
  fakeEmitter.assertEmitted(events.TodoCreated)
  fakeEmitter.assertEmittedCount(events.TodoCreated, 1)
  fakeEmitter.assertEmitted(events.TodoCreated, ({ data }) => data.todo.title === 'a')
  fakeEmitter.assertNotEmitted(events.TodoDeleted)
})
```

- **Avec `fake()`, les listeners ne s'exécutent pas** (vérifié : `NotifyOwner.calls` reste vide), puis le comportement normal revient à la sortie du bloc.
- `using` n'exige pas de `cleanup` : c'est `Symbol.dispose` de l'objet retourné. `emitter.fake()` puis `emitter.restore()` manuel fonctionne aussi.

## 6. Contrats et injection de dépendances

Pour une dépendance remplaçable (e-mail, paiement, stockage) : **classe abstraite** comme clé de liaison, pas une `interface` (effacée à la compilation, donc inutilisable par le conteneur).

```ts
// app/contracts/notifier.ts
export abstract class Notifier {
  abstract send(to: string, message: string): Promise<string>
}
```

```ts
// providers/billing_provider.ts  (node ace make:provider billing --register)
import type { ApplicationService } from '@adonisjs/core/types'
import { Notifier } from '#contracts/notifier'

export default class BillingProvider {
  constructor(protected app: ApplicationService) {}

  register() {
    this.app.container.singleton(Notifier, async () => {
      const { default: EmailNotifier } = await import('#services/email_notifier')
      return new EmailNotifier()
    })
  }
}
```

```ts
// app/services/reminder_service.ts : reçoit le contrat, ignore l'implémentation
@inject()
export default class ReminderService {
  constructor(protected notifier: Notifier) {}
}
```

Vérifié :

- `app.container.make(Notifier)` renvoie l'implémentation liée (`instanceOf EmailNotifier`).
- `singleton` : deux `make` renvoient la **même** instance (`same=true`, une seule construction).
- L'`await import(...)` est valide **dans la fabrique** (fonction `async`), pas dans `register()` lui-même : voir section 11.
- `make:provider <nom> --register` enregistre dans `adonisrc.ts` ; sans `--register`, question interactive.
- Les providers sont ajoutés à `providers: [...]` ; ils ne servent qu'à **lier** : pas de logique métier.

### Remplacer en test : `swap` / `restore`

```ts
app.container.swap(Notifier, () => ({ send: async (to: string) => `fake:${to}` }) as Notifier)
const result = await (await app.container.make(ReminderService)).remind('ada@x.io')   // 'fake:ada@x.io'
app.container.restore(Notifier)                                                       // retour à l'implémentation réelle
```

Vérifié : pendant le swap, `ReminderService` reçoit le faux ; après `restore`, la vraie implémentation. Le `swap` a la **priorité la plus haute** du conteneur (doc : « Swaps » avant liaisons contextuelles et ordinaires). Toujours appeler `restore` (dans `try/finally` ou un hook de nettoyage) pour ne pas contaminer les tests suivants.

### Faut-il tout enregistrer en singleton ?

Le skill d'omakei l'érige en règle (« les services DOIVENT être des singletons »). **Mesuré sur ce projet** :

```
résolution @inject (service + repository) : 2,16 µs par make()
résolution singleton                      : 0,40 µs par make()
```

L'écart (~1,8 µs par requête) est **négligeable** devant une requête SQL (centaines de µs à ms). Enregistrer en singleton n'a de sens que pour : un **client coûteux ou à état partagé** (connexion, client HTTP, cache), ou une implémentation liée à un contrat (section ci-dessus). Pour le reste, laisser le conteneur résoudre avec `@inject()` ; un singleton n'apporte ici que le risque de conserver de l'état par erreur.

## 7. Transactions : au niveau du service

La transaction délimite **une opération métier** : elle se pose dans le service, et le client transactionnel se transmet aux repositories.

```ts
// app/repositories/todo_repository.ts
import type { TransactionClientContract } from '@adonisjs/lucid/types/database'
import Todo from '#models/todo'

export default class TodoRepository {
  async create(data: { userId: number; title: string }, trx?: TransactionClientContract) {
    return Todo.create(data, trx ? { client: trx } : undefined)
  }
}
```

```ts
// app/services/todo_workflow_service.ts
import { inject } from '@adonisjs/core'
import db from '@adonisjs/lucid/services/db'

@inject()
export default class TodoWorkflowService {
  constructor(protected todos: TodoRepository) {}

  async createMany(userId: number, titles: string[]) {
    const created = await db.transaction(async (trx) => {
      const rows = []
      for (const title of titles) rows.push(await this.todos.create({ userId, title }, trx))
      return rows
    })
    // effets de bord APRÈS le commit
    for (const todo of created) await TodoCreated.dispatch(todo)
    return created
  }
}
```

Vérifié par test :

- Succès : 3 lignes en base, 3 événements reçus.
- **Échec au 3e élément** : `0` ligne en base (rollback complet) et **aucun événement** émis, puisque le `dispatch` est après la transaction.
- `db.transaction(async (trx) => ...)` est **géré** : commit si la fonction réussit, rollback si elle lève (doc Lucid).
- Transmission : `Todo.create(data, { client: trx })` pour la création, `query.useTransaction(trx)` pour une requête, `model.useTransaction(trx)` pour un modèle existant.

### Pas d'appel externe dans une transaction (mesuré)

Une transaction **retient une connexion du pool** pendant toute sa durée. Sur ce projet (pool `min=2`, `max=10`) :

```
avant           : used=0  free=1  pending=0
dans la trx     : used=1  free=0  pending=0
après           : used=0  free=1  pending=0
10 trx ouvertes : used=10 free=0  pending=1   → la requête suivante ATTEND
```

Donc : aucun appel HTTP, e-mail ou envoi de fichier dans le callback de `db.transaction`. Appeler l'extérieur **avant** (puis ouvrir une transaction courte) ou **après** (événement). Introspection du pool, pour un diagnostic ponctuel :

```ts
const pool = (db.connection() as any).getWriteClient().client.pool
pool.numUsed(); pool.numFree(); pool.numPendingAcquires(); pool.min; pool.max
```

> `db.connection().pool` (utilisé par le skill d'omakei) vaut `undefined` en v7 : le pool se trouve sous `getWriteClient().client.pool`. API interne non documentée, à réserver au diagnostic.

## 8. Autorisation et multi-tenant

- **Autorisation** : dans le contrôleur, **avant** d'appeler le service ; le service suppose l'appelant autorisé et ne reçoit jamais `bouncer`. Voir `authorization-bouncer.md`.
- **Isolation des données** : passer le `userId` (ou `tenantId`) **explicitement** à chaque méthode de service/repository ; ne jamais le stocker sur une instance de service. Un oubli du filtre est alors visible à la lecture et au typecheck. Tester l'isolation avec deux utilisateurs (voir `testing.md`).
- Un middleware ne doit pas écrire sur `ctx` avec un cast `as any` : étendre `HttpContext` proprement ou renvoyer la valeur via l'`auth.user`.

## 9. Structure recommandée (niveau 1 à 4)

```
app/
├── controllers/        mince : valider, appeler un service, répondre
├── services/           règles métier, transactions
├── repositories/       requêtes non triviales (alias #repositories/* à ajouter)
├── contracts/          classes abstraites des dépendances interchangeables (alias #contracts/*)
├── events/  listeners/ effets de bord découplés (start/events.ts les relie)
├── policies/           Bouncer
├── exceptions/         erreurs métier
├── transformers/       forme des données vers le frontend
├── validators/         VineJS (+ types Infer)
└── models/             relations, getters ; colonnes dans database/schema.ts (généré)
providers/              liaisons du conteneur seulement
start/events.ts         enregistrement des listeners (make:preload events --register)
```

Noms de fichiers en **snake_case**, une classe par fichier, imports par alias `#xxx/*` jamais en `../../`.

## 10. Checklist avant de fusionner

- [ ] Chaque couche ajoutée répond à un besoin du tableau de la section 2.
- [ ] Les alias `#xxx/*` des nouveaux dossiers existent (`npm run typecheck` le détecte).
- [ ] Aucun `HttpContext` ni `bouncer` passé à un service.
- [ ] Aucun `emitter.on` hors de `start/events.ts` ; `emitter.on(evt, [listeners.X])` avec un **tableau**.
- [ ] `start/events.ts` est enregistré dans `adonisrc.ts` (`make:preload events --register`).
- [ ] Transactions : uniquement des opérations de base ; `dispatch` après le commit.
- [ ] Test d'un rollback complet **et** de l'absence d'événement en cas d'échec.
- [ ] Tests de contrat : `swap` suivi de `restore`.
- [ ] Pas de modèle Lucid ni de cache non borné en propriété d'un service durable.
- [ ] `node ace serve` lancé au moins une fois (génère `#generated/events`, `#generated/listeners`).

## 11. Corrections par rapport à `omakei/adonisjs-architecture-skill` (v6)

Chaque ligne a été vérifiée par la documentation v7, un test ou le typecheck.

| Affirmation du skill v6 | Réalité en v7 |
|---|---|
| `emitter.emit(new PaymentReceived(payment))` | événements de classe : `PaymentReceived.dispatch(payment)` ; classe étendant `BaseEvent` |
| `emitter.on(Event, [Listener])` avec `() => import('#events/...')` | `emitter.on(events.X, [listeners.Y])` avec les barrels `#generated/events` et `#generated/listeners` |
| « `emitter.emit()` exécute les listeners **séquentiellement** » | **concurrents** avec `emit`/`dispatch` ; `emitSerial` pour l'ordre (doc + test) |
| un listener qui échoue fait échouer la requête | l'erreur va à `emitter.onError` ; `emit` ne rejette pas (test : `rejected=false`) |
| tout service doit être un singleton (GC sous charge) | écart mesuré 1,8 µs par résolution : négligeable ; singleton seulement pour client coûteux ou contrat |
| provider avec `await import(...)` directement dans `register()` | `TS1308: 'await' expressions are only allowed within async functions` ; l'`await` va dans la **fabrique** `async` du `singleton`/`bind` |
| middleware tenant : `(ctx as any).tenantId = ...` avec `ctx` jamais déclaré | `TS2304: Cannot find name 'ctx'` et `TS2339: Property 'tenantId' does not exist on type 'HttpContext'` |
| `db.connection().pool` pour surveiller le pool | `undefined` ; pool sous `getWriteClient().client.pool` (API interne) |
| `vine.compile(vine.object({...}))` | `vine.create({...})` ; types via `Infer<typeof validator>` |
| `withGlobalTransaction()` | déprécié : `wrapInGlobalTransaction()` (voir `testing.md`) |
| `cleanup(() => emitter.restore())` obligatoire | `using fakeEmitter = emitter.fake()` restaure automatiquement |
| `app/providers/` pour les providers | dossier racine `providers/` (alias `#providers/*`) |
| `app/dtos/` obligatoire par domaine | inutile pour la plupart des cas : `Infer` du validateur pour l'entrée, **transformer** pour la sortie |
| Repository obligatoire pour toute requête | utile seulement pour les requêtes réutilisées ou transactionnelles |
| « Aucun barrel `index.ts` » | juste, et la v7 **génère** ses propres barrels (`#generated/controllers`, `events`, `listeners`, `policies`) à ne pas éditer |

Idées conservées telles quelles (confirmées) : contrôleurs minces, services orchestrateurs, exceptions métier typées, contrats par classes abstraites, `swap`/`restore` pour les tests, transactions au niveau du service, aucun appel externe dans une transaction, effets de bord après le commit, snake_case et alias `#`.

## 12. Vérification

```bash
npm run typecheck                        # alias manquants, injections, types Infer
node ace serve --hmr                     # génère #generated/events et #generated/listeners
node ace list:routes
node ace test unit functional
```
