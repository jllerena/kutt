# Kutt to Turso Migration Plan

## Overview

This document outlines the refactoring plan to migrate Kutt from its current multi-database support (SQLite, PostgreSQL, MySQL) to use Turso as the exclusive database solution while maintaining the `docker compose up -d` deployment experience.

## Architecture

### Current State
- Multi-database support via Knex.js
- SQLite (better-sqlite3) as default
- PostgreSQL and MySQL as options
- Database-specific logic scattered throughout codebase

### Target State
- Turso as the sole database solution
- libSQL client for Node.js integration
- Simplified configuration and deployment
- Improved performance with SQLite-compatible edge database

## Technical Implementation

### 1. Dependencies Update

**File: `package.json`**
```json
{
  "dependencies": {
    "@libsql/client": "^0.6.0",
    "knex": "^3.1.0",
    // Remove: better-sqlite3, pg, mysql2
  }
}
```

### 2. Environment Variables

**New Environment Variables:**
- `TURSO_URL`: Turso database connection URL
- `TURSO_AUTH_TOKEN`: Authentication token for Turso
- `TURSO_SYNC_URL`: Optional sync URL for replication

**Updated `.example.env`:**
```env
# Turso Database Configuration
TURSO_URL=libsql://turso-db:8080
TURSO_AUTH_TOKEN=your-auth-token
TURSO_SYNC_URL=libsql://sync-url:8080

# Legacy DB variables (deprecated, kept for backward compatibility)
# DB_CLIENT=better-sqlite3
# DB_FILENAME=db/data
```

### 3. Database Connection Logic

**File: `server/knex.js`**
```javascript
const knex = require("knex");
const { createClient } = require("@libsql/client");
const env = require("./env");

// Turso-specific configuration
const isTurso = env.DB_CLIENT === "turso";

const db = knex({
  client: isTurso ? "@libsql/client" : env.DB_CLIENT,
  connection: isTurso ? {
    url: env.TURSO_URL,
    authToken: env.TURSO_AUTH_TOKEN,
    syncUrl: env.TURSO_SYNC_URL,
  } : {
    // Existing connection logic for backward compatibility
    ...(env.DB_CLIENT === "better-sqlite3" && { filename: env.DB_FILENAME }),
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

// Update database type detection
db.isTurso = isTurso;
db.isSQLite = env.DB_CLIENT === "sqlite3" || env.DB_CLIENT === "better-sqlite3";
db.isPostgres = env.DB_CLIENT === "pg" || env.DB_CLIENT === "pg-native";
db.isMySQL = env.DB_CLIENT === "mysql" || env.DB_CLIENT === "mysql2";

// Update ILIKE compatibility
db.compatibleILIKE = (db.isPostgres || db.isTurso) ? "andWhereILike" : "andWhereLike";

module.exports = db;
```

### 4. Environment Validation

**File: `server/env.js`**
```javascript
const supportedDBClients = [
  "turso",
  "pg",
  "pg-native", 
  "sqlite3",
  "better-sqlite3",
  "mysql",
  "mysql2"
];

const spec = {
  // Existing variables...
  DB_CLIENT: str({ choices: supportedDBClients, default: "turso" }),
  
  // Turso-specific variables
  TURSO_URL: str({ default: "libsql://turso-db:8080" }),
  TURSO_AUTH_TOKEN: str({ default: "" }),
  TURSO_SYNC_URL: str({ default: "" }),
  
  // Legacy variables (deprecated)
  DB_FILENAME: str({ default: "db/data" }),
  DB_HOST: str({ default: "localhost" }),
  // ... other legacy DB variables
};
```

### 5. Docker Compose Configuration

**New File: `docker-compose.turso.yml`**
```yaml
services:
  turso-db:
    image: ghcr.io/tursodatabase/turso-server:latest
    container_name: kutt-turso
    environment:
      - TURSO_DATABASE_NAME=kutt
      - TURSO_DATABASE_PATH=/var/lib/turso/kutt.db
    volumes:
      - turso_data:/var/lib/turso
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "turso", "db", "status"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

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
    ports:
      - "3000:3000"
    depends_on:
      turso-db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  turso_data:
  custom:
```

**Updated `docker-compose.yml` (default to Turso):**
```yaml
services:
  turso-db:
    image: ghcr.io/tursodatabase/turso-server:latest
    container_name: kutt-turso
    environment:
      - TURSO_DATABASE_NAME=kutt
      - TURSO_DATABASE_PATH=/var/lib/turso/kutt.db
    volumes:
      - turso_data:/var/lib/turso
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "turso", "db", "status"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

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
    ports:
      - "3000:3000"
    depends_on:
      turso-db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  turso_data:
  custom:
```

### 6. Dockerfile Updates

**File: `Dockerfile`**
```dockerfile
# specify node.js image
FROM node:22-alpine

# use production node environment by default
ENV NODE_ENV=production

# set working directory.
WORKDIR /kutt

# download dependencies while using Docker's caching
RUN --mount=type=bind,source=package.json,target=package.json \
    --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

# copy the rest of source files into the image
COPY . .

# expose the port that the app listens on
EXPOSE 3000

# intialize database and run the app
CMD npm run migrate && npm start
```

### 7. Migration Compatibility

All existing migrations should work with Turso since it's SQLite-compatible. However, some migrations may need updates:

**File: `server/migrations/20241223062111_indexes.js`**
```javascript
// Update for Turso compatibility
const isMySQL = env.DB_CLIENT === "mysql" || env.DB_CLIENT === "mysql2";
const isTurso = env.DB_CLIENT === "turso";

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
```

## Deployment Instructions

### Quick Start (Turso)
```bash
# Clone the repository
git clone https://github.com/thedevs-network/kutt.git
cd kutt

# Set environment variables
echo "JWT_SECRET=your-secure-jwt-secret" > .env
echo "DEFAULT_DOMAIN=yourdomain.com" >> .env

# Start with Turso
docker compose up -d
```

### Migration from Existing Database
```bash
# Export existing data
npm run db:export

# Update configuration to use Turso
echo "DB_CLIENT=turso" >> .env
echo "TURSO_URL=libsql://turso-db:8080" >> .env

# Start new Turso setup
docker compose -f docker-compose.turso.yml up -d

# Import data
npm run db:import
```

## Benefits of Turso Migration

1. **Simplified Architecture**: Single database solution reduces complexity
2. **Better Performance**: SQLite-based with edge optimization
3. **Easier Deployment**: No need for database server setup
4. **Cost Effective**: Lower resource requirements
5. **Modern Stack**: Active development and community support
6. **Edge Ready**: Built for distributed deployments

## Backward Compatibility

The migration plan maintains backward compatibility by:
- Keeping existing environment variables
- Supporting legacy database clients during transition
- Providing migration scripts for existing data
- Maintaining the same API interface

## Testing Strategy

1. **Unit Tests**: Test database connection and query logic
2. **Integration Tests**: Test migrations with Turso
3. **E2E Tests**: Test full application functionality
4. **Performance Tests**: Compare performance with current setup
5. **Migration Tests**: Test data migration from existing databases

## Rollback Plan

If issues arise with Turso:
1. Switch back to SQLite using `DB_CLIENT=better-sqlite3`
2. Restore from database backups
3. Update Docker Compose configuration
4. Redeploy with previous configuration

## Timeline

1. **Phase 1** (Week 1): Dependencies and environment setup
2. **Phase 2** (Week 1): Database connection logic
3. **Phase 3** (Week 2): Docker configuration
4. **Phase 4** (Week 2): Testing and optimization
5. **Phase 5** (Week 3): Documentation and deployment

## Next Steps

1. Review and approve this plan
2. Set up development environment with Turso
3. Begin implementation starting with dependencies
4. Test each component before proceeding to next
5. Update documentation throughout the process