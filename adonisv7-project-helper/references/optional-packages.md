# Paquets optionnels : quand les ajouter et comment

Le kit Vue n'installe **aucun** de ces paquets. Chacun s'ajoute avec `node ace add <paquet>`, qui installe puis configure (provider dans `adonisrc.ts`, fichier de configuration, variables d'environnement validées dans `start/env.ts`, migration éventuelle).

**Statut de validation** : seuls `@adonisjs/bouncer`, `@adonisjs/limiter` et `health_checks` ont été installés et testés ici (voir `authorization-bouncer.md`, `security-middleware.md`, `errors-logging.md`). Pour tous les autres, les commandes et exemples ci-dessous sont **repris de la documentation officielle v7 lue le jour de la rédaction, sans avoir été exécutés** : les relire dans la page officielle indiquée avant de les appliquer.

| Besoin | Paquet | Commande | Page officielle |
|---|---|---|---|
| Autorisation (roles, ownership) | `@adonisjs/bouncer` | `node ace add @adonisjs/bouncer` | `guides/auth/authorization.md` **(testé)** |
| Limitation de débit | `@adonisjs/limiter` | `node ace add @adonisjs/limiter --store=database` | `guides/security/rate-limiting.md` **(testé)** |
| Sondes de santé | (cœur) | `node ace configure health_checks` | `guides/digging-deeper/health-checks.md` **(testé)** |
| Temps réel (SSE) | `@adonisjs/transmit` | `node ace add @adonisjs/transmit` | `guides/digging-deeper/server-sent-events.md` (voir `sse-guide.md`) |
| E-mails | `@adonisjs/mail` | `node ace add @adonisjs/mail` | `guides/digging-deeper/mail.md` |
| Cache | `@adonisjs/cache` | `node ace add @adonisjs/cache` | `guides/digging-deeper/cache.md` |
| Tâches en arrière-plan | `@adonisjs/queue` | `node ace add @adonisjs/queue` | `guides/digging-deeper/queues.md` |
| Fichiers (disque, S3...) | `@adonisjs/drive` | `node ace add @adonisjs/drive` | `guides/digging-deeper/drive.md` |
| Traductions | `@adonisjs/i18n` | `node ace add @adonisjs/i18n` | `guides/digging-deeper/i18n.md` |
| Redis | `@adonisjs/redis` | `node ace add @adonisjs/redis` | `guides/database/redis.md` |
| Verrous atomiques | `@adonisjs/lock` | (voir page) | `guides/digging-deeper/locks.md` |
| OAuth / réseaux sociaux | `@adonisjs/ally` | (voir page) | `guides/auth/social-authentication.md` |
| Jetons d'API | (auth) | (voir page) | `guides/auth/access-tokens-guard.md` |

Toutes les pages : `https://docs.adonisjs.com/<chemin>` (voir `official-docs.md`).

## Règles communes

1. **Toujours** passer par `node ace add` (pas `npm i` seul) : c'est lui qui enregistre le provider et crée la configuration.
2. Lire ce que la commande a modifié (`git diff`) : elle touche `adonisrc.ts`, `start/env.ts`, `start/kernel.ts`, `.env`, parfois `database/migrations/`. Pour toute variable ajoutée, appliquer la règle de `database-migrations.md` (section 8).
3. **Un store `memory` en test**, un vrai store (`database` ou `redis`) en production. Sans cela les tests partagent un état (cas rencontré avec le limiteur : un test qui épuise la limite fait échouer les suivants ; voir `testing.md`, section 9).
4. Les fakes de test existent pour `mail`, `hash`, `emitter`, `drive` : `using fake = mail.fake()` (doc « Test doubles »). Ne pas appeler un vrai fournisseur d'e-mail ou de stockage depuis les tests.
5. Pour la production, vérifier que la variable de store (`LIMITER_STORE`, `CACHE_STORE`...) ne vaut pas `memory` quand l'application tourne sur plusieurs instances.
6. Après `add`, lancer `node ace list` : les nouvelles commandes (`make:mail`, `make:job`, `make:policy`...) apparaissent.

## E-mails (`@adonisjs/mail`)

Extrait de la documentation :

```ts
import mail from '@adonisjs/mail/services/main'

await mail.send((message) => {
  message.to(user.email).from('noreply@example.com').subject('Welcome')
})
```

- `mail.send` envoie tout de suite ; `mail.sendLater` passe par une file (nécessite `@adonisjs/queue`).
- Les transports (SMTP, Resend, SES...) se choisissent à l'installation : `node ace add @adonisjs/mail --transports=smtp`.
- En test : `using fake = mail.fake()` puis assertions sur les messages envoyés (doc « Test doubles »).
- Le dossier des classes d'e-mail est `app/mails/` (alias `#mails/*` déjà déclaré dans `package.json`).

## Cache (`@adonisjs/cache`)

Extrait de la documentation :

```ts
import cache from '@adonisjs/cache/services/main'

const posts = await cache.getOrSet({
  key: `posts:page:${page}`,
  ttl: '10m',
  factory: () => Post.query().paginate(page, 20),
})
```

- Choisir le pilote à l'installation ; avec `database`, la commande crée une migration pour la table de cache : lancer `node ace migration:run` avant usage.
- **Invalider** le cache quand la donnée change (sinon l'utilisateur voit du périmé) : c'est la partie qui coûte, pas l'ajout de `getOrSet`.
- Ne pas mettre en cache des données propres à un utilisateur sous une clé partagée.

## Tâches en arrière-plan (`@adonisjs/queue`)

Extrait de la documentation :

```bash
node ace make:job process_payment      # app/jobs/process_payment.ts
```

```ts
import { Job } from '@adonisjs/queue'
import type { JobOptions } from '@adonisjs/queue/types'

export default class ProcessPayment extends Job<{ orderId: number }> {
  static options: JobOptions = { queue: 'default', maxRetries: 3 }

  async execute() {
    const { orderId } = this.payload
    // travail à effectuer
  }

  async failed(error: Error) { /* journaliser l'échec définitif */ }
}
```

Un job reçoit un **payload sérialisable** (identifiants, pas de modèle Lucid ni de `HttpContext`). Prévoir un processus « worker » séparé en production (voir la page officielle pour le lancement).

## Fichiers (`@adonisjs/drive`)

Drive étend `MultipartFile` avec `moveToDisk(key, disk?)`. Générer un nom unique (UUID) pour éviter les collisions, valider le fichier (taille, extension) avec VineJS (`vine.file({...})`) et ne jamais faire confiance au nom fourni par le client. Pour S3 : `node ace add @adonisjs/drive --services=s3`. Pour Supabase, passer `moveAs: 'buffer'` (limite documentée du streaming).

## i18n (`@adonisjs/i18n`)

Traductions dans `resources/lang/<locale>/*.json` (ou `.yaml`), lecture par `i18n.t('messages.greeting')`, avec `i18n.t(clé, valeurs, valeurParDéfaut)`. Utile si l'interface doit exister en français et en anglais ; le tutoriel actuel n'en a pas besoin.

## Ce qu'il ne faut pas ajouter « par réflexe »

- **Redis** : seulement si on a besoin d'un store partagé (limiteur multi-instances, cache, queue). Pour une application unique avec une base SQL, `database` ou `memory` suffisent.
- **Queue** : seulement pour un travail lent (e-mail, traitement). Une action de 50 ms n'en a pas besoin.
- **Cache** : seulement après avoir mesuré un besoin.
- Les imports de services suivent le motif `@adonisjs/<paquet>/services/main` (vérifié : `@adonisjs/lock/services/main`, `@adonisjs/mail/services/main`, `@adonisjs/cache/services/main`, `@adonisjs/limiter/services/main`) ; ceux du cœur sont `@adonisjs/core/services/<nom>` avec seulement : `ace`, `app`, `config`, `dumper`, `emitter`, `encryption`, `hash`, `logger`, `repl`, `router`, `server`, `test_utils`, `url_builder` (liste lue dans `node_modules/@adonisjs/core/build/services/`). Voir `official-docs.md` pour les imports erronés fréquents.
