# Commandes Ace, services et injection de dépendances

Validé sur le kit Vue v7 : commande avec argument et flag, service injecté dans un contrôleur, tests (23 tests verts). Documentation officielle : `official-docs.md` (Command line, Core concepts).

## 1. Commandes Ace disponibles dans le kit

`node ace list` (extrait réel) : `add`, `build`, `configure`, `eject`, `list`, `list:routes`, `repl`, `serve`, `test`, `generate:key`, `db:seed`, `db:truncate`, `db:wipe`, `migration:run|rollback|status|fresh|refresh|reset`, et les générateurs `make:command|controller|event|exception|factory|listener|middleware|migration|model|page|policy|preload|provider|seeder|service|session-table|test|transformer|validator|view`.

Options notables (aide réelle) : `make:model --migration --controller --transformer`, `make:factory <Model>`, `make:test <nom> --suite=unit`, `node ace add <paquet>` (installe et configure), `node ace configure <paquet>`.

> `make:policy` n'existe qu'après `node ace add @adonisjs/bouncer` ; `make:mail`, `make:job`... n'existent qu'avec leurs paquets.

## 2. Créer une commande

```bash
node ace make:command todos:count      # crée commands/todos_count.ts (racine du projet)
```

Les commandes du projet sont découvertes automatiquement dans `commands/` (commentaire de `adonisrc.ts`). Elles ne sont **pas** sous `app/`.

```ts
import { BaseCommand, args, flags } from '@adonisjs/core/ace'
import type { CommandOptions } from '@adonisjs/core/types/ace'

export default class TodosCount extends BaseCommand {
  static commandName = 'todos:count'
  static description = 'Count the todos of a user'
  static options: CommandOptions = { startApp: true }   // démarre l'application : base, conteneur, modèles

  @args.string({ description: 'Email of the user' })
  declare email: string

  @flags.boolean({ description: 'Print as a table', alias: 't' })
  declare table: boolean

  async run() {
    const { default: User } = await import('#models/user')
    const { default: TodoStatService } = await import('#services/todo_stat_service')
    const user = await User.findByOrFail('email', this.email)
    const service = await this.app.container.make(TodoStatService)
    const total = await service.countFor(user.id)

    if (this.table) {
      const table = this.ui.table()
      table.head(['User', 'Todos'])
      table.row([user.email, String(total)])
      table.render()
      return
    }
    this.logger.info(`${user.email}: ${total} todo(s)`)
  }
}
```

Aide générée automatiquement (`node ace todos:count --help`) :

```
Usage:  node ace todos:count [options] [--] <email>
Arguments:  email        Email of the user
Options:    -t, --table  Print as a table
```

Règles vérifiées :

- **`startApp: true` est obligatoire** pour toucher à la base ou au conteneur ; sans lui la commande tourne sans application démarrée.
- **Imports** : un `import User from '#models/user'` statique en tête de fichier **fonctionne aussi** avec `startApp: true` (vérifié : la commande s'exécute et atteint la base). Les imports dynamiques de l'exemple sont facultatifs ; ils servent à ne charger les modèles que lorsque la commande tourne réellement (par exemple pour que `node ace list` n'importe pas l'ORM).
- Argument : `@args.string()` (obligatoire par défaut, `required: false` pour optionnel) ; plusieurs valeurs : `@args.spread()`, **dernier argument obligatoirement**. Flags : `@flags.boolean()`, `@flags.string()`, `@flags.number()`, `@flags.array()`.
- Sortie : `this.logger.info|success|warning|error(...)`, `this.ui.table()`, `this.prompt` (questions interactives).
- Échec : lever une exception (`findByOrFail` sans résultat → `E_ROW_NOT_FOUND`, la commande se termine en échec ; vérifié par `command.assertFailed()` dans le test) ou `this.exitCode = 1`.
- Une commande lancée sans variables d'environnement valides échoue au démarrage (validation `start/env.ts`).

Lancer : `node ace todos:count ada@example.com --table`. Tests : voir `testing.md`, section 11.

## 3. Services (logique métier)

```bash
node ace make:service TodoStat      # app/services/todo_stat_service.ts
```

**Le nom généré est `todo_stat_service.ts`** (suffixe `_service` ajouté) et le fichier contient `export class TodoStatService { // Your code here }`, soit un **export nommé**, pas un `export default`. Le projet déclare l'alias `#services/*` dans `package.json` (aliases présents : `#exceptions`, `#mails`, `#services`, `#generated`, `#transformers`, `#providers`, `#policies`, `#abilities`, `#database`, `#start`, ... ; **il n'existe pas de `#commands/*`**, d'où l'import relatif dans les tests de commande).

Version utilisée dans les contrôleurs (export par défaut, simple à importer) :

```ts
// app/services/todo_stat_service.ts
import { inject } from '@adonisjs/core'
import Todo from '#models/todo'
import TodoLimitReachedException from '#exceptions/todo_limit_reached_exception'

@inject()
export default class TodoStatService {
  static MAX_TODOS = 3

  async countFor(userId: number) {
    const row = await Todo.query().where('userId', userId).count('* as total').first()
    return Number(row?.$extras.total ?? 0)
  }

  async assertCanCreate(userId: number) {
    if ((await this.countFor(userId)) >= TodoStatService.MAX_TODOS) {
      throw new TodoLimitReachedException()
    }
  }
}
```

Choisir **un seul** style d'export dans le projet et s'y tenir (le générateur utilise l'export nommé : `import { TodoStatService } from '#services/todo_stat_service'`).

Quand créer un service : logique réutilisée par un contrôleur **et** une commande (ici, `countFor`), règle métier à tester isolément, appel à un service externe. Une action de contrôleur de 5 lignes n'en a pas besoin.

## 4. Injection de dépendances (`@inject`)

Injection par méthode de contrôleur (forme validée) : `HttpContext` est **toujours le premier paramètre**, les dépendances suivent.

```ts
import { inject } from '@adonisjs/core'
import type { HttpContext } from '@adonisjs/core/http'
import TodoStatService from '#services/todo_stat_service'

export default class TodosController {
  @inject()
  async store({ auth, request, response }: HttpContext, todoStats: TodoStatService) {
    const user = auth.use('web').user!
    await todoStats.assertCanCreate(user.id)
    // ...
  }
}
```

- Le décorateur `@inject()` est **obligatoire** (sur la méthode pour l'injection de méthode, sur la classe pour l'injection de constructeur) : sans lui le conteneur ne résout rien et le paramètre vaut `undefined`.
- Injection de constructeur : `@inject()` sur la classe, `constructor(protected todoStats: TodoStatService) {}`.
- Dans une commande ou un script : `await this.app.container.make(MaClasse)` (ou `app.container.make` depuis `@adonisjs/core/services/app`).
- Pour **tester** avec un faux service, remplacer la liaison dans le conteneur (voir la doc « Test doubles », section container swaps) plutôt que de modifier la classe.

Les services hors requête (commande, tâche) ne peuvent pas recevoir `HttpContext` : leur passer l'identifiant utilisateur (`countFor(userId)`), pas `ctx`.

## 5. Providers et préchargements

- `node ace make:provider` : classe avec `register()`, `boot()`, `start()`, `ready()`, `shutdown()`. Les providers du kit sont listés dans `adonisrc.ts` (`providers`). `providers/api_provider.ts` du kit en est un exemple (ajoute `ctx.serialize`).
- `node ace make:preload nom` : fichier chargé au démarrage (`preloads` dans `adonisrc.ts`, ex. `start/routes.ts`, `start/kernel.ts`).
- Les **hooks du processus de build** (`indexEntities`, `generateRegistry`...) ne sont pas des providers : voir `tuyau-transformers.md`.

## 6. Événements (émetteur)

```bash
node ace make:event UserRegistered
node ace make:listener SendWelcomeEmail
```

Les événements typés sont indexés dans `.adonisjs/server/events.ts` (`#generated/events`). Utilisation : `await emitter.emit(events.UserRegistered, { user })` ; test avec `emitter.fake()` (voir `testing.md`). Les fichiers `events.ts` et `listeners.ts` du kit sont générés : ne pas les éditer.

## 7. REPL

`node ace repl` charge l'application : utile pour essayer un modèle ou un service (`loadModels()`, `await models.User.all()`). À utiliser en développement uniquement.

## 8. Vérification

```bash
node ace list | grep -E "make:|todos"       # générateurs et commandes du projet
node ace todos:count --help                 # aide générée depuis les décorateurs
npm run typecheck                           # les décorateurs et imports d'alias sont typés
node ace test unit
```
