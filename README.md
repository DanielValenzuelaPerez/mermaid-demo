```mermaid
flowchart LR
    S[Start]
    A[Enter Email Address]
    B[Enter Name]
    C{Existing User?}
    D{Accept Conditions}

    S --> A
    A --> C
    C -->|no| B
    B --> D
    D -->|no| A

```