# Entity-Relationship Diagram

```mermaid
erDiagram
    Customer ||--o{ Order : places
    Order ||--|{ LineItem : contains
```