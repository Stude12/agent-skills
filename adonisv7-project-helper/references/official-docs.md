# Documentation officielle AdonisJS v7 : où chercher, quoi vérifier

**Règle** : ne jamais deviner une API AdonisJS v7. Les données d'entraînement contiennent surtout des versions 5 et 6. En cas de doute, lire la page officielle ci-dessous, puis vérifier dans `node_modules` ou par un test.

## 1. Sources

| Source | URL | Usage |
|---|---|---|
| Index pour LLM | `https://docs.adonisjs.com/llms.txt` | liste des pages avec description (104 pages au moment de la vérification) |
| Documentation complète en un fichier | `https://docs.adonisjs.com/llms-full.txt` | question transversale (volumineux) |
| Une page en Markdown | `https://docs.adonisjs.com/<chemin>.md` | lire la page exacte |
| Guide de migration v6 → v7 | `https://docs.adonisjs.com/v6-to-v7.md` | différences de version |
| Code source de la doc | `https://github.com/adonisjs/v7-docs` | contenu brut |

Les 104 URL de `llms.txt` ont été récupérées une à une : toutes répondent 200 avec un Markdown à frontmatter.

### Piège des URL

Le site renvoie **200 même pour une page qui n'existe pas** (il sert alors la page HTML d'accueil, environ 250 Ko). Vérifier le contenu, pas le code HTTP :

```bash
curl -s -L https://docs.adonisjs.com/guides/basics/routing.md | head -1     # doit afficher '---'
```

Et les chemins utilisent des **tirets**, pas des underscores, pour les noms de page composés :

| Valide | Invalide (renvoie le HTML d'accueil) |
|---|---|
| `guides/digging-deeper/mail.md` | `guides/digging_deeper/mail.md` |
| `guides/testing/api-tests.md` | `guides/testing/api_tests.md` |
| `deployment.md` | `start/deployment.md` |
| `stacks-and-starter-kits.md` | `start/pick_your_path` |

Vérifié : `guides/digging-deeper/mail` → 35 266 octets de Markdown ; `guides/digging_deeper/mail` → 254 656 octets de HTML.

## 2. Index par sujet (chemins vérifiés)

**Démarrer** : `introduction`, `installation`, `stacks-and-starter-kits`, `folder-structure`, `dev-environment`, `configuration`, `deployment`, `faqs`.

**Bases** (`guides/basics/`) : `routing`, `controllers`, `http-context`, `middleware`, `request`, `response`, `body-parser`, `validation`, `file-uploads`, `session`, `session-tagging`, `url-builder`, `exception-handling`, `debugging`, `static-file-server`.

**Front** (`guides/frontend/`) : `inertia`, `transformers`, `api-client` (Tuyau), `vite`, `edgejs`, `tanstack-query`.

**Données** (`guides/database/`) : `lucid`, `redis`.

**Auth** (`guides/auth/`) : `introduction`, `verifying-user-credentials`, `session-guard`, `access-tokens-guard`, `basic-auth-guard`, `custom-auth-guard`, `social-authentication`, `authorization`.

**Sécurité** (`guides/security/`) : `hashing`, `encryption`, `cors`, `securing-ssr-applications`, `rate-limiting`.

**Concepts** (`guides/concepts/`) : `application-lifecycle`, `dependency-injection`, `service-providers`, `container-services`, `codegen`, `barrel-files`, `assembler-hooks`, `scaffolding`, `extending-adonisjs`.

**Aller plus loin** (`guides/digging-deeper/`) : `cache`, `drive`, `emitter`, `health-checks`, `i18n`, `locks`, `logger`, `mail`, `queues`, `server-sent-events`, `opentelemetry`.

**Ace** (`guides/ace/`) : `introduction`, `creating-commands`, `arguments`, `flags`, `prompts`, `terminal-ui`, `repl`.

**Tests** (`guides/testing/`) : `introduction`, `api-tests`, `browser-tests`, `console-tests`, `resetting-state-between-tests`, `database-assertions`, `test-doubles`.

**Référence** (`reference/`) : `application`, `adonisrc-rcfile`, `commands`, `edge`, `events`, `exceptions`, `helpers`, `types-helpers`.

**Tutoriel officiel** (`tutorial/react/` et `tutorial/hypermedia/`) : `overview`, `cli-and-repl`, `database-and-models`, `routes-controller-views`, `forms-and-validation`, `styling-and-cleanup`, `authorization`. Le tutoriel officiel Inertia est en **React** ; il n'existe pas de version Vue (le dépôt `todo_list-adonis-tutoriel` en est une adaptation Vue).

## 3. Correspondance sujet → référence de ce skill

| Sujet | Référence locale | Page officielle principale |
|---|---|---|
| Auth session | `auth-starter.md` | `guides/auth/session-guard` |
| Autorisation | `authorization-bouncer.md` | `guides/auth/authorization` |
| Routes typées, transformers | `tuyau-transformers.md` | `guides/frontend/api-client`, `guides/frontend/transformers` |
| Layouts, `<Form>`, `<Link>` | `inertia-layouts.md` | `guides/frontend/inertia` |
| Migrations, seeders | `database-migrations.md` | `guides/database/lucid` |
| Tests | `testing.md` | `guides/testing/*` |
| Erreurs, logs | `errors-logging.md` | `guides/basics/exception-handling`, `guides/digging-deeper/logger` |
| Sécurité, middlewares | `security-middleware.md` | `guides/security/*`, `guides/basics/middleware` |
| Commandes, services, DI | `ace-services.md` | `guides/ace/*`, `guides/concepts/dependency-injection` |
| Architecture, événements, transactions | `architecture.md` | `guides/digging-deeper/emitter`, `guides/concepts/dependency-injection`, `guides/database/lucid` |
| Déploiement, Docker | `docker.md` | `deployment` |
| Paquets optionnels | `optional-packages.md` | `guides/digging-deeper/*` |

## 4. Pièges d'autres sources (vérifiés contre la doc et `node_modules`)

Deux skills tiers ont été lus pendant la rédaction. `omakei/adonisjs-architecture-skill` vise la **v6** : ses écarts avec la v7 sont listés et vérifiés dans `architecture.md`, section 11. `kamranallahyar/adonisjs-v7-agent-skill` contient plusieurs exemples **qui ne correspondent pas à la v7** ; ne pas les recopier :

| Exemple erroné | Réalité vérifiée |
|---|---|
| `https://docs.adonisjs.com/start/deployment`, `.../start/tutorial/...`, `.../guides/digging_deeper/mail` | chemins inexistants (renvoient le HTML d'accueil) ; voir tableau ci-dessus |
| `import { rateLimit } from '@adonisjs/core/rate_limiter'` | n'existe pas ; le limiteur est `@adonisjs/limiter` (`limiter.define`, `allowRequests().every()`) |
| `import Event from '@adonisjs/core/services/event'` ; `Event.emit(...)` | le service du cœur est `@adonisjs/core/services/emitter` (`emitter.emit`) |
| `import ... from '@adonisjs/core/services/response'` | il n'existe pas de service `response` (liste des services du cœur dans `optional-packages.md`) |
| `import { tracer } from '@adonisjs/core/services/otel'` | OpenTelemetry est le paquet `@adonisjs/otel` (`defineConfig`, `destinations`) |
| `auth.use('session')`, `login(user, { rememberMe: true })` | le guard du kit s'appelle **`web`** ; le remember me passe par `useRememberMeTokens` et `login(user, remember)` (voir `auth-starter.md`) |
| `import { schema, rules } from '@adonisjs/core/validator'` | la validation v7 est **VineJS** : `vine.create({...})` ; `schema`/`rules` sont de la v5 |
| `import { BaseModel, column } from '@adonisjs/lucid'` (modèle avec `@column` manuels) | le kit génère `XxxSchema` dans `database/schema.ts` ; le modèle n'ajoute que relations et getters (`database-migrations.md`) |
| `auth.use('session').verifyCredentials(...)` | `User.verifyCredentials(email, password)` fourni par `withAuthFinder` (`auth-starter.md`) |
| `useTransmit()` côté client | l'API du client est `transmit.subscription(canal)` puis `create()`, `onMessage()`, `delete()` (`sse-guide.md`) |
| Starter `node ace init my-app --kit=web` | création par `npm create adonisjs@latest <nom> -- --kit=<hypermedia\|vue\|react\|api>` |
| `node ace make:policy` avec Bouncer non installé | la commande n'existe qu'après `node ace add @adonisjs/bouncer` |
| Imports de composants `import { Inertia } from '@adonisjs/inertia'` | `<Form>` et `<Link>` viennent de `@adonisjs/inertia/vue` (ou `/react`) |

Le site `skills.sh` signale ce skill comme « Pass » aux audits de sécurité : cela ne garantit pas l'exactitude technique du contenu.

## 5. Autres changements v6 → v7 à connaître

Repris de `v6-to-v7.md` (lu en entier) ; voir aussi `adonis-v6-to-v7-upgrade-agent.md` pour la procédure.

- **Imports** : `@ioc:...` et `App/...` n'existent plus (ESM seulement) ; utiliser les alias `#models/*`, `#controllers/*`... de `package.json`.
- **URL** : `router.makeUrl` est déprécié → `urlFor` de `@adonisjs/core/services/url_builder` ; dans Edge, `route()` → `urlFor()`.
- **Classes HTTP** : `Request`/`Response` renommées `HttpRequest`/`HttpResponse` (macros, déclarations de module).
- **Helpers supprimés** : `getDirname`/`getFilename` → `import.meta.dirname`/`filename` ; `slash` → `stringHelpers.toUnixSlash` ; `cuid` → UUID.
- **Flash** : la clé `errors` est supprimée ; les erreurs de validation sont sous `inputErrorsBag`.
- **Hooks de l'assembler** renommés (`onBuildStarting` → `buildStarting`, `onSourceFileChanged` → `fileChanged`, `onDevServerStarted` → `devServerStarted`, `onBuildCompleted` → `buildFinished`).
- **Globs de tests** : `tests/unit/**/*.spec.{ts,js}` (et non `(.ts|.js)`).
- **Chiffrement** : `config/encryption.ts` dédié (plus de `appKey` dans `config/app.ts`).
- **Pages d'état** non rendues pour les requêtes JSON ; `request.all()` fusionne désormais champs et fichiers multipart ; les noms de routes sont **générés** depuis les contrôleurs.
- **Inertia** : `entrypoint`, `sharedData` retirés de `config/inertia.ts` ; les données partagées passent par le middleware Inertia (`app/middleware/inertia_middleware.ts`, méthode `share`).
- **Youch** n'est plus embarqué : le dépendre en `devDependency`.
- **Hooks d'arrêt** de l'application exécutés dans l'ordre inverse.

## 6. Méthode de travail recommandée

1. Chercher le sujet dans `llms.txt` (ou le tableau ci-dessus) ; lire **la** page `.md`.
2. Vérifier les imports dans `node_modules/<paquet>/package.json` (champ `exports`) ou les types `.d.ts`.
3. Écrire un test ou lancer la commande ; ne pas s'appuyer sur une seule lecture.
4. Si le skill local et la documentation divergent, **la documentation et l'exécution priment**, puis corriger la référence locale.
