# Base de données : migrations, schéma généré, seeders

## 1. Choix et bascule du moteur

Le kit v7 utilise **SQLite** par défaut (`better-sqlite3`, fichier `tmp/db.sqlite3`, aucune variable `DB_*`). Pour PostgreSQL ou MySQL/MariaDB :

1. Installer le pilote : `npm i pg` ou `npm i mysql2` (MariaDB utilise `mysql2`). Si SQLite n'est plus utilisé : `npm uninstall better-sqlite3` et retirer `"better-sqlite3": true` de `allowScripts` dans `package.json`.
2. `config/database.ts` : ajouter `import env from '#start/env'`, décommenter le bloc `pg` ou `mysql`, passer `connection` à `'pg'`/`'mysql'`, supprimer le bloc `sqlite`.
3. `start/env.ts` : déclarer les variables (règle générale : section 8 ci-dessous) :

```ts
DB_HOST: Env.schema.string({ format: 'host' }),
DB_PORT: Env.schema.number(),
DB_USER: Env.schema.string(),
DB_PASSWORD: Env.schema.string.optional(),
DB_DATABASE: Env.schema.string(),
```

4. `.env` et `.env.example` :

```env
DB_HOST=127.0.0.1
DB_PORT=3306        # 5432 pour PostgreSQL
DB_USER=adonis
DB_PASSWORD=secret  # vide dans .env.example
DB_DATABASE=<projet>-db
```

5. Démarrer la base (voir `docker.md`), puis `node ace migration:run`.

> Le bloc `schemaGeneration` de `config/database.ts` (règles dans `database/schema_rules.js`) pilote la génération de `database/schema.ts`. Il est dans chaque connexion du kit : le conserver sur la connexion choisie.

## 2. Migrations

```bash
node ace make:migration todos            # crée database/migrations/<timestamp>_create_todos_table.ts
node ace migration:run                   # applique ET régénère database/schema.ts
node ace migration:status
node ace migration:rollback              # développement uniquement
```

Exemple (table `todos`) :

```ts
import { BaseSchema } from '@adonisjs/lucid/schema'

export default class extends BaseSchema {
  protected tableName = 'todos'

  async up() {
    this.schema.createTable(this.tableName, (table) => {
      table.increments('id').notNullable()
      table.integer('user_id').unsigned().notNullable().references('users.id').onDelete('CASCADE')
      table.string('title').notNullable()
      table.text('description').nullable()
      table.enum('status', ['pending', 'in_progress', 'done']).notNullable().defaultTo('pending')
      table.timestamp('created_at').notNullable()
      table.timestamp('updated_at').nullable()
    })
  }

  async down() {
    this.schema.dropTable(this.tableName)
  }
}
```

- **Conventions** : colonnes en `snake_case` (`user_id`, `created_at`), propriétés de modèle en `camelCase` (`userId`, `createdAt`). Respecter cette règle, sinon le schéma généré et les requêtes divergent.
- `increments('id')` + `integer(...).unsigned()` pour les clés étrangères (types compatibles).
- `down()` doit défaire exactement `up()`.
- Une migration appliquée ne se modifie pas : créer une nouvelle migration (`alterTable`).

## 3. Schéma auto-généré (`database/schema.ts`)

`migration:run` scanne la base et écrit `database/schema.ts` : une classe par table (`UserSchema`, `TodoSchema`...) étendant `BaseModel`, avec les `declare` de chaque colonne. **Ne jamais l'éditer à la main.**

Conséquences :

- Un modèle ne contient que **relations, getters, mixins** :

```ts
import { belongsTo } from '@adonisjs/lucid/orm'
import type { BelongsTo } from '@adonisjs/lucid/types/relations'
import { TodoSchema } from '#database/schema'
import User from '#models/user'

export default class Todo extends TodoSchema {
  @belongsTo(() => User)
  declare user: BelongsTo<typeof User>
}
```

- **Ordre obligatoire** : migration → `migration:run` → modèle. Créer le modèle avant la migration provoque une erreur d'import `#database/schema`.
- Après `alterTable`, relancer `migration:run` pour régénérer les types.

## 4. Exemple d'évolution : catégories (8 catégories système)

Structure :

```
categories (id, name unique, color, created_at)
todos (..., category_id nullable -> categories.id ON DELETE SET NULL)
```

Migration 1 : `node ace make:migration create_categories_table`

```ts
table.increments('id').primary()
table.string('name').unique().notNullable()
table.string('color').notNullable()
table.timestamp('created_at').notNullable()
```

Migration 2 : `node ace make:migration add_category_to_todos`

```ts
async up() {
  this.schema.alterTable('todos', (table) => {
    table.integer('category_id').unsigned().nullable().after('user_id')
      .references('categories.id').onDelete('SET NULL')
  })
}
async down() {
  this.schema.alterTable('todos', (table) => {
    table.dropForeign(['category_id'])
    table.dropColumn('category_id')
  })
}
```

> Le tutoriel nomme la colonne `categoryId` (camelCase) : c'est incohérent avec `user_id`. Utiliser `category_id`. `.after('user_id')` n'existe que sur MySQL/MariaDB ; le retirer pour PostgreSQL et SQLite.

Modèles :

```ts
// app/models/category.ts
import { hasMany } from '@adonisjs/lucid/orm'
import type { HasMany } from '@adonisjs/lucid/types/relations'
import { CategorySchema } from '#database/schema'
import Todo from '#models/todo'

export default class Category extends CategorySchema {
  @hasMany(() => Todo)
  declare todos: HasMany<typeof Todo>
}

// app/models/todo.ts : ajouter
@belongsTo(() => Category)
declare category: BelongsTo<typeof Category>
```

## 5. Seeders

```bash
node ace make:seeder Categories     # database/seeders/categories_seeder.ts
node ace db:seed
```

```ts
import { BaseSeeder } from '@adonisjs/lucid/seeders'
import Category from '#models/category'

export default class extends BaseSeeder {
  async run() {
    await Category.updateOrCreateMany('name', [
      { name: 'Work', color: '#EF4444' },
      { name: 'Personal', color: '#3B82F6' },
      { name: 'Shopping', color: '#10B981' },
      { name: 'Health', color: '#F59E0B' },
      { name: 'Learning', color: '#8B5CF6' },
      { name: 'Finance', color: '#EC4899' },
      { name: 'Home', color: '#6366F1' },
      { name: 'Travel', color: '#14B8A6' },
    ])
  }
}
```

- `updateOrCreateMany('name', [...])` rend le seeder **idempotent** (le tutoriel utilise `createMany`, qui échoue au second lancement à cause de l'unicité de `name`).
- Seeders = données de référence (catégories, rôles, config). Éviter d'y mettre de vraies données utilisateur.
- Ordre d'exécution : fichiers triés par nom ; pour imposer un ordre, préfixer (`01_...`).

## 6. Contrôleur et validateurs après ajout de la catégorie

- `Todo.query().where('userId', user.id).preload('category')` ; passer aussi `categories` aux props.
- `categoryId: vine.number().min(1).nullable().optional()` dans `create` et `update`.
- Sérialiser avec un transformer unique (voir `tuyau-transformers.md`) pour `index` **et** pour les `transmit.broadcast()` : même forme partout (`id`, `title`, `description`, `status`, `createdAt`, `category`).

## 7. Vérification

```bash
node ace migration:status
node ace db:seed
node ace list:routes
npm run typecheck
```

Contrôler en base (`docker compose exec db mariadb -uadonis -psecret <projet>-db -e 'show tables'`) que `adonis_schema`, `users`, `todos`, `categories` existent.

## 8. Variables d'environnement : règle unique

Valable pour **toute** variable lue par `env.get()` (base de données, limiteur, cache, mail...) :

1. la **déclarer dans `start/env.ts`** avec son type (`Env.schema.string()`, `.number()`, `.enum([...])`, `.optional()`) ; sans cela `env.get('X')` ne compile pas ;
2. la **renseigner** dans `.env`, `.env.example` (sans secret) **et** `.env.test` si les tests en dépendent ;
3. au démarrage, la validation de `start/env.ts` **arrête le processus** si une variable obligatoire manque ou est invalide : c'est voulu, y compris en production et dans une commande Ace.

Le kit n'utilise pas `DB_CONNECTION` : la connexion par défaut est fixée dans `config/database.ts`.
