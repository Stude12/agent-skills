# Sécurité : middlewares, limitation de débit, CSRF, secrets

Validé sur le kit Vue v7 (limiteur installé et testé : 429 observé). Documentation officielle : `official-docs.md` (Security, Basics > Middleware).

## 1. Middlewares : les trois piles

`start/kernel.ts` du kit :

```ts
server.use([...])   // avant le routeur, pour TOUTES les requêtes (même 404)
  // container_bindings, static, cors, vite, inertia_middleware

router.use([...])   // pour toute requête qui matche une route
  // bodyparser, session, shield (CSRF), initialize_auth, silent_auth
  // (+ initialize_bouncer si Bouncer est installé)

export const middleware = router.named({ guest: ..., auth: ... })   // middlewares nommés, à la demande
```

- **Ordre** : celui du tableau. `session` doit précéder `shield` et `initialize_auth` (ils lisent la session).
- Un middleware nommé s'applique ensuite : `.use(middleware.auth())`, `.use(middleware.guest())`, ou avec option : `middleware.auth({ guards: ['web'] })`.
- Générer : `node ace make:middleware nom` (puis l'enregistrer dans `start/kernel.ts`).
- Forme :

```ts
import type { HttpContext } from '@adonisjs/core/http'
import type { NextFn } from '@adonisjs/core/types/http'

export default class MyMiddleware {
  async handle(ctx: HttpContext, next: NextFn) {
    // avant le contrôleur
    const output = await next()
    // après le contrôleur
    return output
  }
}
```

Pour un middleware **appliqué à un groupe de routes**, utiliser `router.group(...).use(...)` (voir `auth-starter.md`).

## 2. Limitation de débit (`@adonisjs/limiter`)

```bash
node ace add @adonisjs/limiter --store=database    # ou --store=redis
```

**Valeurs acceptées par `--store` : `redis` ou `database` uniquement.** `--store=memory` échoue (`Unable to configure @adonisjs/limiter`). Le store `memory` est déjà défini dans `config/limiter.ts` à côté du store choisi : on le sélectionne ensuite avec `LIMITER_STORE=memory` dans `.env`.

Ce que la configuration a créé :

- `config/limiter.ts` (stores `database` et `memory`, défaut = `env.get('LIMITER_STORE')`) ;
- `start/limiter.ts` (middleware `throttle` global : 10 requêtes par minute par IP) ;
- `adonisrc.ts` : `limiter_provider` ;
- `LIMITER_STORE` dans `.env` et déclaré dans `start/env.ts` ;
- une migration `create_rate_limits_table` (store `database`).

### Définir et appliquer un throttle

```ts
// start/limiter.ts
import limiter from '@adonisjs/limiter/services/main'

export const loginThrottle = limiter.define('login', (ctx) => {
  return limiter
    .allowRequests(3)
    .every('1 minute')
    .usingKey(`login_${ctx.request.ip()}`)
})
```

```ts
// start/routes.ts
import { loginThrottle } from '#start/limiter'

router.post('login', [controllers.Session, 'store']).use(loginThrottle)
```

Options utiles : `.blockFor('30 mins')` (rallonge le blocage si l'utilisateur insiste), `.store('redis')`, `.usingKey(...)` (par utilisateur, clé API...). Limites dynamiques : la callback de `limiter.define` reçoit le `ctx` (utilisateurs connectés vs invités).

### Réponse observée (testée)

| Requête | Résultat après dépassement |
|---|---|
| Formulaire HTML | **429**, en-tête `Retry-After: 60` |
| `Accept: application/json` | **429**, `{"errors":[{"message":"Too many requests","retryAfter":60}]}` + `Retry-After` |

Les 3 premières requêtes sont traitées normalement (ici : redirection 302 pour un formulaire invalide, 422 en JSON). Le 4e appel déclenche l'exception `E_TOO_MANY_REQUESTS`.

### Tests

```ts
import limiter from '@adonisjs/limiter/services/main'

group.each.setup(async () => {
  await limiter.clear()
  return async () => { await limiter.clear() }   // aussi après : le store mémoire est partagé par la suite
})
```

Sans nettoyage après le test, les tests suivants héritent de l'IP déjà limitée (le test navigateur de connexion a expiré pour cette raison). Prévoir `LIMITER_STORE=memory` dans `.env.test` pour ne pas dépendre d'une table. Voir `testing.md`, section 9.

### Cas d'usage

- **Connexion / inscription** : 3 à 5 tentatives par minute et par IP, clé combinée IP + email si besoin.
- **Création de ressources** (ex. todos) : limite par utilisateur (`user_${id}`).
- **API publique** : limite globale plus large, `blockFor` pour les abus.

## 3. CSRF (`@adonisjs/shield`)

- Activé par défaut dans `config/shield.ts` (`csrf.enabled: true`). Les composants `<Form route="...">` d'Inertia ajoutent le jeton automatiquement.
- Un `fetch`/`axios` manuel doit envoyer le jeton. Shield le lit d'abord dans le champ `_csrf`, puis dans l'en-tête `X-CSRF-TOKEN`. Avec `enableXsrfCookie: true` (valeur du kit), il pose aussi un cookie chiffré `XSRF-TOKEN` que des bibliothèques comme Axios renvoient automatiquement en `X-XSRF-TOKEN`.
- Exclure des routes (webhooks) : `csrf.exceptRoutes` dans `config/shield.ts`. Chaque entrée doit correspondre **exactement** au motif de la route (`/payments/:id/callback`) ; **les jokers ne sont pas pris en charge** (utiliser une fonction pour exclure un groupe).
- Valeurs du kit (`config/shield.ts`) : `csrf.methods: ['POST','PUT','PATCH','DELETE']`, `csp.enabled: false`, `xFrame: DENY`, `hsts: 180 days`, `contentTypeSniffing` activé.
- En test, `.withCsrfToken()` nécessite le plugin `shieldApiClient()` (voir `testing.md`). Sans jeton valide, une requête de formulaire reçoit **302** (redirection avec erreur), pas 403.
- Le même package pose les en-têtes de sécurité (HSTS, X-Frame-Options, no-sniff ; la CSP est **désactivée** par défaut dans le kit) : régler `config/shield.ts`, pas des middlewares maison.

## 4. Secrets et configuration

- `.env` n'est jamais versionné (`.gitignore` du kit) ; `.env.example` liste les variables **sans** valeurs sensibles.
- `APP_KEY` : `node ace generate:key`. Unique par environnement, jamais commité.
- Toute variable lue par `env.get()` doit être déclarée dans `start/env.ts` (règle complète : `database-migrations.md`, section 8).
- Mots de passe : hachés automatiquement par `withAuthFinder(hash)` sur le modèle `User`. Ne jamais renvoyer un modèle brut au frontend : passer par un transformer (voir `tuyau-transformers.md`).
- Docker : ne pas copier `.env` dans l'image (`.dockerignore`), voir `docker.md`.

## 5. CORS

`config/cors.ts` existe dans le kit. Pour un frontend Inertia servi par la même origine, les valeurs par défaut conviennent. Pour une API consommée par un autre domaine : `origin`, `methods`, `credentials`. Voir le skill `cors-explained`.

## 6. Checklist avant mise en production

- [ ] `throttle` sur connexion, inscription et toute route coûteuse ;
- [ ] `csrf.enabled: true` et aucune route sensible dans `exceptRoutes` ;
- [ ] policies Bouncer sur toute route qui modifie une ressource d'autrui (voir `authorization-bouncer.md`) ;
- [ ] `APP_KEY` et mots de passe de base hors du dépôt ;
- [ ] `LIMITER_STORE=redis` ou `database` (pas `memory`) si plusieurs instances.
