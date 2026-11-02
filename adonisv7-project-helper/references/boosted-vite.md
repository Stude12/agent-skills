# Boosted + Vite (Design System Orange, basé sur Bootstrap 5)

Boosted fournit composants UI, typographie et couleurs conformes à la charte Orange. Vite compile et sert les assets frontend du kit.

Documentation officielle : <https://boosted.orange.com/docs/5.3/getting-started/vite/>

## 1. Installer

```bash
npm i boosted @popperjs/core
npm i -D sass          # uniquement pour la variante A (SCSS)
```

`@popperjs/core` est une **peer dependency obligatoire** de Boosted (tooltips, dropdowns...). Sans elle :

- `vite build` échoue : `Rolldown failed to resolve import "@popperjs/core" from .../boosted.esm.js`
- le dev server affiche : `Failed to resolve import "@popperjs/core" from "node_modules/.vite/deps/boosted.js"`

Si l'erreur persiste alors que le paquet est installé, le cache de pré-bundling Vite est périmé (généré avant l'installation) :

```bash
rm -rf node_modules/.vite
npm run dev            # ou: npx vite --force
```

## 2. Variante A : SCSS (celle du tutoriel, avec la police Orange)

```bash
mkdir -p inertia/scss public/fonts
cp -r node_modules/boosted/dist/fonts/* public/fonts/
```

```scss
// inertia/scss/styles.scss
@use "boosted/scss/orange-helvetica" with ($font-path: "/fonts/");
@use "boosted/scss/boosted";
```

```ts
// inertia/app.ts : remplacer l'import './css/app.css'
import './scss/styles.scss'
import 'boosted' // JS des composants interactifs (dropdown, collapse, modal...)
```

```ts
// vite.config.ts : évite les warnings de dépréciation Sass venant de Boosted
css: { preprocessorOptions: { scss: { quietDeps: true } } },
```

Regroupement des polices dans `assets/fonts/` (optionnel, repris du tutoriel) :

```ts
// vite.config.ts
build: {
  rollupOptions: {
    output: {
      assetFileNames: (assetInfo: any): string => {
        if (assetInfo.names?.some((name: string) => /\.(woff2?|ttf|otf|eot)$/i.test(name))) {
          return 'assets/fonts/[name][extname]'
        }
        return 'assets/[name][extname]'
      },
    },
  },
},
```

> Le kit v7 utilise Vite 8 (Rolldown). `rollupOptions` est le nom utilisé par le tutoriel ; `rolldownOptions` est le nom natif de Vite 8. Vérifier lequel est accepté par la version installée (`npx vite build` doit passer sans warning) avant de figer l'un ou l'autre.

## 3. Variante B : CSS précompilé (sans Sass, sans police Orange)

```ts
// inertia/app.ts
import 'boosted/dist/css/boosted.min.css'
import 'boosted'
```

Plus simple, mais pas de personnalisation des variables Sass ni de police Helvetica Neue Orange.

## 4. Vérifier `vite.config.ts` du kit

```ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import adonisjs from '@adonisjs/vite/client'

export default defineConfig({
  plugins: [
    vue(),
    adonisjs({ entryPoints: ['inertia/app.ts'], reload: ['resources/views/**/*.edge'] }),
  ],
  resolve: {
    alias: {
      '~/': `${import.meta.dirname}/inertia/`,
      '@generated': `${import.meta.dirname}/.adonisjs/client/`,
    },
  },
})
```

- L'option s'appelle `entryPoints` (casse exacte) ; `entrypoints` fait échouer le build.
- Le plugin `import inertia from '@adonisjs/inertia/vite'` **n'existe pas** (`Missing "./vite" specifier in "@adonisjs/inertia" package`, vérifié dans le champ `exports`) : ne pas l'ajouter à `plugins`.
- L'alias `@generated` pointe vers `.adonisjs/client/` (registry Tuyau, types `Data`). Voir `tuyau-transformers.md`.

## 5. Vue racine Edge

`resources/views/inertia_layout.edge` doit garder `@vite(['inertia/app.ts'])`, `@inertiaHead()` et `@inertia()`. On peut y régler `lang`, le titre par défaut (`<title data-inertia>`) et retirer la lecture du cookie de thème (`app_theme`) si le thème du kit n'est plus utilisé.

## 6. Pièges

- **Le CSS du kit** (`inertia/css/app.css`, variables `--paper`, `--ink`, classes `.btn--primary`, `.field__input`, `.auth__*`, `.header--*`) alimente les pages `login`, `signup`, `dashboard`, les composants (`logo`, `nav_link`, `theme_toggle`, `flash_toasts`) et les layouts `app`, `auth`, `marketing`, `settings`. Le retirer ou ne plus l'importer laisse ces écrans non stylés. Deux options : réécrire les pages avec les classes Boosted (`btn btn-primary`, `form-control`, `container`...), ou supprimer ce qui ne sert plus.
- Les styles du kit sont aussi chargés par `vue-sonner/style.css` pour les toasts : conserver cet import dans `app.ts`.
- Les composants JS de Boosted (`data-bs-toggle`, `data-bs-target`) ne se ré-initialisent pas toujours après une navigation Inertia ; pour les menus du header, préférer un état Vue (`ref`) plutôt que le JS Boosted.
- Après `cp` des polices, vérifier que `public/fonts/` est bien versionné (il ne figure pas dans `.gitignore`, contrairement à `public/assets`).

## 7. Vérification

```bash
npm run typecheck
npx vite build      # puis: rm -rf public/assets (dossier ignoré par git)
npm run dev         # http://localhost:3333
```
