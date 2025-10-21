# Kutt to Turso Implementation Guide

This guide provides step-by-step instructions for implementing the Turso migration with specific code changes.

## Step 1: Update Dependencies

### File: `package.json`

**Changes:**
```json
{
  "dependencies": {
    "@libsql/client": "^0.6.0",
    "bcryptjs": "2.4.3",
    "bull": "4.16.5",
    "cookie-parser": "1.4.7",
    "cors": "2.8.5",
    "date-fns": "2.30.0",
    "dotenv": "16.4.7",
    "envalid": "8.0.0",
    "express": "4.21.2",
    "express-rate-limit": "7.5.0",
    "express-validator": "6.14.2",
    "geoip-lite": "1.4.10",
    "hbs": "4.2.0",
    "helmet": "7.1.0",
    "ioredis": "5.4.2",
    "isbot": "5.1.19",
    "jsonwebtoken": "9.0.2",
    "knex": "3.1.0",
    "ms": "2.1.3",
    "nanoid": "3.3.8",
    "nodemailer": "6.9.16",
    "passport": "0.7.0",
    "passport-jwt": "4.0.1",
    "passport-local": "1.0.0",
    "passport-localapikey-update": "0.6.0",
    "useragent": "2.3.0"
  }
}
```

**Removed dependencies:**
- `better-sqlite3`
- `pg`
- `mysql2`

## Step 2: Update Environment Configuration

### File: `.example.env`

**Add new Turso variables:**
```env
# Turso Database Configuration (New)
DB_CLIENT=turso
TURSO_URL=libsql://turso-db:8080
TURSO_AUTH_TOKEN=
TURSO_SYNC_URL=

# Legacy Database Variables (Deprecated - kept for backward compatibility)
# DB_CLIENT=better-sqlite3
# DB_FILENAME=db/data
# DB_HOST=localhost
# DB_PORT=5432
# DB_NAME=kutt
# DB_USER=postgres
# DB_PASSWORD=
# DB_SSL=false
# DB_POOL_MIN=0
# DB_POOL_MAX=10
```

## Step 3: Update Environment Validation

### File: `server/env.js`

**Complete replacement:**
```javascript
require("dotenv").config();
const { cleanEnv, num, str, bool } = require("envalid");
const { readFileSync } = require("node:fs");

const supportedDBClients = [
  "turso",
  "pg",
  "pg-native",
  "sqlite3",
  "better-sqlite3",
  "mysql",
  "mysql2"
];

// make sure custom alphabet is not empty
if (process.env.LINK_CUSTOM_ALPHABET === "") {
  delete process.env.LINK_CUSTOM_ALPHABET;
}

// make sure jwt secret is not empty
if (process.env.JWT_SECRET === "") {
  delete process.env.JWT_SECRET;
}

// if is started with the --production argument, then set NODE_ENV to production
if (process.argv.includes("--production")) {
  process.env.NODE_ENV = "production";
}

const spec = {
  PORT: num({ default: 3000 }),
  SITE_NAME: str({ example: "Kutt", default: "Kutt" }),
  DEFAULT_DOMAIN: str({ example: "kutt.it", default: "localhost:3000" }),
  LINK_LENGTH: num({ default: 6 }),
  LINK_CUSTOM_ALPHABET: str({ default: "abcdefghkmnpqrstuvwxyzABCDEFGHKLMNPQRSTUVWXYZ23456789" }),
  TRUST_PROXY: bool({ default: true }),
  
  // Database Configuration
  DB_CLIENT: str({ choices: supportedDBClients, default: "turso" }),
  
  // Turso-specific variables
  TURSO_URL: str({ default: "libsql://turso-db:8080" }),
  TURSO_AUTH_TOKEN: str({ default: "" }),
  TURSO_SYNC_URL: str({ default: "" }),
  
  // Legacy database variables (deprecated)
  DB_FILENAME: str({ default: "db/data" }),
  DB_HOST: str({ default: "localhost" }),
  DB_PORT: num({ default: 5432 }),
  DB_NAME: str({ default: "kutt" }),
  DB_USER: str({ default: "postgres" }),
  DB_PASSWORD: str({ default: "" }),
  DB_SSL: bool({ default: false }),
  DB_POOL_MIN: num({ default: 0 }),
  DB_POOL_MAX: num({ default: 10 }),
  
  // Redis Configuration
  REDIS_ENABLED: bool({ default: false }),
  REDIS_HOST: str({ default: "127.0.0.1" }),
  REDIS_PORT: num({ default: 6379 }),
  REDIS_PASSWORD: str({ default: "" }),
  REDIS_DB: num({ default: 0 }),
  
  // Application Configuration
  DISALLOW_ANONYMOUS_LINKS: bool({ default: true }),
  DISALLOW_REGISTRATION: bool({ default: true }),
  SERVER_IP_ADDRESS: str({ default: "" }),
  SERVER_CNAME_ADDRESS: str({ default: "" }),
  CUSTOM_DOMAIN_USE_HTTPS: bool({ default: false }),
  JWT_SECRET: str({ devDefault: "securekey" }),
  
  // Mail Configuration
  MAIL_ENABLED: bool({ default: false }),
  MAIL_HOST: str({ default: "" }),
  MAIL_PORT: num({ default: 587 }),
  MAIL_SECURE: bool({ default: false }),
  MAIL_USER: str({ default: "" }),
  MAIL_FROM: str({ default: "", example: "Kutt <support@kutt.it>" }),
  MAIL_PASSWORD: str({ default: "" }),
  
  // Other Configuration
  ENABLE_RATE_LIMIT: bool({ default: false }),
  REPORT_EMAIL: str({ default: "" }),
  CONTACT_EMAIL: str({ default: "" }),
  NODE_APP_INSTANCE: num({ default: 0 }),
};

for (const key in spec) {
  const file_key = key + "_FILE";
  if (!(file_key in process.env)) continue;
  try {
    process.env[key] = readFileSync(process.env[file_key], "utf8").trim();
  } catch {
    // on error, env_FILE just doesn't get applied.
  }
}

const env = cleanEnv(process.env, spec);

module.exports = env;
```

## Step 4: Update Database Connection Logic

### File: `server/knex.js`

**Complete replacement:**
```javascript
const knex = require("knex");
const env = require("./env");

const isTurso = env.DB_CLIENT === "turso";
const isSQLite = env.DB_CLIENT === "sqlite3" || env.DB_CLIENT === "better-sqlite3";
const isPostgres = env.DB_CLIENT === "pg" || env.DB_CLIENT === "pg-native";
const isMySQL = env.DB_CLIENT === "mysql" || env.DB_CLIENT === "mysql2";

const db = knex({
  client: isTurso ? "@libsql/client" : env.DB_CLIENT,
  connection: isTurso ? {
    url: env.TURSO_URL,
    authToken: env.TURSO_AUTH_TOKEN,
    syncUrl: env.TURSO_SYNC_URL,
  } : {
    ...(isSQLite && { filename: env.DB_FILENAME }),
    host: env.DB_HOST,
    port: env.DB_PORT,
    database: env.DB_NAME,
    user: env.DB_USER,
    password: env.DB_PASSWORD,
    ssl: env.DB_SSL,
    pool: {
      min: env.DB_POOL_MIN,
      max: env.DB_POOL_MAX
    }
  },
  useNullAsDefault: true,
});

db.isTurso = isTurso;
db.isPostgres = isPostgres;
db.isSQLite = isSQLite;
db.isMySQL = isMySQL;

db.compatibleILIKE = (isPostgres || isTurso) ? "andWhereILike" : "andWhereLike";

module.exports = db;
```

## Step 5: Update Knex Configuration

### File: `knexfile.js`

**Complete replacement:**
```javascript
// this configuration is for migrations only
// and since jwt secret is not required, it's set to a placeholder string to bypass env validation
if (!process.env.JWT_SECRET) {
  process.env.JWT_SECRET = "securekey";
}

const env = require("./server/env");

const isTurso = env.DB_CLIENT === "turso";
const isSQLite = env.DB_CLIENT === "sqlite3" || env.DB_CLIENT === "better-sqlite3";

module.exports = {
  client: isTurso ? "@libsql/client" : env.DB_CLIENT,
  connection: isTurso ? {
    url: env.TURSO_URL,
    authToken: env.TURSO_AUTH_TOKEN,
    syncUrl: env.TURSO_SYNC_URL,
  } : {
    ...(isSQLite && { filename: env.DB_FILENAME }),
    host: env.DB_HOST,
    database: env.DB_NAME,
    user: env.DB_USER,
    port: env.DB_PORT,
    password: env.DB_PASSWORD,
    ssl: env.DB_SSL,
  },
  useNullAsDefault: true,
  migrations: {
    tableName: "knex_migrations",
    directory: "server/migrations",
    disableMigrationsListValidation: true,
  }
};
```

## Step 6: Create New Docker Compose Configuration

### File: `docker-compose.yml` (Updated Default)

**Complete replacement:**
```yaml
services:
  turso-db:
    image: ghcr.io/tursodatabase/turso-server:latest
    container_name: kutt-turso
    environment:
      - TURSO_DATABASE_NAME=kutt
      - TURSO_DATABASE_PATH=/var/lib/turso/kutt.db
      - TURSO_LOG_LEVEL=info
    volumes:
      - turso_data:/var/lib/turso
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/v1/status"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    networks:
      - kutt-network

  server:
    build:
      context: .
    volumes:
       - custom:/kutt/custom
    environment:
      DB_CLIENT: turso
      TURSO_URL: libsql://turso-db:8080
      TURSO_AUTH_TOKEN: ""
      JWT_SECRET: ${JWT_SECRET:-securekey}
      DEFAULT_DOMAIN: ${DEFAULT_DOMAIN:-localhost:3000}
      SITE_NAME: ${SITE_NAME:-Kutt}
      DISALLOW_REGISTRATION: ${DISALLOW_REGISTRATION:-true}
      DISALLOW_ANONYMOUS_LINKS: ${DISALLOW_ANONYMOUS_LINKS:-true}
      TRUST_PROXY: ${TRUST_PROXY:-true}
    ports:
      - "3000:3000"
    depends_on:
      turso-db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - kutt-network

volumes:
  turso_data:
  custom:

networks:
  kutt-network:
    driver: bridge
```

### File: `docker-compose.turso.yml` (Alternative Configuration)

**New file:**
```yaml
services:
  turso-db:
    image: ghcr.io/tursodatabase/turso-server:latest
    container_name: kutt-turso
    environment:
      - TURSO_DATABASE_NAME=kutt
      - TURSO_DATABASE_PATH=/var/lib/turso/kutt.db
      - TURSO_LOG_LEVEL=debug
      - TURSO_HTTP_PORT=8080
    volumes:
      - turso_data:/var/lib/turso
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/v1/status"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s
    restart: unless-stopped
    networks:
      - kutt-network

  redis:
    image: redis:7-alpine
    container_name: kutt-redis
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    networks:
      - kutt-network

  server:
    build:
      context: .
    volumes:
       - custom:/kutt/custom
    environment:
      DB_CLIENT: turso
      TURSO_URL: libsql://turso-db:8080
      TURSO_AUTH_TOKEN: ""
      REDIS_ENABLED: true
      REDIS_HOST: redis
      REDIS_PORT: 6379
      JWT_SECRET: ${JWT_SECRET:-securekey}
      DEFAULT_DOMAIN: ${DEFAULT_DOMAIN:-localhost:3000}
      SITE_NAME: ${SITE_NAME:-Kutt}
      DISALLOW_REGISTRATION: ${DISALLOW_REGISTRATION:-false}
      DISALLOW_ANONYMOUS_LINKS: ${DISALLOW_ANONYMOUS_LINKS:-false}
      ENABLE_RATE_LIMIT: true
      TRUST_PROXY: ${TRUST_PROXY:-true}
    ports:
      - "3000:3000"
    depends_on:
      turso-db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - kutt-network

volumes:
  turso_data:
  redis_data:
  custom:

networks:
  kutt-network:
    driver: bridge
```

## Step 7: Update Migrations for Turso Compatibility

### File: `server/migrations/20241223062111_indexes.js`

**Update for Turso compatibility:**
```javascript
const env = require("../env");

const isMySQL = env.DB_CLIENT === "mysql" || env.DB_CLIENT === "mysql2";
const isTurso = env.DB_CLIENT === "turso";

/**
 * @param { import("knex").Knex } knex
 * @returns { Promise<void> }
 */
async function up(knex) {
  // make apikey unique
  await knex.schema.alterTable("users", function(table) {
    table.unique("apikey");
  });

  // IF NOT EXISTS is not available on MySQL but available on Turso/SQLite
  const ifNotExists = isMySQL ? "" : "IF NOT EXISTS";

  // create them separately because one string with break lines didn't work on MySQL
  await Promise.all([
    knex.raw(`CREATE INDEX ${ifNotExists} links_domain_id_index ON links (domain_id);`),
    knex.raw(`CREATE INDEX ${ifNotExists} links_user_id_index ON links (user_id);`),
    knex.raw(`CREATE INDEX ${ifNotExists} links_address_index ON links (address);`),
    knex.raw(`CREATE INDEX ${ifNotExists} links_expire_in_index ON links (expire_in);`),
    knex.raw(`CREATE INDEX ${ifNotExists} domains_address_index ON domains (address);`),
    knex.raw(`CREATE INDEX ${ifNotExists} domains_user_id_index ON domains (user_id);`),
    knex.raw(`CREATE INDEX ${ifNotExists} hosts_address_index ON hosts (address);`),
    knex.raw(`CREATE INDEX ${ifNotExists} visits_link_id_index ON visits (link_id);`),
  ]);
};

/**
 * @param { import("knex").Knex } knex
 * @returns { Promise<void> }
 */
async function down(knex) {
  await knex.schema.alterTable("users", function(table) {
    table.dropUnique(["apikey"]);
  });

  await Promise.all([
    knex.raw(`DROP INDEX links_domain_id_index;`),
    knex.raw(`DROP INDEX links_user_id_index;`),
    knex.raw(`DROP INDEX links_address_index;`),
    knex.raw(`DROP INDEX links_expire_in_index;`),
    knex.raw(`DROP INDEX domains_address_index;`),
    knex.raw(`DROP INDEX domains_user_id_index;`),
    knex.raw(`DROP INDEX hosts_address_index;`),
    knex.raw(`DROP INDEX visits_link_id_index;`),
  ]);
};

module.exports = {
  up,
  down,
}
```

## Step 8: Create Migration Scripts

### File: `scripts/migrate-to-turso.js`

**New file:**
```javascript
#!/usr/bin/env node

const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

async function migrateToTurso() {
  console.log('🚀 Starting migration to Turso...');
  
  // 1. Backup current database
  console.log('📦 Backing up current database...');
  if (fs.existsSync('db/data')) {
    fs.copyFileSync('db/data', `db/data.backup.${Date.now()}`);
    console.log('✅ Database backed up');
  }
  
  // 2. Update package.json
  console.log('📋 Updating dependencies...');
  const packageJson = JSON.parse(fs.readFileSync('package.json', 'utf8'));
  
  // Remove old database clients
  delete packageJson.dependencies['better-sqlite3'];
  delete packageJson.dependencies['pg'];
  delete packageJson.dependencies['mysql2'];
  
  // Add libSQL client
  packageJson.dependencies['@libsql/client'] = '^0.6.0';
  
  fs.writeFileSync('package.json', JSON.stringify(packageJson, null, 2));
  console.log('✅ Dependencies updated');
  
  // 3. Install new dependencies
  console.log('📦 Installing new dependencies...');
  try {
    execSync('npm install', { stdio: 'inherit' });
    console.log('✅ Dependencies installed');
  } catch (error) {
    console.error('❌ Failed to install dependencies:', error.message);
    process.exit(1);
  }
  
  // 4. Create .env file for Turso
  if (!fs.existsSync('.env')) {
    console.log('📝 Creating .env file...');
    const envContent = `DB_CLIENT=turso
TURSO_URL=libsql://turso-db:8080
TURSO_AUTH_TOKEN=
JWT_SECRET=your-secure-jwt-secret-change-this
DEFAULT_DOMAIN=localhost:3000
`;
    fs.writeFileSync('.env', envContent);
    console.log('✅ .env file created');
  }
  
  console.log('🎉 Migration to Turso completed!');
  console.log('');
  console.log('Next steps:');
  console.log('1. Review and update the .env file with your settings');
  console.log('2. Run: docker compose up -d');
  console.log('3. Run: docker compose exec server npm run migrate');
  console.log('4. Test the application');
}

if (require.main === module) {
  migrateToTurso().catch(console.error);
}

module.exports = migrateToTurso;
```

## Step 9: Update Documentation

### File: `README.md`

**Add Turso section:**
```markdown
## Docker

Make sure Docker is installed, then you can start the app from the root directory:

### Default Setup (Turso)

```sh
docker compose up -d
```

This will start Kutt with Turso as the database.

### Alternative Setups

Various docker-compose configurations are available. Use `docker compose -f <file_name> up` to start the one you want:

- [`docker-compose.yml`](./docker-compose.yml): **Default Kutt setup with Turso database.**
- [`docker-compose.turso.yml`](./docker-compose.turso.yml): Kutt with Turso and Redis.
  - Optional environment variables: `REDIS_ENABLED`, `JWT_SECRET`, `DEFAULT_DOMAIN`

### Legacy Database Setups (Deprecated)

- [`docker-compose.sqlite-redis.yml`](./docker-compose.sqlite-redis.yml): Starts Kutt with SQLite and Redis.
- [`docker-compose.postgres.yml`](./docker-compose.postgres.yml): Starts Kutt with Postgres and Redis.
- [`docker-compose.mariadb.yml`](./docker-compose.mariadb.yml): Starts Kutt with MariaDB and Redis.

## Turso Database

Kutt now uses Turso as the default database. Turso is a SQLite-compatible, edge-optimized database that provides:

- **Better Performance**: SQLite-based with edge optimization
- **Simplified Setup**: No need for separate database server installation
- **Lower Resource Usage**: Efficient containerized deployment
- **Modern Architecture**: Built for distributed applications

### Turso Configuration

The app can be configured via environment variables for Turso:

| Variable | Description | Default | Example |
| -------- | ----------- | ------- | ------- |
| `DB_CLIENT` | Database client type | `turso` | `turso` |
| `TURSO_URL` | Turso database connection URL | `libsql://turso-db:8080` | `libsql://turso-db:8080` |
| `TURSO_AUTH_TOKEN` | Authentication token for Turso | `` | `your-auth-token` |
| `TURSO_SYNC_URL` | Optional sync URL for replication | `` | `libsql://sync-url:8080` |

### Migration from Other Databases

If you're migrating from SQLite, PostgreSQL, or MySQL:

1. Backup your existing data
2. Update your `.env` file to use Turso settings
3. Run the migration script: `node scripts/migrate-to-turso.js`
4. Start with Docker: `docker compose up -d`
```

## Step 10: Testing

### File: `test/turso-integration.test.js`

**New file:**
```javascript
const request = require('supertest');
const app = require('../server/server');

describe('Turso Integration Tests', () => {
  beforeAll(async () => {
    // Ensure Turso is running and migrations are applied
    const db = require('../server/knex');
    await db.migrate.latest();
  });

  afterAll(async () => {
    // Clean up
    const db = require('../server/knex');
    await db.destroy();
  });

  test('Database connection with Turso', async () => {
    const db = require('../server/knex');
    expect(db.isTurso).toBe(true);
    
    // Test basic query
    const result = await db.raw('SELECT 1 as test');
    expect(result[0].test).toBe(1);
  });

  test('Application starts with Turso', async () => {
    const response = await request(app)
      .get('/')
      .expect(200);
    
    expect(response.text).toContain('Kutt');
  });

  test('Database migrations work with Turso', async () => {
    const db = require('../server/knex');
    
    // Check if tables exist
    const tables = await db('sqlite_master')
      .where('type', 'table')
      .select('name');
    
    const tableNames = tables.map(t => t.name);
    expect(tableNames).toContain('users');
    expect(tableNames).toContain('links');
    expect(tableNames).toContain('domains');
  });
});
```

## Deployment Commands

### Quick Start Commands
```bash
# Clone and setup
git clone https://github.com/thedevs-network/kutt.git
cd kutt

# Run migration script
node scripts/migrate-to-turso.js

# Set your environment
echo "JWT_SECRET=your-secure-jwt-secret" >> .env
echo "DEFAULT_DOMAIN=yourdomain.com" >> .env

# Start with Turso
docker compose up -d

# Run migrations
docker compose exec server npm run migrate

# Check logs
docker compose logs -f
```

### Development Commands
```bash
# Start in development mode
npm run dev

# Run tests
npm test

# Create new migration
npm run migrate:make add_new_feature

# Rollback migration
npm run migrate:rollback
```

This implementation guide provides all the necessary code changes and scripts to successfully migrate Kutt from its current multi-database setup to use Turso exclusively while maintaining the simple `docker compose up -d` deployment experience.