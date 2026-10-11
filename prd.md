# ShopMind --- Product Requirements Document (PRD)

**Version:** 1.0\
**Status:** Initial scope\
**Product type:** AI-powered e-commerce platform with an autonomous
shopping agent

## 1. Product Overview

ShopMind helps users find and evaluate products from a natural-language
shopping goal. Its AI agent extracts requirements, researches products,
compares options, explains recommendations, and can prepare a purchase
workflow. Any order placement must require explicit user approval.

**Example:** "Find me a laptop under ₹80,000 for Java development, with
at least 16GB RAM, 512GB SSD, and good battery life."

## 2. Problem & Vision

Online shopping often requires users to search across many products,
manually compare specifications, read reviews, verify policies, and
track prices or orders.

**Vision:** Turn a shopping goal into a transparent, personalized, and
safely controlled shopping journey.

## 3. Target Users

-   Shoppers who want products matched to a budget and specific
    requirements.
-   Buyers who need help comparing specifications, reviews, warranty,
    and price.
-   Users who want to automate repetitive research and price/order
    follow-up tasks.

## 4. Goals and Non-Goals

### Goals

-   Reduce manual product research and comparison.
-   Provide explainable recommendations that respect mandatory
    constraints.
-   Demonstrate real Agentic AI: planning, tool calling, observing
    results, and replanning.
-   Support a complete journey from product discovery to approved order
    creation.
-   Build a secure, testable, observable, production-style system.

### Initial non-goals

-   A real-money payment gateway.
-   Mobile or voice applications.
-   Multi-region infrastructure or Kubernetes in the first release.
-   Unrestricted autonomous purchasing.
-   Dependence on live retailer integrations before the core workflow
    works; seeded/demo catalog data may be used initially and must be
    labelled clearly.

## 5. Core Product Scope

1.  **E-commerce core:** Registration/login, product catalog, product
    details, search and filters, cart, checkout workflow, order history,
    and order tracking.
2.  **AI shopping agent:** Natural-language goal input, structured
    requirement extraction, task planning, authorized tool calling,
    execution status, observation, bounded retries, and replanning.
3.  **Product intelligence:** Specification and stock checks, price
    comparison, review summaries where data exists, hard-constraint
    filtering, ranking, and trade-off explanations.
4.  **RAG knowledge base:** Answers grounded in available product
    manuals, warranty/return policies, shipping information, and FAQs
    using embeddings and vector retrieval.
5.  **Memory and personalization:** Session/task state and user-approved
    preferences; users must be able to inspect or manage saved
    preferences.
6.  **Human approval and safety:** Explicit purchase confirmation,
    authorization checks, price/stock revalidation, and duplicate-order
    prevention.
7.  **Automation and notifications:** Order-status notifications and
    configurable price alerts.
8.  **Production engineering:** Tests, API documentation, structured
    logs, monitoring, error handling, containerized local setup, and CI.

## 6. Primary User Journey

1.  User signs in and submits a shopping goal.
2.  Agent extracts budget, mandatory requirements, and soft preferences;
    it asks a clarifying question when an essential detail is missing.
3.  Agent creates a bounded plan and uses approved tools to search and
    inspect products.
4.  The system verifies hard constraints and ranks eligible products
    using deterministic business rules.
5.  Agent presents a comparison, evidence, limitations, and a
    recommendation.
6.  User selects a product and reviews the exact cart, price, and order
    details.
7.  After explicit approval, the backend revalidates stock, price, and
    authorization before creating the order.
8.  User can view order status and manage supported alerts.

## 7. Functional Requirements

-   **FR-01 --- Authentication:** Secure registration, login, and
    protected user-specific resources.
-   **FR-02 --- Catalog & Search:** Browse, filter, and inspect product
    specifications and availability.
-   **FR-03 --- Goal Understanding:** Convert natural-language requests
    into validated structured requirements.
-   **FR-04 --- Agent Workflow:** Plan tasks, call authorized tools,
    record task status, observe results, and replan when needed.
-   **FR-05 --- Comparison & Recommendation:** Enforce hard constraints,
    rank suitable options, and explain trade-offs without inventing
    facts.
-   **FR-06 --- RAG:** Answer product/policy questions from retrieved
    sources and indicate when evidence is unavailable.
-   **FR-07 --- Memory:** Persist task state and permitted preferences
    with user-level isolation.
-   **FR-08 --- Cart & Orders:** Manage cart contents and create orders
    only through validated, authorized workflows.
-   **FR-09 --- Approval:** Require explicit confirmation before
    purchase; recheck price and inventory immediately before order
    creation.
-   **FR-10 --- Tracking & Alerts:** Display order status and support
    target-price alerts.
-   **FR-11 --- Observability:** Show agent progress, tool activity
    summaries, failures, and final results without exposing private
    internal reasoning.
-   **FR-12 --- Evaluation:** Maintain repeatable tests for requirement
    extraction, tool use, constraint violations, task completion, and
    failure recovery.

## 8. Non-Functional Requirements

-   **Security:** Spring Security/JWT, authorization at service/tool
    boundaries, input validation, secrets kept out of source control,
    and protection against prompt-injection-driven tool misuse.
-   **Data integrity:** Validate ownership, price, stock, and order
    state on the backend; use idempotency for order creation.
-   **Reliability:** Apply bounded timeouts/retries and appropriate
    failure handling; never retry unsafe actions blindly.
-   **Performance:** Set and measure API and agent-workflow latency
    targets; keep AI workflows separate from ordinary API latency
    measurements.
-   **Maintainability:** Clear service boundaries, documented APIs,
    migrations, configuration by environment, and automated tests.
-   **Observability:** Structured logs, correlation IDs, health checks,
    metrics, and traceable agent tasks.
-   **Privacy:** Store only necessary user data and isolate each user's
    conversations, preferences, cart, and orders.

## 9. High-Level Architecture & Technology

-   **Frontend:** Next.js, TypeScript, Tailwind CSS.
-   **Backend:** Java 21, Spring Boot, Spring Security, Spring Data JPA.
-   **Agentic AI:** LangChain4j with a configurable model provider
    (OpenAI or Ollama).
-   **Data:** PostgreSQL; pgvector for semantic retrieval; Redis for
    cache/session or short-lived task state.
-   **Messaging:** Apache Kafka for asynchronous domain events and
    notifications.
-   **Reliability:** Resilience4j, timeouts, bounded retries,
    idempotency, and controlled workflow compensation where needed.
-   **Testing:** JUnit 5, Mockito, API/integration tests, and
    Testcontainers.
-   **Delivery/operations:** Docker Compose, GitHub Actions, Actuator;
    add advanced monitoring and Kubernetes only when justified.

Logical service boundaries: API Gateway, User, Product, Search, Cart,
Order, Agent, and Notification/Automation. Build incrementally; do not
require every service to be implemented simultaneously. The agent must
access business data and actions through validated service/tool APIs,
never direct unrestricted database access.

## 10. Data and Integrations

Core data entities include User, UserPreference, Product, Category,
Inventory, PriceRecord, ProductReview, Cart, CartItem, Order, OrderItem,
AgentTask, AgentStep, AgentApproval, KnowledgeDocument, DocumentChunk,
Conversation/Memory, PriceAlert, and Notification.

Initial catalog/review/policy data may be seeded for demonstration.
External retailer, payment, and shipping integrations must be documented
as unavailable until implemented and tested.

## 11. Initial UI Screens

-   Login/Register
-   Home and shopping-goal input
-   Product listing/search and product details
-   Agent task progress
-   Product comparison/recommendations
-   Cart and purchase approval
-   Order history/tracking
-   Profile/preferences and price alerts

## 12. Acceptance Criteria

The initial release is acceptable when:

1.  A user can sign in and submit a shopping goal.
2.  The agent extracts requirements into a validated structure and
    handles missing essential information.
3.  The agent uses tools to search the catalog and returns products with
    verifiable attributes.
4.  Products that violate mandatory constraints are excluded or clearly
    identified as unavailable alternatives.
5.  The recommendation includes a comparison and does not fabricate
    missing price, stock, or review data.
6.  Purchase cannot be executed without valid explicit approval.
7.  Price, inventory, ownership, and order validity are rechecked
    server-side before order creation.
8.  Repeated submission/retry of the same approved order request does
    not create duplicate orders.
9.  User data is isolated, errors are handled safely, and core flows
    have automated tests.
10. The project runs from documented setup instructions and includes API
    documentation and a demo workflow.

## 13. Success Metrics

Measure requirement-extraction accuracy, hard-constraint violation rate,
task completion rate, tool-call success, recommendation relevance,
workflow latency, failure recovery, token/cost usage, and
duplicate-order prevention. Establish baselines using a repeatable set
of 50--100 shopping goals; do not claim quality targets until measured.

## 14. Delivery Phases

1.  **Foundation:** Repository, documentation, data model, security,
    catalog, search, cart, and order basics.
2.  **Single-agent workflow:** Requirement extraction, planning, tool
    registry, execution state, comparison, and approval.
3.  **Knowledge & memory:** RAG, vector retrieval, session state, and
    permitted user preferences.
4.  **Reliability & automation:** Idempotency, events, notifications,
    price alerts, and recovery.
5.  **Quality & release:** Automated tests, agent evaluation, security
    checks, UI polish, Docker setup, CI, documentation, and demo.

Multi-agent orchestration, real payments, Kubernetes, voice, mobile, and
advanced analytics are later enhancements---not prerequisites for the
first working release.
