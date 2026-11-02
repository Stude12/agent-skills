# Guide SSE pour le frontend avec AdonisJS Transmit

Ce guide décrit comment utiliser AdonisJS Transmit pour implémenter des mises à jour temps réel côté frontend via Server-Sent Events (SSE).

## 1. Installer Transmit

```bash
node ace add @adonisjs/transmit
npm install @adonisjs/transmit-client
```

- `@adonisjs/transmit` configure le serveur pour diffuser des événements SSE.
- `@adonisjs/transmit-client` permet au frontend de se connecter et d écouter les canaux.

## 2. Configurer Transmit côté serveur

Le fichier `config/transmit.ts` est généré automatiquement.

```ts
import { defineConfig } from '@adonisjs/transmit'

export default defineConfig({
  pingInterval: false,
  transport: null,
})
```

Pour un déploiement multi-instance, utiliser Redis :

```ts
import { defineConfig } from '@adonisjs/transmit'
import env from '#start/env'

export default defineConfig({
  pingInterval: false,
  transport: 'redis',
  redis: {
    host: env.get('REDIS_HOST'),
    port: env.get('REDIS_PORT'),
    password: env.get('REDIS_PASSWORD'),
  },
})
```

## 3. Enregistrer les routes Transmit

Dans `start/routes.ts`, enregistrer les routes Transmit et les protéger avec l authentification :

```ts
import router from '@adonisjs/core/services/router'
import { middleware } from '#start/kernel'
import transmit from '@adonisjs/transmit/services/main'

transmit.registerRoutes((route) => {
  route.middleware(middleware.auth())
})
```

## 4. Autoriser les canaux

Exemple : créer un preload `start/transmit.ts` pour autoriser les abonnements. Utiliser `node ace make:preload transmit` puis accepter l enregistrement dans `.adonisrc.ts`.

```ts
import transmit from '@adonisjs/transmit/services/main'

transmit.authorize<{ userId: string }>('todos/:userId', (ctx, { userId }) => {
  return ctx.auth.user?.id === +userId
})
```

- Le canal `todos/:userId` est dynamique.
- Seul l utilisateur propriétaire du canal peut s y abonner.
- Le preload est chargé au démarrage du serveur grâce à `.adonisrc.ts`.
- **Vérifier l'enregistrement** : `adonisrc.ts` doit contenir `() => import('#start/transmit')` dans `preloads`. S'il manque, `authorize` n'est jamais chargé et l'abonnement au canal d'un autre utilisateur est accepté (vérifié dans un navigateur : 204 sans le preload, 400 avec). `make:preload transmit` pose une question interactive : en script, ajouter `--register`.
- **Contrôle dans un navigateur** : `POST __transmit/subscribe` pour `todos/<son id>` renvoie 204 ; pour `todos/<id d'un autre>` il renvoie 400.
- **L'autorisation ne s'applique qu'à l'abonnement** : sans `registerRoutes((route) => route.middleware(middleware.auth()))`, `ctx.auth.user` n'est pas disponible dans le callback (doc officielle, « Authenticating private channels »).

## 5. Diffuser des événements depuis le serveur

Utiliser `transmit.broadcast()` après chaque action CRUD :

```ts
transmit.broadcast(`todos/${user.id}`, {
  action: 'created',
  todo: todo.toJSON(),
})
```

Exemples de diffusion :

- Création : `action: 'created'`
- Mise à jour : `action: 'updated'`
- Suppression : `action: 'deleted'`

## 6. Initialiser le client frontend

Créer `inertia/transmit.ts` :

```ts
import { Transmit } from '@adonisjs/transmit-client'

export const transmit = new Transmit({
  baseUrl: window.location.origin,
})
```

## 7. S abonner aux événements côté frontend

L'API de `@adonisjs/transmit-client` (v1.x) est : `transmit.subscription(canal)` retourne un objet `Subscription` avec `create()`, `delete()` et `onMessage(handler)`. Les méthodes `transmit.listen()` / `transmit.unsubscribe()` **n'existent pas**.

Exemple d intégration Vue :

```ts
const userId = computed(() => page.props.user?.id)
let subscription: ReturnType<typeof transmit.subscription> | null = null

onMounted(async () => {
  if (!userId.value) return
  subscription = transmit.subscription(`todos/${userId.value}`)
  subscription.onMessage((message: MessageType) => {
    // message est directement le payload passé à transmit.broadcast()
    // mettre à jour l état local en fonction de message.action
  })
  await subscription.create()
})

onBeforeUnmount(async () => {
  await subscription?.delete()
})
```

- `onMessage` reçoit directement le payload (pas d'objet `event.data`).
- Garder un état local (`items = ref([...props.todos])`) et le resynchroniser avec `watch(() => props.todos, ...)` lors des navigations Inertia.
- `transmit.on('connected' | 'disconnected' | 'reconnecting', cb)` permet de suivre l'état de la connexion.

## 8. Notes de production

- Protéger les routes Transmit avec l authentification.
- Utiliser Redis pour les déploiements multi-instance.
- Éviter les canaux publics non autorisés.
- Envoyer un payload minimal pour réduire la charge réseau.
- Sérialiser explicitement le payload (`id`, `title`, `category`...) plutôt que `model.toJSON()` quand une relation est préchargée : la forme doit être identique à celle des props Inertia du contrôleur.
