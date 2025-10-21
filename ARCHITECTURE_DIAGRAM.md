# Kutt to Turso Architecture Diagram

## Current Architecture

```mermaid
graph TB
    subgraph "Current Multi-DB Setup"
        A[Kutt Application] --> B[Knex.js ORM]
        B --> C{Database Client}
        C --> D[SQLite<br/>better-sqlite3]
        C --> E[PostgreSQL<br/>pg]
        C --> F[MySQL<br/>mysql2]
    end
    
    subgraph "Docker Compose"
        G[App Container] --> H[Volume: db_data_sqlite]
        G --> I[Volume: custom]
    end
    
    subgraph "Deployment Options"
        J[docker-compose.yml<br/>SQLite]
        K[docker-compose.postgres.yml<br/>PostgreSQL + Redis]
        L[docker-compose.mariadb.yml<br/>MariaDB + Redis]
    end
```

## Target Architecture with Turso

```mermaid
graph TB
    subgraph "New Turso-Based Setup"
        A[Kutt Application] --> B[Knex.js ORM]
        B --> C[@libsql/client]
        C --> D[Turso Database<br/>SQLite-compatible]
    end
    
    subgraph "Docker Compose"
        E[App Container] --> F[Volume: custom]
        G[Turso Container] --> H[Volume: turso_data]
        E --> I[Turso Network Connection]
    end
    
    subgraph "Simplified Deployment"
        J[docker-compose.yml<br/>Turso (Default)]
        K[docker-compose.turso.yml<br/>Turso + Custom Config]
    end
```

## Migration Flow

```mermaid
graph LR
    subgraph "Phase 1: Preparation"
        A[Current Setup] --> B[Backup Data]
        B --> C[Install Dependencies]
    end
    
    subgraph "Phase 2: Configuration"
        C --> D[Update Environment]
        D --> E[Modify Database Logic]
        E --> F[Update Docker Config]
    end
    
    subgraph "Phase 3: Migration"
        F --> G[Deploy Turso]
        G --> H[Migrate Data]
        H --> I[Validate Migration]
    end
    
    subgraph "Phase 4: Cleanup"
        I --> J[Remove Old DB Logic]
        J --> K[Update Documentation]
        K --> L[Final Testing]
    end
```

## Container Architecture

```mermaid
graph TB
    subgraph "Docker Network: kutt-network"
        subgraph "kutt-server"
            A1[Node.js App]
            A2[Knex.js]
            A3[@libsql/client]
            A4[Express.js]
            A1 --> A2
            A2 --> A3
            A1 --> A4
        end
        
        subgraph "kutt-turso"
            B1[Turso Server]
            B2[SQLite Engine]
            B3[Data Files]
            B1 --> B2
            B2 --> B3
        end
        
        subgraph "Volumes"
            C1[turso_data<br/>Persistent Storage]
            C2[custom<br/>Custom Files]
        end
    end
    
    A3 -->|libsql://turso-db:8080| B1
    B3 --> C1
    A4 --> C2
    
    D[External User] -->|Port 3000| A4
```

## Data Flow Comparison

### Current Multi-DB Data Flow
```mermaid
sequenceDiagram
    participant U as User
    participant A as Kutt App
    participant K as Knex.js
    participant DB as Database (SQLite/PG/MySQL)
    
    U->>A: HTTP Request
    A->>K: Database Query
    K->>DB: SQL Query
    DB->>K: Query Result
    K->>A: Processed Data
    A->>U: HTTP Response
```

### New Turso Data Flow
```mermaid
sequenceDiagram
    participant U as User
    participant A as Kutt App
    participant K as Knex.js
    participant L as @libsql/client
    participant T as Turso DB
    
    U->>A: HTTP Request
    A->>K: Database Query
    K->>L: libSQL Query
    L->>T: SQLite Query
    T->>L: Query Result
    L->>K: Processed Data
    K->>A: Data
    A->>U: HTTP Response
```

## Environment Variables Mapping

### Current Environment
```mermaid
graph LR
    A[DB_CLIENT] --> B[better-sqlite3<br/>pg<br/>mysql2]
    C[DB_FILENAME] --> D[db/data]
    E[DB_HOST] --> F[localhost]
    G[DB_PORT] --> H[5432/3306]
    I[DB_NAME] --> J[kutt]
    K[DB_USER] --> L[postgres/mysql]
    M[DB_PASSWORD] --> N[password]
```

### New Turso Environment
```mermaid
graph LR
    A[DB_CLIENT] --> B[turso]
    C[TURSO_URL] --> D[libsql://turso-db:8080]
    E[TURSO_AUTH_TOKEN] --> F[auth-token]
    G[TURSO_SYNC_URL] --> H[optional-sync-url]
```

## Migration Benefits

```mermaid
mindmap
  root((Turso Migration))
    Performance
      SQLite Speed
      Edge Optimization
      Lower Latency
    Simplicity
      Single Database
      Less Configuration
      Easier Debugging
    Cost
      Lower Resources
      No DB Server License
      Reduced Complexity
    Modern
      Active Development
      Community Support
      Regular Updates
    Scalability
      Edge Ready
      Replication Support
      Global Distribution
```

## Deployment Comparison

### Current Deployment Complexity
```mermaid
graph TD
    A[Choose Database] --> B{SQLite?}
    B -->|Yes| C[Simple File Setup]
    B -->|No| D[Install PostgreSQL/MySQL]
    D --> E[Configure Connection]
    E --> F[Set Up Redis?]
    F --> G[Configure Docker Compose]
    G --> H[Deploy]
```

### New Turso Deployment
```mermaid
graph TD
    A[Run docker-compose up] --> B[Turso Container Starts]
    B --> C[App Container Starts]
    C --> D[Connects to Turso]
    D --> E[Ready to Use]