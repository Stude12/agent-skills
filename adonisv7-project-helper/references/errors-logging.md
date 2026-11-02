# Erreurs, exceptions personnalisées et logs

Validé sur le kit Vue v7 (16 tests fonctionnels verts, comportements observés par sondes). Documentation officielle : `official-docs.md` (Basics > Exception handling, Digging deeper > Logger, Health checks).

## 1. Le gestionnaire global du kit

`app/exceptions/handler.ts` :

```ts
export default class HttpExceptionHandler extends ExceptionHandler {
  protected debug = !app.inProduction           // Youch (page/JSON détaillés) hors production
  protected renderStatusPages = app.inProduction // pages d'erreur du frontend seulement en production

  protected statusPages: Record<StatusPageRange, StatusPageRenderer> = {
    '404': (_, { inertia }) => inertia.render('errors/not_found', {}),
    '500..599': (_, { inertia }) => inertia.render('errors/server_error', {}),
  }

  async handle(error: unknown, ctx: HttpContext) { return super.handle(error, ctx) }
  async report(error: unknown, ctx: HttpContext) { return super.report(error, ctx) }
}
```

Constats observés (en environnement non production) :

- une route inconnue renvoie **404**, et en JSON le corps contient `message`, `name` **et `frames` (la stack complète)** : c'est `debug` actif. En production, seul le message est renvoyé ;
- `renderStatusPages` vaut `false` hors production : **la page `errors/not_found` n'apparaît qu'en production**. Pour la voir en développement, mettre temporairement `renderStatusPages = true` ;
- les pages d'erreur sont des composants Inertia (`inertia/pages/errors/not_found.vue`, `server_error.vue`).

## 2. Exception personnalisée

```bash
node ace make:exception TodoLimitReached     # app/exceptions/todo_limit_reached_exception.ts
```

Le générateur produit seulement `static status = 500`. Version complète (validée) :

```ts
import { Exception } from '@adonisjs/core/exceptions'
import type { HttpContext } from '@adonisjs/core/http'

export default class TodoLimitReachedException extends Exception {
  static status = 409
  static code = 'E_TODO_LIMIT_REACHED'

  constructor() {
    super('You reached the maximum number of todos')
  }

  // appelée avant handle() : journalisation
  report(error: this, { logger, auth }: HttpContext) {
    logger.warn({ userId: auth.user?.id, code: error.code }, 'todo limit reached')
  }

  // remplace le gestionnaire global pour cette exception
  async handle(error: this, ctx: HttpContext) {
    if (ctx.request.accepts(['html', 'json']) === 'json') {
      return ctx.response.status(error.status).send({ code: error.code, message: error.message })
    }
    ctx.session.flash('error', error.message)
    return ctx.response.redirect().back()
  }
}
```

Usage : `throw new TodoLimitReachedException()` depuis un service ou un contrôleur ; aucun `try/catch` nécessaire.

### Piège : `report()` n'est pas appelé pour 400, 401 et 422

Le gestionnaire ignore par défaut ces statuts pour le **reporting** (`ignoreStatuses = [400, 422, 401]`, lu dans `@adonisjs/http-server`). Une exception en `static status = 422` est bien gérée (`handle` s'exécute) mais `report()` **n'est jamais appelé** : mon premier essai ne journalisait rien. Avec `409` il est appelé. Choisir un autre statut, ou ajuster la liste :

```ts
// app/exceptions/handler.ts
protected ignoreStatuses = [400, 401]            // retire 422 de la liste
protected ignoreCodes = ['E_TODO_LIMIT_REACHED'] // ou ignorer volontairement un code
```

Les erreurs de validation VineJS (422) ne sont donc pas journalisées par défaut, ce qui est voulu.

### Réponses selon le client (testées)

| Requête | Résultat avec le `handle` ci-dessus |
|---|---|
| `Accept: application/json` | **409** + `{ code, message }` |
| Formulaire HTML | **302** retour arrière, message dans le flash `error` |

Le message flashé est affiché par `flash_toasts.vue` du kit (voir `inertia-layouts.md`).

## 3. Contexte ajouté aux logs d'erreur

```ts
protected context(ctx: HttpContext) {
  return {
    requestId: ctx.request.id(),
    userId: ctx.auth?.user?.id,   // `?.` : l'auth peut ne pas être initialisé sur une 404
    ip: ctx.request.ip(),
  }
}
```

`ctx.auth` n'est pas toujours défini (par exemple pour une 404 avant le middleware d'auth) : utiliser `ctx.auth?.user`.

## 4. Gérer un type d'erreur précis globalement

```ts
import { errors as vineErrors } from '@vinejs/vine'

async handle(error: unknown, ctx: HttpContext) {
  if (error instanceof vineErrors.E_VALIDATION_ERROR) {
    return ctx.response.status(422).send({ errors: error.messages })
  }
  return super.handle(error, ctx)
}
```

Pour Inertia, laisser le comportement par défaut : les erreurs de validation sont renvoyées dans `errors` (props partagées) et affichées par `<Form>`.

## 5. Logger

Configuration du kit (`config/logger.ts`) : Pino ; sortie jolie hors production (`targets.pretty()`), fichier/stdout en production (`targets.file({ destination: 1 })`) ; niveau = `LOG_LEVEL` (déclaré dans `start/env.ts`).

```ts
// dans un contrôleur ou un handler : logger lié à la requête (avec request id)
async store({ logger, auth }: HttpContext) {
  logger.info({ userId: auth.user?.id }, 'Creating todo')
}

// hors requête (service, commande)
import logger from '@adonisjs/core/services/logger'
logger.error({ err }, 'Something failed')
```

- **Préférer `ctx.logger`** dans une requête : il porte le `request_id`. Les logs de test montrent par exemple `request_id: "..."` sous chaque ligne `WARN`.
- Niveaux : `trace`, `debug`, `info`, `warn`, `error`, `fatal`. En production : `LOG_LEVEL=info` ou `warn`.
- Messages à gabarit : `logger.info('User %s logged in', user.id)`.
- Ne jamais journaliser de mot de passe, de jeton ni de modèle `User` complet (`auth.user` contient le hash).
- Dans les tests, les logs apparaissent dans la sortie : pour les masquer, `LOG_LEVEL=silent` dans `.env.test` (vérifié : le test passe et plus aucune ligne `WARN` n'est affichée).

## 6. Health checks (optionnel)

```bash
node ace configure health_checks
```

```ts
// start/health.ts
import { HealthChecks, DiskSpaceCheck, MemoryHeapCheck } from '@adonisjs/core/health'
export const healthChecks = new HealthChecks().register([new DiskSpaceCheck(), new MemoryHeapCheck()])
```

Le contrôleur généré (`app/controllers/health_checks_controller.ts`) expose **deux** méthodes : `live` (le processus répond, sans vérifier de dépendance) et `ready` (exécute les checks). Les routes ne sont **pas** ajoutées automatiquement :

```ts
// start/routes.ts
router.get('/health/live', [controllers.HealthChecks, 'live'])
router.get('/health/ready', [controllers.HealthChecks, 'ready'])
```

**Piège de typage** : le contrôleur généré contient `return response.ok()` dans `live`, ce qui échoue à `npm run typecheck` (`TS2554: Expected 1-2 arguments, but got 0`, `response.ok` exige un corps ; le même appel figure dans la doc officielle). Corriger par `return response.ok({ status: 'ok' })` (vérifié : typecheck propre).

Vérifié par test : `/health/live` et `/health/ready` répondent 200 ; le corps de `ready` contient `isHealthy`, `status`, `finishedAt`, `debugInfo`, `checks`. Le contrôleur généré renvoie `response.serviceUnavailable(report)` (503) quand `report.isHealthy` est faux ; ce cas n'a pas été provoqué en test.

Utile pour la sonde de santé Docker/Kubernetes (`healthcheck` du compose, voir `docker.md`). Exposer la route sans authentification, mais **sans** détails sensibles en production, ou la protéger par un en-tête secret.

## 7. Vérification

```bash
node ace test functional      # les tests couvrent 409 JSON, redirection HTML, flash
node ace serve --hmr          # tester /inexistant : page de debug (Youch) hors production
NODE_ENV=production node ace build && node build/bin/server.js   # page errors/not_found réelle
```
