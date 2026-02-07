<!-- docs/database/erd.md -->
# データベースER図
```mermaid
erDiagram
    users ||--o{ portfolios : "has"
    users {
        uuid id PK
        varchar email UK
        timestamptz created_at
    }
