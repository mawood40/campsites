**Product Requirements Document  
Campsite Reviews Membership Platform**

AI-Implementable Full-Stack Web Application  
React SPA • FastAPI Edge/BFF/Backend • Redis • PostgreSQL • Docker

| **Document Purpose**     | Executable PRD for autonomous implementation of the product without human code changes.                 |
|--------------------------|---------------------------------------------------------------------------------------------------------|
| **Intended Implementer** | AI coding agent operating from repository instructions, tests, and CI/CD workflows.                     |
| **Deployment Model**     | Single-computer local development/test; front-end and back-end deployed to separate servers via SSH.    |
| **Configuration Policy** | No environment-specific code branches. Runtime differences allowed only through .env and \*.json files. |

| **Primary business objective: authenticated members can discover campsites, add new campsites with photos and pricing, and create/view campsite reviews with role-based administration.** |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 1. Product Overview

## 1.1 Vision

Build a membership-based web application where authenticated members can
view campsites reviewed by the community, add new campsites, rate
campsites with comments, and review all feedback for a campsite.

## 1.2 Product Goals

The system must provide a clean member experience, strict role-based
access control, production-grade deployment, and an architecture that an
AI agent can implement, test, package, and deploy without manual
source-code edits between development, test, and production.

## 1.3 Target Users

Subscriber users who browse campsites and submit reviews; admin users
who can do everything subscribers can do and can remove individual
subscribers.

## 1.4 Key Product Capabilities

Email-based signup and login; authenticated React SPA; campsite catalog
with average rating; campsite creation including image upload;
thumbs-up/thumbs-down review workflow with comments; per-campsite review
history; admin management of subscriber accounts.

# 2. Product Scope

## 2.1 In Scope

Account signup and login, session/token-based authentication, role-based
authorization, campsite CRUD limited to create and read for v1, image
upload/storage integration, review creation, campsite rating
aggregation, admin subscriber removal, observability, containerization,
SSH deployment, local single-machine orchestration, and
configuration-only environment switching.

## 2.2 Out of Scope

Social features beyond campsite reviews, messaging, campsite booking,
payment processing for memberships, map-based route planning, mobile
native apps, offline support, advanced moderation workflows, and
analytics dashboards beyond operational metrics.

# 3. User Roles and Permissions

## 3.1 Subscriber

Can sign up, log in, browse campsites, view average campsite rating,
open reviews for a campsite, create a campsite, upload one primary
campsite image, and submit thumbs-up or thumbs-down ratings with
comments.

## 3.2 Admin

Has all subscriber permissions and can remove individual subscriber
accounts. Admin removal must deactivate or delete the target subscriber
according to configured policy, while preserving referential integrity
of campsite and review data.

## 3.3 Authorization Model

Authorization must be enforced at the edge and validated again in
downstream services. The SPA must not rely on hidden UI controls as the
only security barrier.

# 4. User Experience Summary

## 4.1 Entry Experience

Unauthenticated users are presented with Login and Sign Up pages. Email
address is mandatory. Password rules and email verification behavior
must be configurable.

## 4.2 Authenticated Experience

After authentication, the user enters the SPA and sees a list/grid of
campsites. Each campsite card shows name, location, price, general
information, primary image, and average review summary.

## 4.3 Add Campsite Flow

The user selects the '+' action, completes a form with campsite name,
location, pricing, general information, and picture upload, then
submits. On success the new campsite appears in the list.

## 4.4 Review Flow

The user chooses thumbs-up or thumbs-down, leaves comments, and submits
the review. The campsite aggregate rating updates after successful
submission.

## 4.5 Review Inspection Flow

The user selects a campsite Review action to inspect all reviews for
that campsite, including rating polarity, comments, author display name
or configured pseudonym, and timestamps.

## 4.6 Admin Management Flow

Admin users can open subscriber management and remove individual
subscribers. The UI must prevent accidental removal with confirmation.

# 5. Architecture

## 5.1 Required Topology

The system must use a three-level full-stack topology: (1) React SPA
front-end, (2) FastAPI Python BFF behind an API Gateway/edge layer, and
(3) FastAPI Python backend service with PostgreSQL and Redis. All
artifacts are containerized.

## 5.2 Logical Request Path

Browser -\> Edge/API Gateway -\> Python BFF -\> Python Backend Service
-\> PostgreSQL/Redis/Object Storage. The edge handles TLS termination,
routing, rate limiting, and coarse auth enforcement. The BFF shapes data
for the SPA. The backend owns domain rules and persistence.

## 5.3 Front-End Responsibilities

React SPA handles routing, view rendering, form validation,
token/session handling through secure browser patterns, optimistic and
non-optimistic UI updates as defined, and calls only the BFF-facing API
surface.

## 5.4 BFF Responsibilities

The FastAPI BFF provides client-oriented endpoints, request aggregation,
response shaping, DTO/schema translation, edge-compatible auth
propagation, upload brokering if needed, and client-specific error
normalization.

## 5.5 Backend Responsibilities

The backend service owns business logic, role enforcement at service
level, data validation, persistence, aggregate calculations, user
lifecycle rules, and event/log emission.

## 5.6 Redis Responsibilities

Redis is used for cache, rate-limit counters or token/session support
where needed, short-lived job state, and derived read optimization.
Redis must not become the system of record for durable business data.

## 5.7 PostgreSQL Responsibilities

PostgreSQL is the source of truth for users, campsites, reviews, and
administrative actions.

## 5.8 Static and Media Assets

The architecture must support containerized services plus externalized
persistent storage for uploaded campsite images. The storage adapter
must be environment-configurable. Local development may use a local
S3-compatible service or mounted filesystem; deployed environments may
use server-local persistent volumes or object storage.

# 6. Functional Requirements

## 6.1 Authentication

The system must support signup, login, logout, password hashing, secure
session/token issuance, protected route enforcement, and role-aware
authorization.

## 6.2 Signup

A new user must be able to create an account using email and password.
Duplicate email addresses are not allowed. Signup confirmation and email
verification must be configurable.

## 6.3 Login

A registered user must be able to log in using email and password.
Authentication failures must use safe error messages.

## 6.4 Campsite Listing

Authenticated users must be able to list campsites with pagination or
incremental loading. Each record must include the average review state
and review count.

## 6.5 Campsite Details

Authenticated users must be able to open a campsite detail view showing
full information and recent review summary.

## 6.6 Campsite Creation

Authenticated users must be able to create a campsite with required
fields: name, location, pricing, general info, and one image. Validation
rules must be centrally defined and shared through schemas/specs.

## 6.7 Image Upload

The system must support image upload with content-type and size
validation, safe filename handling, and persistence outside the
container filesystem.

## 6.8 Review Submission

Authenticated users must be able to submit exactly one rating payload
per review action containing thumbs-up or thumbs-down and free-form
comments.

## 6.9 Review Retrieval

Authenticated users must be able to view all reviews for a campsite.
Results must be ordered by configured default, such as newest first.

## 6.10 Average Rating

The average review presented for a campsite must be computed from stored
review signals. For binary ratings, the average may be represented as
percentage positive or normalized score. The product must define one
canonical representation and use it consistently in UI and API.

## 6.11 Admin Subscriber Removal

Admin users must be able to remove individual subscribers. Removal must
require confirmation and be auditable.

## 6.12 Auditability

Administrative actions and security-relevant user lifecycle events must
be logged with actor, target, timestamp, and result.

## 6.13 Error Handling

The user interface must present clear validation and failure messages
while backend and BFF return structured error responses.

# 7. Non-Functional Requirements

## 7.1 AI-Only Implementability

The repository must contain sufficient specifications, schemas, test
fixtures, scripts, and developer instructions so that an AI agent can
implement, run, verify, package, and deploy the system with no human
source-code intervention.

## 7.2 No Code Drift Across Environments

The exact same application code and container images must be promotable
across development, test, and production. Differences are allowed only
through .env files, JSON config files, secrets injection, and
infrastructure endpoints.

## 7.3 Deterministic Local Environment

A single developer computer must be able to run the entire system for
development and test using Docker Compose or equivalent local
orchestration.

## 7.4 Deployment Portability

Front-end and back-end artifacts must be independently deployable to
separate servers through scripted SSH-based deployment without
rebuilding different artifacts per environment.

## 7.5 Reliability

Core read and write operations must be idempotent or safely retriable
where appropriate. Failures must not corrupt user, campsite, or review
data.

## 7.6 Security

Passwords must be salted and hashed with a modern algorithm, secrets
must never be committed to source control, uploads must be validated,
RBAC must be enforced server-side, and API boundaries must be
authenticated and authorized.

## 7.7 Performance

The application must provide responsive campsite browsing and review
submission under expected membership load. Caching and pagination must
be used where beneficial.

## 7.8 Observability

Structured logs, health checks, readiness checks, metrics hooks, and
correlation/request IDs must be present across edge, BFF, and backend.

## 7.9 Testability

Unit, integration, contract, and end-to-end tests must run on the same
code paths used in production. Seed data and fixtures must be
environment-configurable, not code-switched.

## 7.10 Maintainability

The codebase must follow strict project structure, schema-driven
validation, migration discipline, and separation of concerns between
SPA, BFF, and backend.

# 8. Architecture Detail

## 8.1 Component Inventory

| **Component**    | **Technology**             | **Primary Responsibility**                                                           | **State**             | **Deployment Unit**       |
|------------------|----------------------------|--------------------------------------------------------------------------------------|-----------------------|---------------------------|
| SPA              | React                      | Authentication views, campsite views, forms, API consumption                         | Stateless             | Container                 |
| Edge/API Gateway | Gateway/Reverse Proxy      | TLS termination, routing, rate limiting, auth boundary, static asset delivery option | Stateless             | Server service/container  |
| BFF              | FastAPI                    | Client-oriented API, aggregation, response shaping                                   | Mostly stateless      | Container                 |
| Backend          | FastAPI                    | Domain logic, persistence orchestration, RBAC, aggregates                            | Stateful via DB/Redis | Container                 |
| Redis            | Redis                      | Cache, counters, ephemeral coordination                                              | Stateful              | Container/service         |
| PostgreSQL       | PostgreSQL                 | Source of truth                                                                      | Stateful              | Container/service         |
| Media Storage    | Filesystem or object store | Persist campsite images                                                              | Stateful              | Persistent volume/service |

## 8.2 Sequence Overview

Signup/login requests flow through the edge into the BFF, which
coordinates with the backend authentication domain and supporting
services. Campsite listing and review reads may be cache-assisted, but
writes must commit to PostgreSQL before success is returned. Review
aggregates must be updated transactionally or through controlled
asynchronous recomputation with consistency guarantees.

## 8.3 Trust Boundaries

The browser is untrusted. The edge is the internet-facing trust
boundary. The BFF is trusted for client shaping but not as the sole
enforcer of business authorization. The backend is the business
authority. Database and Redis are private network services only.

# 9. Domain Model

## 9.1 Core Entities

| **Entity**  | **Required Fields**                                                                     | **Notes**                                | **Persistence** |
|-------------|-----------------------------------------------------------------------------------------|------------------------------------------|-----------------|
| User        | id, email, password_hash, role, status, created_at                                      | Role ∈ {admin, subscriber}; email unique | PostgreSQL      |
| Campsite    | id, name, location, pricing_text/value, general_info, image_url, created_by, created_at | One primary image required in v1         | PostgreSQL      |
| Review      | id, campsite_id, user_id, rating_value, comments, created_at                            | Binary rating: thumbs_up or thumbs_down  | PostgreSQL      |
| AdminAction | id, actor_user_id, target_user_id, action_type, reason, created_at, result              | Supports subscriber removal audit        | PostgreSQL      |

## 9.2 Derived Data

Each campsite must expose review_count and average_rating_summary as
derived fields. The backend remains responsible for canonical
computation; the BFF may reshape representation for the SPA.

# 10. API Requirements

## 10.1 API Style

Use versioned JSON APIs. Schemas must be explicitly defined. Error
payloads must be structured and machine-readable.

## 10.2 SPA to BFF Contract

The SPA must only call BFF endpoints. The BFF contract is stable,
documented, and decoupled from backend internal schemas.

## 10.3 BFF to Backend Contract

The BFF/backend contract must use internal API schemas or generated
clients and must support contract testing.

## 10.4 Auth Propagation

The edge and BFF must forward only the minimum trusted identity context
required. The backend must validate claims or tokens according to the
selected trust model.

## 10.5 Upload Contract

Image upload endpoints must define multipart or pre-signed upload flow
explicitly and document maximum sizes, accepted formats, and failure
behavior.

# 11. Project Structure

## The repository must be organized so an AI agent can discover the correct implementation boundaries, test commands, and deployment commands without inference from tribal knowledge.

repo-root/  
apps/  
frontend-spa/  
src/  
public/  
tests/  
package.json  
bff-service/  
app/  
tests/  
pyproject.toml  
backend-service/  
app/  
tests/  
pyproject.toml  
packages/  
api-contracts/  
shared-schemas/  
shared-config/  
infra/  
docker/  
compose/  
ssh-deploy/  
gateway/  
db/  
config/  
base/  
app.json  
auth.json  
storage.json  
environments/  
dev/  
test/  
prod/  
scripts/  
local/  
ci/  
deploy/  
docs/  
prd/  
architecture/  
runbooks/  
ai-implementation-guides/  
migrations/  
.env.example  
README.md

## 11.1 Structure Rules

Front-end, BFF, and backend must remain separate deployable units.

Shared schemas and contracts must live in dedicated packages, not copied
across services.

Infrastructure scripts must be checked into source control and
executable by CI/CD and autonomous agents.

Config JSON files define feature flags, policy switches, storage
adapters, and environment-specific endpoint values.

# 12. Configuration and Environment Management

## 12.1 Immutable Application Code

No source-code edits are allowed when moving from development to test to
production. Promotion uses the same built artifact.

## 12.2 Allowed Runtime Inputs

Only .env files, JSON config files, injected secrets, and infrastructure
bindings may differ across environments.

## 12.3 Configuration Precedence

The application must define and document a deterministic precedence
order, such as defaults -\> JSON base config -\> environment JSON
override -\> .env -\> secrets.

## 12.4 Secrets Handling

Secrets must never be stored in committed JSON files. Secret values are
injected through .env or deployment secrets stores.

## 12.5 Validation

Startup must validate configuration and fail fast on missing or invalid
settings.

# 13. Local Development and Test Topology

## 13.1 Single-Machine Operation

A single development computer must run the SPA, edge-compatible routing
layer if needed, BFF, backend, Redis, PostgreSQL, and media storage
dependency using Docker-based orchestration.

## 13.2 Developer Workflow

The local workflow must support bootstrapping, seed data load, migration
execution, test execution, linting, type checks, end-to-end tests, and
image build commands from scripted entry points.

## 13.3 VE/Test Parity

Validation, evaluation, and test environments must exercise the same
runtime paths as production. Mocking is permitted only in scoped test
layers; deployed application containers must remain unchanged.

# 14. Deployment Architecture

## 14.1 Deployment Targets

The front-end artifact is deployed to a front-end server. The BFF,
backend, Redis, PostgreSQL, and storage integrations are deployed to a
back-end server or back-end service cluster, subject to infrastructure
policy.

## 14.2 Packaging

Each deployable component is built into a Docker image. Image tags must
be immutable for promotion.

## 14.3 SSH Deployment

Deployment must be performed through scripted SSH workflows that copy
configuration, pull immutable images, run migrations in a controlled
step, and restart services safely.

## 14.4 Rollback

The deployment design must support rollback to a previously known image
tag and compatible configuration set.

## 14.5 Database Migration Discipline

Schema migrations must be explicit, versioned, automated, and safe to
run in CI and deployment pipelines.

# 15. Security Requirements

## 15.1 Authentication Security

Passwords must be hashed with a strong adaptive algorithm. Session or
token storage strategy must minimize browser risk.

## 15.2 Authorization Security

Role checks must be enforced in backend service handlers and on
administrative operations. Admin-only actions must return authorization
errors for subscribers.

## 15.3 Upload Security

Uploaded images must be validated for size and type, scanned or
sanitized per policy, and stored outside the writable application layer.

## 15.4 Network Security

Only the edge is internet-facing. Internal services communicate over
private networks. Datastores are not publicly exposed.

## 15.5 Audit and Compliance Posture

Security-relevant events must be logged in structured form without
leaking secrets or sensitive credential material.

# 16. Data Requirements

## 16.1 Referential Integrity

User, campsite, and review relationships must be preserved with explicit
foreign-key behavior.

## 16.2 Deletion Policy

Subscriber removal policy must be configurable as soft delete or hard
delete with retained references. The default recommendation is soft
deactivation.

## 16.3 Data Retention

Operational data retention and log retention windows must be
configuration-driven.

## 16.4 Seed Data

Local/test seed data must be scriptable and deterministic.

# 17. Testing Requirements

| **Test Layer** | **Purpose**                                 | **Required Coverage**                                              | **Runs In** |
|----------------|---------------------------------------------|--------------------------------------------------------------------|-------------|
| Unit           | Verify isolated functions/components        | Validation, domain rules, UI components                            | CI + local  |
| Integration    | Verify service and datastore behavior       | API handlers, DB ops, cache behavior, auth flows                   | CI + local  |
| Contract       | Verify BFF/backend schema compatibility     | Request/response contract adherence                                | CI          |
| End-to-End     | Verify user journeys through running system | Signup/login, add campsite, submit review, admin remove subscriber | CI + local  |

## 17.1 Acceptance Test Minimum Set

Unauthenticated users are redirected to login/signup for protected
routes.

A subscriber can sign up, log in, add a campsite with image, and submit
a thumbs-up or thumbs-down review.

Campsite listing shows average review and review count correctly.

The Review action displays all reviews for a campsite.

An admin can remove a subscriber and the action is audited.

The same container images run in local/test and production without code
change.

# 18. Observability and Operations

## 18.1 Logging

All services emit structured logs with request IDs and service
identifiers.

## 18.2 Health Endpoints

Each service exposes liveness and readiness endpoints suitable for
orchestration and deployment validation.

## 18.3 Metrics

The application must expose operational metrics hooks for request
latency, error rates, job timing, and cache usage.

## 18.4 Runbooks

The repository must include startup, deployment, rollback, migration,
and incident runbooks readable by humans and AI agents.

# 19. AI Implementation Constraints

## 19.1 Repository Self-Description

README and docs must include exact commands for local setup, tests,
builds, migrations, and deployment.

## 19.2 Machine-Readable Contracts

OpenAPI specs, JSON schemas, fixture files, and environment templates
must be present and current.

## 19.3 Script-First Operations

All repeated actions must be represented by scripts or make/task targets
rather than free-form human instructions.

## 19.4 Strict Failure Modes

Build, test, and startup steps must fail loudly on invalid config,
migration mismatch, missing secrets, or contract drift.

## 19.5 Human-Free Promotion

The deployment pipeline must be able to promote an already-built image
and environment-specific configuration set to the target servers via
SSH.

# 20. Open Design Decisions to Resolve During Implementation

Choose auth model: secure cookie session vs token-based SPA auth.

Choose image persistence strategy for deployed environments: mounted
volume vs object storage.

Choose canonical average-rating presentation: percent positive, ratio,
or signed score.

Choose subscriber removal behavior: soft deactivate default strongly
recommended.

Choose whether email verification is mandatory before first login or
feature access.

# 21. Release Acceptance Criteria

All mandatory user flows work end-to-end in a single-machine environment
and in deployed server environments.

No code differences exist between the tested artifact and the production
artifact.

Front-end and back-end deployment scripts work over SSH using immutable
image tags.

Config validation succeeds using only .env and JSON runtime files.

Database migrations apply cleanly and rollback procedures are
documented.

Security, observability, and audit requirements are demonstrably
implemented.

# 22. Recommended Deliverables for the AI Agent

React SPA application with login/signup, campsite list/detail, add
campsite form, and review UI.

FastAPI BFF with client-facing API routes and auth/context handling.

FastAPI backend service with domain logic, migrations, and database
access layer.

PostgreSQL schema migrations and seed data scripts.

Redis integration for caching/session or short-lived operational needs.

Dockerfiles, compose files, SSH deployment scripts, and environment
templates.

Automated unit, integration, contract, and end-to-end tests.

Operational runbooks and architecture documentation.
