---
name: adonisjs-v7-skill
description: 'Assistant pour créer et configurer une application AdonisJS v7, choisir le frontend, gérer la base de données, Docker, les variables d environnement et les références de fonctionnalités. Use when asked to scaffold an AdonisJS v7 project, configure database or realtime frontend, build feature workflows, or add documentation references.'
---

# Skill AdonisJS v7

Ce skill guide la création et l'évolution d'une application AdonisJS v7 : choix du kit, base de données, Boosted (Design System Orange), layouts Inertia, authentification et autorisation, fonctionnalités CRUD, tests, sécurité, erreurs et logs, commandes et services, temps réel (SSE), Docker. Le détail est dans `references/` ; ce fichier donne le déroulé et renvoie vers la bonne référence.

**Règle** : ne jamais deviner une API v7 (les données d'entraînement sont surtout v5/v6). En cas de doute, lire la page officielle (`references/official-docs.md`), vérifier dans `node_modules`, puis tester.

## When to Use This Skill

- L'utilisateur souhaite démarrer un projet AdonisJS v7
- Il doit choisir entre Hypermedia, Vue, React ou API
- Il veut une base de données autre que SQLite (PostgreSQL, MySQL, MariaDB)
- Il demande une configuration Docker ou un exemple de `.env`
- Il veut ajouter Boosted/Orange, des layouts, une fonctionnalité CRUD, des tests ou du temps réel

## Prerequisites

- Node.js >= 24 et npm
- Optionnel : Docker pour la base de données ou l'image de production

## Index des références

| Sujet | Fichier |
|---|---|
| Boosted + Vite (SCSS ou CSS, polices Orange, pièges `@popperjs/core`) | `references/boosted-vite.md` |
| Layouts Inertia, `<Link>`/`<Form>` typés, toasts flash | `references/inertia-layouts.md` |
| Auth fournie par le kit, modèle `User`, middlewares, remember me | `references/auth-starter.md` |
| Moteur de base, migrations, `schema.ts` généré, seeders, catégories | `references/database-migrations.md` |
| Tuyau (routes typées), transformers, types `Data` | `references/tuyau-transformers.md` |
| Docker : base locale, image de production, compose | `references/docker.md` |
| Fonctionnalité CRUD complète (exemple Todos) | `references/feature-guide.md` |
| Tests : plugins, base de test, factories, HTTP, navigateur, console, doubles | `references/testing.md` |
| Autorisation : abilities, policies Bouncer | `references/authorization-bouncer.md` |
| Sécurité : middlewares, limitation de débit, CSRF, secrets | `references/security-middleware.md` |
| Erreurs, exceptions personnalisées, logger, health checks | `references/errors-logging.md` |
| Architecture : quand ajouter service/repository/événement/contrat, transactions, événements v7 | `references/architecture.md` |
| Commandes Ace, services, injection de dépendances | `references/ace-services.md` |
| Paquets optionnels (mail, cache, queue, drive, i18n, redis) | `references/optional-packages.md` |
| Documentation officielle : URL vérifiées, pièges, changements v6 → v7 | `references/official-docs.md` |
| Temps réel avec Transmit (SSE) | `references/sse-guide.md` |
| Migration v6 → v7 | `references/adonis-v6-to-v7-upgrade-agent.md` |

## Step-by-Step Workflows

### 1. Créer le projet

```bash
npm create adonisjs@latest <nom-du-projet> -- --kit=<kit>
```

Flags acceptés : `--kit`, `--git-init`, `--pkg=<npm|pnpm|yarn>`, `--skip-migrations`, `--verbose`. Il n'existe **pas** de `--db`, `--adapter` ni `--auth-guard` : la base se change après la création (étape 3).

Kits : `hypermedia` (full-stack rendu serveur), `vue`, `react` (Inertia), `api` (sans interface).

Le kit `vue` génère déjà : Inertia + Vue 3, l'authentification session (modèle `User`, `SessionController`, `NewAccountController`, validateurs VineJS), la migration `users`, des layouts et un CSS maison (`inertia/css/app.css`), une page `home`, Tuyau et les transformers. **Inertia est déjà installé**, ne pas le rajouter. Voir `auth-starter.md` et `tuyau-transformers.md`.

### 2. Frontend

- **Boosted (Orange / Bootstrap 5)** : `npm i boosted @popperjs/core` (le second est **obligatoire**, sinon Vite échoue). Deux variantes (SCSS avec police Orange, ou CSS précompilé) et les pièges (le CSS du kit dont dépendent les pages `login`/`dashboard`) sont dans `references/boosted-vite.md`.
- **Layouts et navigation** : `references/inertia-layouts.md` (layout persistant via `defineOptions`, toasts flash lus dans `usePage().flash`, `<Link route>` et `<Form route>`).
- **Temps réel (SSE)** : demander à l'utilisateur s'il veut des mises à jour temps réel ; si oui, suivre `references/sse-guide.md` (AdonisJS Transmit).

### 3. Base de données

**Par défaut le kit utilise SQLite** (`better-sqlite3`, `tmp/db.sqlite3`, aucune variable `DB_*`). Pour PostgreSQL ou MySQL/MariaDB (`pg` / `mysql2`) :

1. installer le pilote et adapter `config/database.ts` ;
2. **déclarer les variables `DB_*` dans `start/env.ts`** ;
3. renseigner `.env` et `.env.example` ;
4. démarrer la base (`docker.md`) puis `node ace migration:run` (génère aussi `database/schema.ts`, dont dépendent les modèles).

Détail complet : `references/database-migrations.md`.

Nom de base par défaut : `{{projectName}}-db`. Demander à l'utilisateur si ce nom lui convient, sinon utiliser sa valeur.

### 4. Docker

Exemples validés dans `references/docker.md` : MariaDB, PostgreSQL, MySQL (développement) et Dockerfile de production (`node ace build`, testé). Commande de base : `docker compose up -d --wait`.

### 5. Fonctionnalité métier

Suivre `references/feature-guide.md` : migration, `migration:run`, modèle, validateurs, contrôleur, routes, page Inertia, tests. Les évolutions courantes sont dans `database-migrations.md` (catégories, seeders) et `auth-starter.md` (remember me).

Pour protéger les données : filtre par propriétaire ou policy (`authorization-bouncer.md`), limitation de débit sur les routes sensibles (`security-middleware.md`). Pour la logique réutilisable : service injecté (`ace-services.md`) ; pour décider s'il faut un service, un repository, des événements ou un contrat : `architecture.md` (échelle de décision, ne pas tout structurer par principe). Pour les erreurs métier : exception personnalisée (`errors-logging.md`).

### 6. Tests

`references/testing.md` : installer `@japa/api-client` et `playwright`, enregistrer les plugins (dont `shieldApiClient()` pour `.withCsrfToken()`), utiliser une base de test distincte, `testUtils.db().truncate()` entre les tests, et **tester le refus autant que le succès**.

## Variables d'environnement

Pour une base non SQLite : `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_DATABASE`. **Toute** variable lue par `env.get()` doit être déclarée dans `start/env.ts` : règle unique et détail dans `references/database-migrations.md`, section 8.

## Conseils pratiques

- SQLite (défaut, zéro configuration) pour les prototypes ; PostgreSQL ou MariaDB/MySQL pour la production.
- Ordre à respecter : migration → `node ace migration:run` → modèle (le modèle importe `#database/schema`).
- Une soumission de formulaire **redirige** (302) ; un refus Bouncer répond 302 en HTML et 403 en JSON ; `report()` d'une exception n'est pas appelé pour les statuts 400, 401, 422 (voir `errors-logging.md`).
- Événements v7 : `emitter.on(events.X, [listeners.Y])` avec un **tableau**, `start/events.ts` à créer par `make:preload events --register`, barrels générés au démarrage de `serve` (`architecture.md`).
- Transaction dans le service, jamais d'appel externe dedans ; `dispatch` après le commit (`architecture.md`).
- Le guard du kit s'appelle `web` (`auth.use('web')`), les validateurs utilisent `vine.create`.
- Ne jamais éditer les fichiers générés (`database/schema.ts`, `.adonisjs/**`).
- Ne jamais renvoyer un modèle Lucid brut à Inertia : passer par un transformer.
- Valider avec `npm run typecheck` (tsc + vue-tsc), `node ace list:routes` et `node ace test unit functional`.
- Installer un paquet par `node ace add` (pas `npm i` seul) et relire `git diff` après.

## References

- AdonisJS official documentation: <https://docs.adonisjs.com> ; index pour LLM : <https://docs.adonisjs.com/llms.txt>
- Boosted : <https://boosted.orange.com/docs/5.3/getting-started/vite/>
