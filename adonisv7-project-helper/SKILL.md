---
name: adonisjs-v7-skill
description: 'Assistant pour créer et configurer une application AdonisJS v7, choisir le frontend, gérer la base de données, Docker, les variables d environnement et les références de fonctionnalités. Use when asked to scaffold an AdonisJS v7 project, configure database or realtime frontend, build feature workflows, or add documentation references.'
---

# Skill AdonisJS v7

Ce skill guide la création d une application AdonisJS v7. Il aide à choisir le nom du projet, le kit frontend, la base de données, la configuration Docker et les variables d environnement.

## When to Use This Skill

- L utilisateur souhaite démarrer un projet AdonisJS v7
- Il doit choisir entre Hypermedia, Vue, React ou API
- Il veut une base de données autre que SQLite
- Il demande une configuration Docker ou un exemple de `.env`

## Prerequisites

- Node.js installé
- npm disponible
- Connaissance du choix de kit AdonisJS
- Optionnel : Docker si l utilisateur veut des services containerisés

## Step-by-Step Workflows

### 1. Créer le projet AdonisJS

Utiliser la commande officielle :

```bash
npm create adonisjs@latest <nom-du-projet> -- --kit=<kit>
```

Kits possibles :

- `hypermedia`
- `vue`
- `react`
- `api`

### 2. Choisir l interface graphique

- `hypermedia` pour une application AdonisJS full-stack avec rendu serveur
- `vue` pour une application avec Vue.js côté frontend
- `react` pour une application avec React
- `api` si aucune interface graphique n est requise

### 2.1. Référence SSE pour le frontend

Pour un frontend qui veut des mises à jour temps réel, utiliser AdonisJS Transmit avec Server-Sent Events (SSE) est une bonne option. Transmit prend en charge l abonnement sécurisé, la diffusion côté serveur et l écoute côté client.

Demander à l utilisateur s il souhaite activer SSE côté frontend.

Pour un guide complet, consulter `references/sse-guide.md`.
Pour créer une fonctionnalité, voir `references/feature-guide.md`.

### 3. Configurer la base de données

Si l utilisateur choisit une base non SQLite, proposer la configuration et le pilote correspondant :

- PostgreSQL : package `pg`
- MySQL / MariaDB : package `mysql2`

Par défaut, le nom de la base de données doit être `{{projectName}}-db`.

- Demander à l utilisateur si ce nom lui convient.
- S il souhaite un autre nom, utiliser sa valeur personnalisée.

### 4. Générer des exemples Docker

Proposer un `docker-compose.yml` si une base de données locale est souhaitée.

#### Exemple `docker-compose.yml` pour PostgreSQL

```yaml
version: '3.8'
services:
  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=adonis
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB={{projectName}}-db
    ports:
      - '5432:5432'
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

#### Exemple `docker-compose.yml` pour MySQL/MariaDB

```yaml
version: '3.8'
services:
  db:
    image: mysql:8.0
    restart: unless-stopped
    environment:
      - MYSQL_ROOT_PASSWORD=secret
      - MYSQL_DATABASE={{projectName}}-db
      - MYSQL_USER=adonis
      - MYSQL_PASSWORD=secret
    ports:
      - '3306:3306'
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

### 5. Exemple de `Dockerfile` pour AdonisJS

```dockerfile
FROM node:24-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm install
COPY . ./
RUN npm run build

FROM node:24-alpine AS runtime
WORKDIR /app
COPY --from=builder /app .
RUN npm prune --production

EXPOSE 3333
CMD ["node", "ace", "serve", "--watch"]
```

### 6. Exemple `docker-compose.yml` avec application et base de données

```yaml
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    ports:
      - '3333:3333'
    environment:
      - PORT=3333
      - NODE_ENV=development
      - DB_CONNECTION=pg # ou mysql
      - DB_HOST=db
      - DB_PORT=5432 # ou 3306
      - DB_USER=adonis
      - DB_PASSWORD=secret
      - DB_DATABASE={{projectName}}-db
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=adonis
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB={{projectName}}-db
    ports:
      - '5432:5432'
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

## Variables d environnement

Indiquer les variables à définir dans le fichier `.env` :

- `DB_CONNECTION`
- `DB_HOST`
- `DB_PORT`
- `DB_USER`
- `DB_PASSWORD`
- `DB_DATABASE`

## Conseils pratiques

- Proposer `SQLite` pour les prototypes simples
- Suggérer PostgreSQL ou MySQL pour les environnements de production
- Adapter la configuration Docker aux besoins de l application

## References

- AdonisJS official documentation: <https://docs.adonisjs.com>
- Référence SSE/Transmit pour frontend : `references/sse-guide.md`
- Guide de création de fonctionnalité : `references/feature-guide.md`
- Guide mise à niveau v6 → v7 : `references/adonis-v6-to-v7-upgrade-agent.md`
