# ShopMind --- Technical Discovery

**Version:** 1.0 \| **Status:** Initial technical decisions and
validation plan\
**Related document:** `PRD.md`

## 1. Purpose

The PRD defines what ShopMind should do. This document explains the
proposed technical approach, technology choices, architecture, risks,
proof-of-concept (POC) experiments, and implementation sequence. It is a
living document: record actual experiment results and update decisions
when evidence changes.

## 2. Product Goal

ShopMind accepts a natural-language shopping goal, extracts
requirements, researches products, compares eligible options, explains
recommendations, and supports a purchase only after explicit user
approval.

**Example:** "Find a laptop under ₹80,000 for Java development with at
least 16GB RAM, 512GB SSD, and good battery life."

The LLM helps interpret requests and select from permitted tools.
Backend code remains authoritative for constraints, price, stock,
permissions, and order creation.

## 3. Discovery Principles

-   Prefer technologies that fit the Java/Spring ecosystem and can be
    tested locally.
-   Validate high-risk assumptions with small POCs before building the
    full system.
-   Build a reliable single-agent journey before multi-agent
    orchestration.
-   Keep authorization, inventory, price, and order rules in backend
    services---not in prompts.
-   Use clearly labelled seeded/demo data until real retailer
    integrations are implemented.
-   Add distributed infrastructure only when it solves a defined need.

## 4. Proposed Technology Stack

  -----------------------------------------------------------------------
  Area                    Proposed choice         Purpose
  ----------------------- ----------------------- -----------------------
  Language/backend        Java 21, Spring Boot    Business logic and APIs

  Gateway/security        Spring Cloud Gateway,   Routing, authentication
                          Spring Security, JWT    and authorization

  Persistence             Spring Data JPA /       Relational persistence
                          Hibernate               

  Database                PostgreSQL              Users, catalog, cart,
                                                  orders and agent tasks

  Vector search           pgvector                Semantic retrieval for
                                                  RAG

  Agent framework         LangChain4j             Model integration,
                                                  structured output and
                                                  tool calling

  Model provider          OpenAI API or Ollama    Compare quality,
                                                  latency, cost and
                                                  hardware needs

  Short-lived state       Redis                   Cache and temporary
                                                  task/session state

  Events                  Apache Kafka            Asynchronous domain
                                                  events and
                                                  notifications

  Frontend                Next.js, TypeScript,    Shopping and agent UI
                          Tailwind CSS            

  Resilience              Resilience4j            Timeouts, bounded
                                                  retries, circuit
                                                  breakers

  Testing                 JUnit 5, Mockito,       Unit and integration
                          Spring test tools,      testing
                          Testcontainers          

  Local setup / CI        Docker Compose, GitHub  Repeatable setup and
                          Actions                 automated checks

  API docs / operations   OpenAPI, Actuator;      API contracts, health
                          later OpenTelemetry,    and observability
                          Prometheus, Grafana     
  -----------------------------------------------------------------------

These are baseline proposals. POCs should validate critical choices.
Kubernetes, real-money payments and multi-region deployment are not
first-release prerequisites.

## 5. High-Level Architecture

``` mermaid
flowchart TB
    U[User] --> FE[Next.js Frontend]
    FE --> GW[API Gateway]
    GW --> US[User Service]
    GW --> PS[Product Service]
    GW --> SS[Search Service]
    GW --> CS[Cart Service]
    GW --> OS[Order Service]
    GW --> AS[Agent Service]
    GW --> NS[Notification / Automation]
    AS --> LLM[LLM Provider]
    AS --> TR[Controlled Tool Registry]
    TR --> PS
    TR --> SS
    TR --> CS
    TR --> OS
    US --> PG[(PostgreSQL)]
    PS --> PG
    CS --> PG
    OS --> PG
    AS --> PG
    AS --> VDB[(pgvector)]
    AS --> R[(Redis)]
    OS -. events .-> K[Kafka]
    PS -. price / inventory events .-> K
    K --> NS
```

**Architecture notes** - These are logical boundaries, not a requirement
to deploy every service independently from day one. - Start
incrementally. Keep service contracts and data ownership clear. - The
Agent Service uses validated tools/APIs; it must never have unrestricted
database access. - PostgreSQL is the durable source of truth for
business records. Redis is not the only copy of orders or approvals. -
Use REST for request/response operations and Kafka for asynchronous
events where useful.

## 6. Service Responsibilities

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  API Gateway                         Routing and edge-level controls

  User Service                        Accounts, roles and preferences

  Product Service                     Product details, specifications,
                                      prices and inventory

  Search Service                      Keyword/filter search and optional
                                      semantic search

  Cart Service                        Cart contents, ownership and
                                      validation

  Order Service                       Validated order creation, order
                                      status and idempotency

  Agent Service                       Goal extraction, planning, tool
                                      orchestration, recommendations and
                                      approvals

  Notification/Automation             Order notifications and scheduled
                                      price alerts
  -----------------------------------------------------------------------

## 7. Agent Execution Workflow

``` mermaid
flowchart TD
    A[Shopping goal] --> B[Extract structured requirements]
    B --> C{Essential information missing?}
    C -- Yes --> D[Ask user to clarify]
    D --> B
    C -- No --> E[Validate requirements]
    E --> F[Create bounded plan]
    F --> G[Select permitted tool]
    G --> H[Authorize and validate input]
    H --> I[Execute backend operation]
    I --> J[Observe structured result]
    J --> K{Goal satisfied?}
    K -- No --> L{Step / cost limit reached?}
    L -- No --> F
    L -- Yes --> M[Return partial result or safe failure]
    K -- Yes --> N[Apply hard filters and ranking]
    N --> O[Show comparison and recommendation]
    O --> P[User reviews exact purchase details]
    P --> Q{Explicit approval?}
    Q -- No --> R[Stop or revise cart]
    Q -- Yes --> S[Revalidate price, stock and permissions]
    S --> T[Idempotent order creation]
    T --> U[Show confirmation and status]
```

### Agent states

`CREATED`, `PLANNING`, `EXECUTING`, `WAITING_FOR_APPROVAL`, `COMPLETED`,
`FAILED`, `CANCELLED`, `NEEDS_INPUT`.

Persist task state and meaningful step/tool metadata. The UI should show
progress, tool activity summaries, evidence, and outcomes---not private
internal chain-of-thought.

### Candidate tools

`searchProducts`, `getProductDetails`, `getProductReviews`,
`checkInventory`, `getPrice`, `getPriceHistory`, `compareProducts`,
`getUserPreferences`, `getCart`, `addToCart`, `createOrder`,
`getOrderStatus`, `cancelOrder`.

Expose only implemented tools. Validate input schemas, ownership,
permissions and results. Sensitive operations require explicit policy
checks and approval. Never let the model execute arbitrary SQL or
unrestricted actions.

## 8. Sequence Diagram: Research to Approved Purchase

``` mermaid
sequenceDiagram
    actor User
    participant UI as Frontend
    participant Agent as Agent Service
    participant LLM as LLM Provider
    participant Tools as Tool Registry
    participant Catalog as Product/Search Service
    participant Cart as Cart Service
    participant Order as Order Service

    User->>UI: Submit shopping goal
    UI->>Agent: Create task
    Agent->>LLM: Extract requirements
    LLM-->>Agent: Structured constraints
    Agent->>Tools: Search products
    Tools->>Catalog: Validated search
    Catalog-->>Tools: Product candidates
    Tools-->>Agent: Structured results
    Agent->>Tools: Fetch details and availability
    Tools->>Catalog: Validate product facts
    Catalog-->>Tools: Specs, price and stock
    Tools-->>Agent: Structured evidence
    Agent->>Agent: Filter and rank deterministically
    Agent-->>UI: Comparison and recommendation
    User->>UI: Approve exact purchase details
    UI->>Agent: Approval bound to cart and total
    Agent->>Cart: Validate ownership and cart
    Cart-->>Agent: Validated cart summary
    Agent->>Order: Create order with idempotency key
    Order->>Catalog: Revalidate price and stock
    Catalog-->>Order: Validation result
    Order-->>Agent: Confirmation or safe rejection
    Agent-->>UI: Order result and status
    UI-->>User: Display result
```

The sequence is a target model; final endpoints and service-call
ownership are confirmed during API design. Approval must bind to exact
items, quantities and total. Any material change requires renewed
approval.

## 9. Data, RAG and Memory

### Data placement

-   **PostgreSQL:** users, preferences, products, prices, inventory,
    carts, orders, tasks, approvals, alerts and notifications.
-   **pgvector:** document embeddings and metadata for semantic
    retrieval.
-   **Redis:** cache and short-lived task/session state.
-   **Kafka:** events such as `OrderCreated`, `OrderShipped`,
    `OrderDelivered`, `PriceChanged` and `AgentApprovalRequired`.

### RAG pipeline

``` mermaid
flowchart LR
    D[Trusted documents] --> I[Validate and ingest]
    I --> C[Chunk and add metadata]
    C --> E[Create embeddings]
    E --> V[(pgvector)]
    Q[User question] --> QE[Query embedding]
    QE --> V
    V --> R[Relevant chunks]
    R --> L[LLM with retrieved context]
    L --> A[Grounded answer with sources]
```

Use RAG for manuals, warranty/return policies, shipping and FAQs. Store
source/version metadata. If evidence is missing, say so rather than
inventing policy. RAG does not replace live price or stock checks.

### Memory distinction

-   **Task/session memory:** current goal, constraints, completed steps,
    tool outcomes and pending approval.
-   **Long-term preferences:** only useful and permitted preferences,
    such as budget range or product characteristics.
-   Enforce user isolation and provide a way to inspect/delete saved
    preferences.
-   Treat retrieved documents and product content as untrusted data, not
    instructions.

## 10. Recommendation and Model Strategy

Use the LLM for language understanding, bounded tool selection,
summarization and explanations. Use deterministic Java logic for: 1.
Mandatory constraint checks. 2. Eligibility filtering. 3. Product
scoring/ranking. 4. Price, inventory and order authorization.

Ranking may consider price fit, specification match, evidence-backed
battery information, review signals and user preferences. Make weights
explicit and testable. Never treat missing data as a verified fact.

Keep model access behind configuration. Compare candidate models on
structured-output validity, tool-call correctness, constraint adherence,
latency, token cost and local hardware requirements.

## 11. Security and Reliability

-   Enforce authentication/authorization in services and sensitive
    tools.
-   Treat prompts, reviews and retrieved documents as untrusted input.
-   Validate tool arguments and use a strict tool allowlist.
-   Require explicit approval before purchase, cancellation or other
    sensitive actions.
-   Bind approval to user and exact order details; revalidate price and
    stock immediately before purchase.
-   Use idempotency keys and event deduplication where appropriate.
-   Apply timeouts and bounded retries; never blindly retry
    non-idempotent actions.
-   Add circuit breakers and workflow compensation where justified.
-   Keep secrets out of Git; avoid logging tokens, credentials or
    unnecessary personal data.
-   Use correlation IDs, task IDs, health checks and structured logs.

## 12. POCs and Feasibility Tests

  -----------------------------------------------------------------------
  POC                     Experiment              Evidence to record
  ----------------------- ----------------------- -----------------------
  1\. Structured          Convert varied prompts  Field accuracy,
  extraction              into validated Java     invalid-output rate,
                          DTOs                    clarification behaviour

  2\. Tool calling        Invoke one read-only    Correct selection,
                          product search tool     valid arguments and
                          through LangChain4j     error handling

  3\. Constraint engine   Filter/rank a fixed     Constraint violation
                          dataset                 rate and repeatable
                                                  ranking

  4\. RAG                 Retrieve from a policy  Retrieval relevance,
                          document and answer     source grounding,
                          questions               abstention when
                                                  evidence is absent

  5\. Memory              Persist task state and  User isolation,
                          an explicit preference  persistence and
                                                  deletion behaviour

  6\. Purchase safety     Require approval and    No unapproved order; no
                          repeat an order request duplicate for same
                                                  idempotent request

  7\. Local               Run required            Repeatable setup,
  infrastructure          dependencies using      health checks and
                          Docker Compose          integration-test
                                                  results
  -----------------------------------------------------------------------

Record date, configuration/model version, test cases, results,
limitations and decisions. A planned POC is not a completed result until
it has been run.

## 13. Testing and Evaluation

-   **Unit:** constraint filtering, ranking, authorization, state
    transitions and idempotency.
-   **Integration:** REST contracts and persistence.
-   **Testcontainers:** PostgreSQL/Redis/Kafka tests where relevant.
-   **Agent evaluation set:** begin with 50--100 goals covering normal
    prompts, missing fields, impossible budgets, unavailable products,
    conflicting constraints, malicious instructions and tool failures.
-   **Metrics:** requirement extraction accuracy, hard-constraint
    violation rate, tool success, task completion, recommendation
    relevance, latency, cost/token usage and safe failure behaviour.
-   **End-to-end:** login → goal → research → recommendation → approval
    → order → tracking.

Set baselines before claiming improvement. Use repeatable cases and
review failures; agent outputs may vary.

## 14. Risks and Mitigations

  -----------------------------------------------------------------------
  Risk                                Mitigation
  ----------------------------------- -----------------------------------
  Invented product facts              Use authoritative catalog data and
                                      server-side validation

  Poor tool calling                   Model POC, strict schemas, bounded
                                      workflow and fallback

  Prompt injection                    Treat external content as data;
                                      enforce tool allowlists and backend
                                      permissions

  Duplicate/unauthorized orders       Explicit approval, idempotency and
                                      ownership checks

  Stale price or stock                Recheck before order creation

  High latency/cost                   Bound steps/tokens, cache safe
                                      reads, measure cost

  Irrelevant RAG results              Evaluate retrieval, apply metadata
                                      filters and disclose missing
                                      evidence

  Microservice complexity             Build incrementally; split services
                                      when justified

  Missing external integrations       Use clearly labelled demo data
                                      until integration is implemented
                                      and tested
  -----------------------------------------------------------------------

## 15. Implementation Sequence

``` mermaid
flowchart TD
    A[1. Repository and docs] --> B[2. Domain model and API contracts]
    B --> C[3. User security, catalog and search]
    C --> D[4. Cart, orders and tracking]
    D --> E[5. Docker setup and core tests]
    E --> F[6. LangChain4j/model extraction POC]
    F --> G[7. Tool registry and task state]
    G --> H[8. Research and recommendations]
    H --> I[9. Approval and idempotent orders]
    I --> J[10. RAG and memory]
    J --> K[11. Events, notifications and alerts]
    K --> L[12. Evaluation, security and observability]
    L --> M[13. UI polish, CI, docs and demo]
    M --> N[14. Multi-agent/Kubernetes/real integrations if justified]
```

### Phase exit criteria

1.  **Foundation:** authenticated APIs and catalog/cart/order flows work
    with tests.
2.  **Single agent:** extraction, read-only tools, comparison and
    tool-error handling work.
3.  **Safe shopping:** approval, price/stock revalidation and idempotent
    order creation pass tests.
4.  **Knowledge and memory:** RAG cites relevant source material; task
    state and permitted preferences persist safely.
5.  **Release quality:** end-to-end demo, evaluation, security checks,
    setup guide and CI pass.

## 16. Decisions and Open Questions

### Current proposed decisions

-   Java 21 + Spring Boot backend.
-   LangChain4j as the first agent framework to validate.
-   PostgreSQL for relational data; pgvector for semantic retrieval.
-   Redis for short-lived state/cache, not the sole source of truth.
-   Explicit purchase approval and deterministic backend validation.
-   Single-agent workflow before multi-agent orchestration.
-   Seeded product data until real data integrations exist.

### Validate through POCs

-   Which model meets quality, latency, cost and hardware constraints?
-   Which embedding/chunking strategy retrieves useful policy passages?
-   How much planning is necessary beyond a bounded workflow?
-   Which components need separate deployment in the first release?
-   When does Kafka justify its operational complexity?
-   What performance and quality targets are realistic based on
    measurement?

**Discovery is complete enough to guide implementation when:** critical
choices have a rationale, risky assumptions have POC evidence or are
explicitly unresolved, service/data boundaries are understood, and the
first phase has clear exit criteria.

**Maintenance rule:** Update this document when decisions change. Record
actual results; never present planned experiments as completed evidence.
