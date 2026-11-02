# Layouts Inertia (Vue)

## 1. Ce que fournit le kit

- `inertia/layouts/` : `app.vue` (pages connectées), `auth.vue` (login/signup), `marketing.vue` (pages publiques, erreurs), `settings.vue`.
- `app/middleware/inertia_middleware.ts` partage avec **toutes** les pages : `errors`, `user` (via `UserTransformer`), `preferences` (thème). Les messages flash passent par le champ `flash` de l'objet page (méthode `flash()` du middleware).
- Pas besoin de modifier `config/inertia.ts` pour partager des données : tout passe par `share()` du middleware.
- Les types `usePage().props` et `usePage().flash` sont inférés globalement (`inertia/types.ts`, namespace `Data` généré). Voir `tuyau-transformers.md`.

## 2. Deux façons d'appliquer un layout

### A. Envelopper le template (ce que fait le kit)

```vue
<script setup lang="ts">
import AuthLayout from '~/layouts/auth.vue'
import { Form, Link } from '@adonisjs/inertia/vue'
</script>

<template>
  <AuthLayout>
    <h1>Welcome back</h1>
    ...
  </AuthLayout>
</template>
```

Simple et explicite, mais le layout est remonté à chaque navigation (perte d'état, par exemple un menu ouvert).

### B. Layout persistant via `defineOptions` (recommandé par le tutoriel)

```vue
<script setup lang="ts">
import DefaultLayout from '~/layouts/default.vue'
defineOptions({ layout: DefaultLayout })
</script>

<template>
  <!-- contenu seul : ne PAS entourer avec <DefaultLayout> -->
  <div class="container">...</div>
</template>
```

Le layout survit aux navigations Inertia (état conservé, transitions possibles). Ne pas mélanger A et B sur la même page, sous peine d'afficher le layout deux fois.

> `defineOptions({ layout })` est le mécanisme Vue 3 / Inertia pour les layouts persistants. Il n'est pas utilisé par les pages du kit : à tester sur le projet avant de généraliser (`npm run typecheck` et vérifier visuellement qu'un seul Header apparaît).

## 3. Autres approches (cas particuliers)

| Approche | Où | Layout décidé par | Complexité |
|---|---|---|---|
| `defineOptions` dans la page | Page Vue | chaque page | simple |
| 3e argument de `resolvePageComponent` dans `inertia/app.ts` | entry point | liste centralisée (`guestPages`) | moyen |
| Prop `layout` renvoyée par le contrôleur | serveur | contrôleur | avancé |
| Middleware qui `inertia.share({ layout })` | serveur | état d'authentification | avancé |

Exemple `app.ts` (liste centralisée) :

```ts
const guestPages = ['home', 'auth/login', 'auth/signup']

resolve: (name) => {
  const pages = import.meta.glob<DefineComponent>('./pages/**/*.vue')
  const layout = guestPages.includes(name) ? GuestLayout : DefaultLayout
  return resolvePageComponent(`./pages/${name}.vue`, pages, layout)
},
```

Inconvénient : la liste est codée en dur. Les approches « contrôleur » et « middleware » mélangent présentation et serveur : à éviter sauf besoin réel.

## 4. Layouts `default` (authentifié) et `guest` (public) du tutoriel

- `default.vue` : `Header` + `<main class="flex-grow-1"><slot/></main>` + `Footer`, dans `<div class="d-flex flex-column min-vh-100">`, plus le `Toaster`.
- `guest.vue` : `<main><slot/></main>` + `Toaster` uniquement.

Toasts flash. Le tutoriel surveille `page.props.flash` ; **dans le kit actuel les flash sont dans `page.flash`** :

```vue
<script setup lang="ts">
import { watch } from 'vue'
import { usePage } from '@inertiajs/vue3'
import { toast, Toaster } from 'vue-sonner'

const page = usePage()

watch(
  () => page.flash,
  (flash) => {
    if (flash.error) toast.error(flash.error, { id: 'flash' })
    if (flash.success) toast.success(flash.success, { id: 'flash' })
  },
  { immediate: true }
)
</script>

<template>
  <Toaster position="top-center" rich-colors />
</template>
```

Le composant `inertia/components/flash_toasts.vue` du kit fait déjà cela (avec `toast.dismiss()` au changement d'URL) : le réutiliser plutôt que de le réécrire.

## 5. Header avec Boosted (extrait)

Points clés du tutoriel :

- `<nav class="navbar navbar-dark bg-dark navbar-expand-md">`, logo Orange (`https://boosted.orange.com/docs/5.3/assets/brand/orange-logo.svg`).
- Lien actif : `:class="{ active: page.url.startsWith('/todos') }"` avec `usePage().url`.
- Menu utilisateur géré par un `ref` (`isDropdownOpen`) plutôt que par le JS Boosted.
- Déconnexion : `router.post('/logout')` (le tutoriel) ou `<Form route="session.destroy">` (kit, CSRF géré automatiquement).
- `user` vient des props partagées : `const user = computed(() => page.props.user)`.

## 6. Footer Boosted

```vue
<template>
  <footer class="footer navbar" data-bs-theme="dark">
    <h2 class="visually-hidden">Sitemap & information</h2>
    <div class="container-xxl footer-terms">
      <ul class="navbar-nav gap-md-3">
        <li class="fw-bold">© Orange {{ new Date().getFullYear() }}</li>
        <li><a class="nav-link" href="#">Terms and conditions</a></li>
        <li><a class="nav-link" href="#">Privacy</a></li>
      </ul>
    </div>
  </footer>
</template>
```

## 7. Liens et formulaires typés (kit)

```vue
<script setup lang="ts">
import { Form, Link } from '@adonisjs/inertia/vue'
</script>

<template>
  <Link route="session.create">Sign in</Link>
  <Link route="todos.show" :params="{ id: todo.id }">Voir</Link>

  <Form route="todos.store" v-slot="{ errors, processing }">
    <input name="title" />
    <span v-if="errors.title">{{ errors.title }}</span>
    <button :disabled="processing">Add</button>
  </Form>
</template>
```

- `route` est le **nom de route** ; la méthode HTTP est déduite du nom (`*.store` → POST).
- Paramètres de route : `:params="{ id }"` sur `<Link>` **et** sur `<Form>` ; query string : `:qs="{ page: 2 }"` sur `<Link>`. (`routeParams` n'est que le nom du type TypeScript et de l'option de `useRouter().visit`, ainsi que de la prop React.) Vérifié dans `node_modules/@adonisjs/inertia/build/src/client/vue/link.d.ts`.
- Navigation programmatique typée : `useRouter()` de `@adonisjs/inertia/vue` (`visit`, `get`, `post`, `patch`, `delete` résolvent URL et méthode depuis le nom de route).
- `<Form>` ajoute le jeton CSRF automatiquement (shield est actif dans le kit).
- `<Link href="/signup">` reste accepté (variante `LinkHrefProps`), mais sans vérification de type sur les noms de routes. Cette variante interdit `route`, `params` et `qs` en même temps que `href`.

## 8. Vérification

- `/` et `/login` : layout guest (pas de Header/Footer).
- `/todos` : layout default (Header, menu utilisateur, Footer collé en bas même avec peu de contenu).
- Toasts flash visibles sur les deux layouts.
- Un seul Header affiché (sinon A et B mélangés).
