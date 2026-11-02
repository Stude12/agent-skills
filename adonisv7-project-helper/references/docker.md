# Docker pour AdonisJS v7

Deux usages distincts : (1) une base de données locale pour le développement (le plus courant), (2) une image de l'application pour la production.

## 1. Base locale pour le développement

### MariaDB (pilote `mysql2`, connexion `mysql`) : testé

```yaml
# docker-compose.yml
services:
  db:
    image: mariadb:11
    restart: unless-stopped
    environment:
      MARIADB_ROOT_PASSWORD: root
      MARIADB_DATABASE: <projet>-db
      MARIADB_USER: adonis
      MARIADB_PASSWORD: secret
    ports:
      - '3306:3306'
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ['CMD', 'healthcheck.sh', '--connect', '--innodb_initialized']
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  db_data:
```

```bash
docker compose up -d --wait     # attend le healthcheck
node ace migration:run
docker compose down             # garde les données ; ajouter -v pour effacer le volume
```

`.env` correspondant : `DB_HOST=127.0.0.1`, `DB_PORT=3306`, `DB_USER=adonis`, `DB_PASSWORD=secret`, `DB_DATABASE=<projet>-db`.

### PostgreSQL

```yaml
services:
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: adonis
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: <projet>-db
    ports:
      - '5432:5432'
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U adonis -d <projet>-db']
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  db_data:
```

`.env` : `DB_PORT=5432`, pilote `pg`, connexion `pg` dans `config/database.ts`.

### MySQL

Même structure que MariaDB avec `image: mysql:8.0` et les variables `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`. Healthcheck : `['CMD', 'mysqladmin', 'ping', '-h', 'localhost']`.

### Remarques

- Le champ `version:` de l'ancien format Compose est obsolète : ne pas l'écrire.
- Nom de base : `<projet>-db` par défaut (à valider avec l'utilisateur). Un tiret est accepté par MariaDB comme nom de base créé via `MARIADB_DATABASE`.
- Si le port 3306/5432 est déjà pris sur la machine, changer le mapping (`'3307:3306'`) **et** `DB_PORT` dans `.env`.
- Ne pas committer de vrais secrets : le `docker-compose.yml` de développement ne contient que des mots de passe jetables.

## 2. Image de l'application (production)

Testé sur le kit Vue : build OK, démarrage OK, `GET /` et `GET /login` répondent 200 avec la base MariaDB ci-dessus.

```dockerfile
FROM node:24-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN node ace build

FROM node:24-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/build ./
RUN npm ci --omit=dev
EXPOSE 3333
CMD ["node", "bin/server.js"]
```

- `node ace build` produit un dossier `build/` autonome (avec son `package.json` et le code compilé, assets Vite inclus) : on y installe seulement les dépendances de production.
- **`npm ci` peut échouer** (`package.json and package-lock.json are not in sync ... Missing: playwright, vue-eslint-parser`) si le lockfile a été généré avec `legacy-peer-deps=true` (réglage présent dans certains `~/.npmrc`) : le conteneur n'a pas ce réglage et résout les peer dependencies (ici celles de `@japa/browser-client`). Deux solutions :
  - ajouter `--legacy-peer-deps` aux deux `npm ci` du Dockerfile (solution testée) ;
  - ou régénérer le lockfile sans ce réglage (`npm install --no-legacy-peer-deps`) puis committer.
- Ajouter un `.dockerignore` : `node_modules`, `build`, `tmp`, `.env`, `.git`, `public/assets`. Le kit n'en fournit pas ; sans lui, `.env` (secrets) est copié dans l'étape de build.
- Ne pas utiliser `node ace serve --watch` dans l'image : c'est un mode développement.
- Variables d'exécution à fournir (`start/env.ts` les valide au démarrage et le processus s'arrête si l'une manque) : `PORT`, `HOST` (**`0.0.0.0`** dans un conteneur, sinon inaccessible depuis l'extérieur), `NODE_ENV`, `LOG_LEVEL`, `APP_KEY`, `SESSION_DRIVER`, et `DB_*` si la base n'est pas SQLite.
- Migrations : les lancer comme étape distincte (`node ace migration:run --force` dans `build/`, ou un job de déploiement), pas dans le `CMD`.
- Avec SQLite en conteneur, monter un volume sur `tmp/` ; sinon les données disparaissent à chaque redémarrage.

## 3. Application + base dans un même compose

```yaml
services:
  app:
    build: .
    restart: unless-stopped
    ports:
      - '3333:3333'
    environment:
      PORT: 3333
      HOST: 0.0.0.0
      NODE_ENV: production
      LOG_LEVEL: info
      APP_KEY: ${APP_KEY}
      SESSION_DRIVER: cookie
      DB_HOST: db
      DB_PORT: 3306
      DB_USER: adonis
      DB_PASSWORD: secret
      DB_DATABASE: <projet>-db
    depends_on:
      db:
        condition: service_healthy

  db:
    image: mariadb:11
    # ... voir section 1
```

- `DB_HOST: db` (nom du service), pas `127.0.0.1`.
- `depends_on` avec `condition: service_healthy` exige le `healthcheck` de la section 1 (un simple `depends_on: [db]` ne garantit pas que la base accepte les connexions).
- `APP_KEY` : générer avec `node ace generate:key` ; ne pas le mettre en clair dans le compose, utiliser un `.env` non versionné ou un secret.
- Cet assemblage complet (app + db dans le même compose) **n'a pas été exécuté tel quel** ; seuls l'image seule et le compose de la base ont été testés (l'image a été lancée avec `docker run` contre la base du compose).

## 4. Vérification

```bash
docker compose up -d --wait
docker compose ps                         # db (healthy)
docker build -t <projet> .
docker run --rm -p 3333:3333 --env-file .env.docker <projet>
curl -s -o /dev/null -w '%{http_code}\n' localhost:3333/    # 200
```

Voir aussi `database-migrations.md` pour la configuration de `config/database.ts` et de `start/env.ts`.
