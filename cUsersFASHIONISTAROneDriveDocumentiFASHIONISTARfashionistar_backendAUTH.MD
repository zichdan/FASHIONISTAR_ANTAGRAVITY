# BACKEND_AUTH.MD - Authentication & Authorization Architectural Instructions

## Overview
This document provides detailed, step-by-step instructions for implementing a robust, industrial-grade Authentication and Authorization system for Fashionistar. As a senior fullstack developer with 10+ years of experience, these instructions ensure enterprise-level security, performance, and maintainability. We will restructure the existing `userauths` app into `apps/authentication` following modular monolith principles, with strict typing, Pydantic schemas, optimized DRF, comprehensive error handling, and robust comments/docstrings for future maintainability.

**Key Principles:**
- **Security First:** Multi-factor authentication (MFA), JWT with refresh tokens, rate limiting, biometric auth (fingerprint), and audit logging.
- **Performance:** Optimized DRF with caching, async tasks where needed, minimal database hits, and N+1 query prevention via related_name.
- **Maintainability:** Strict typing, service layers, robust comments/docstrings in every file/line, try-except blocks, and separation of concerns.
- **Scalability:** Monolithic start, microservice-ready structure with no cross-app FKs to avoid circular imports.
- **Compliance:** GDPR/CCPA-ready with encrypted fields, soft deletes, and detailed logs.
- **Django Version:** Upgrading to Django 6.0 for latest features (async views, improved ORM, better security). Use async where necessary (e.g., external API calls for OAuth).
- **Audit Logging:** Implement `django-auditlog` for lightweight, efficient tracking of staff actions (create, update, delete) without full snapshots, suitable for compliance and security.

**Tools & Libraries:**
- Django 6.0 + DRF for APIs.
- Pydantic for schema validation.
- `drf-api-logger` for request/response logging.
- `django-encrypted-model-fields` for sensitive data.
- JWT via `djangorestframework-simplejwt`.
- `django-auditlog` for audit trails.
- Custom validators and type hints.
- Async support with `asyncio` and `httpx` for OAuth.
- WebAuthn for fingerprint biometric auth.

**File Structure for `apps/authentication` App:**
```
apps/authentication/
├── __init__.py
├── admin.py           # Admin interface with auditlog integration
├── apps.py            # AppConfig
├── models.py          # Enhanced User model with MFA, profiles, Vendor/Client/Staff models (Profile dissolved, wallet_balance moved here)
├── permissions.py     # Custom async permission classes (IsVendor, IsStaff, etc.)
├── serializers.py     # Pydantic-integrated serializers with validation
├── views.py           # Optimized views with try-except and logging
├── urls.py            # Clean URL patterns
├── services/
│   ├── __init__.py
│   ├── auth_service.py  # Business logic layer with types
│   └── helpers.py       # Utility functions
├── cores/
│   ├── __init__.py
│   ├── api_ingress.py   # Input validation and preprocessing
│   ├── text_processor.py # Email/SMS processing
│   └── orchestrator.py   # Workflow coordination
├── tests/
│   ├── __init__.py
│   ├── test_models.py
│   ├── test_views.py
│   └── test_services.py
└── types.py            # Python type definitions
```

**New App: `apps/common` (Utilities)**
To avoid circular imports and provide shared utilities, create a new app `apps/common` containing:
- `TimeStampedModel`: Base model with created_at/updated_at.
- `SoftDeleteModel`: Adds is_deleted/deleted_at for compliance.
- `permissions.py`: Async permission classes like `IsVendor`, `IsStaff`, `IsClient`, `IsOwner`, `IsSupport`, `IsEditor`, `IsSales`.
- Other shared models/utilities.

**Borrowed Concepts from New Repositories:**
- **From hng-stage-2-countries-main:** Simple, clean Django setup with UUID PKs and basic API structure for data management (adapt for user profiles/countries).
- **From hng-stage-3-agent-main:** RAG-inspired vector embeddings and semantic search (borrow for advanced auth features like AI-powered anomaly detection in logs). Async processing for AI tasks.

**Architectural Concepts Incorporated:**
- **Modular Monolith:** Hard boundaries between apps, no cross-app FKs, event-driven via outbox pattern.
- **Event-Driven:** Use outbox table for auth events (e.g., login success).
- **Zero-Downtime Migrations:** Expand-contract strategy for schema changes.
- **Query Performance:** Budgets, explain(), pg_stat_statements; use related_name in all FKs to prevent N+1 (e.g., in Vendor/Product models, related_name='vendors' or 'products').
- **Horizontal Scaling:** Partitioning, read replicas, PgBouncer.
- **Caching:** Intent-based with versioned keys, dogpile protection.
- **Observability:** OpenTelemetry, Jaeger, Sentry for N+1 detection.
- **Governance:** Contract tests, migration tests, CODEOWNERS.
- **API Gateway:** Kong/Hasura for routing.
- **Audit Libraries:** `django-auditlog` for lightweight logging (diffs in JSON), combined with `django-simple-history` for full snapshots on critical models.

**Google Auth Recommendation:**
For industrial-grade synchronization with our monolithic architecture, use **Next.js SSR with NextAuth.js**. It provides server-side session management, integrates seamlessly with JWT, handles OAuth flows securely on the server, and maintains consistency with our backend. Client-side would expose tokens, risking security; SSR ensures server-rendered auth pages with proper hydration.

**Existing Userauths Analysis:**
- Registration allows email OR phone (strict validation: one per user, no both).
- OTP via email/SMS, Redis storage.
- Maintain these validations in new User model; add fields for auth method (google/email/phone).
- Dissolve Profile model: Merge fields into User (image, bio, etc.), move wallet_balance to User.
- Add Vendor/Client models following Helssa's Doctor/Patient pattern (OneToOne to User, with profiles).
- Staff roles: Add choices like 'support', 'reviewer', 'assistant'; audit all actions with `django-auditlog`.

**Service Layer Queries:** Structure like microservices—each app queries only its own models. No cross-app imports/FKs. Use related_name for optimizations (e.g., in Product model, vendor FK has related_name='products' to avoid N+1 when querying Vendor.products).

## Step-by-Step Implementation Instructions

### Step 1: Create Common App and Restructure
- **Action:** Create `apps/common` with TimeStampedModel, SoftDeleteModel, async permissions. Move `userauths` to `apps/authentication`, create subfolders.
- **Why:** Shared utilities prevent duplication; restructure ensures modularity.
- **Comments:** `# Common utilities app for shared models and permissions` in common/__init__.py.
- **Docstrings:** Add module docstrings explaining purpose.
- **Try-Except:** N/A.
- **Logging:** N/A.
- **Async:** Permissions are async.

### Step 2: Enhance Models with Strict Typing and Encryption
- **Action:** Update `models.py` to inherit from common.TimeStampedModel/SoftDeleteModel, add MFA fields, encrypted email/phone, auth_method (google/email/phone). Dissolve Profile into User, move wallet_balance. Add Vendor/Client models with OneToOne to User. Add staff roles with auditlog.
- **Why:** Security and type safety; encrypted fields prevent breaches; role separation.
- **Comments:** `# Enhanced User model with MFA support and merged Profile` above class.
- **Docstrings:** Class and method docstrings, e.g., """User model with authentication methods.""".
- **Try-Except:** Wrap model saves in try-except for integrity errors.
- **Logging:** Log model creation/updates via signals to audit logs.
- **Async:** Use async signals if needed.

### Step 3: Implement Service Layer with Types
- **Action:** Create `services/auth_service.py` with typed functions (e.g., `def authenticate_user(user_data: UserLoginSchema) -> AuthResult`). Use Pydantic schemas for input/output. Add OAuth service for Google. Queries only within app.
- **Why:** Decouples logic from views; enables testing and reuse.
- **Comments:** `# Authenticate user with MFA check` for each function.
- **Docstrings:** Function docstrings with params/returns, e.g., """Authenticate user. Args: user_data. Returns: AuthResult.""".
- **Try-Except:** Catch authentication failures, raise custom exceptions.
- **Logging:** Use `drf-api-logger` decorators on service methods.
- **Async:** Make OAuth calls async with httpx.

### Step 4: Optimize Serializers with Pydantic
- **Action:** Refactor `serializers.py` to integrate Pydantic models for validation. Add custom validators for email/phone/auth_method.
- **Why:** Faster validation than DRF alone; type safety.
- **Comments:** `# Pydantic-backed serializer for login` .
- **Docstrings:** Serializer class docstrings.
- **Try-Except:** Validate inputs in try blocks, return error messages.
- **Logging:** Log serialization errors.
- **Async:** N/A.

### Step 5: Build Views with Error Handling and Messages
- **Action:** Rewrite `views.py` to use service layer, add human-readable messages (e.g., "Login successful" or "Invalid credentials"). Implement MFA, Google OAuth, biometric endpoints.
- **Why:** User-friendly responses; prevents silent failures.
- **Comments:** `# Handle user login with comprehensive error handling` .
- **Docstrings:** View method docstrings.
- **Try-Except:** Wrap all logic in try-except, return 500 on unexpected errors.
- **Logging:** Decorate views with `drf-api-logger` for full request logs.
- **Async:** Use async views for OAuth.

### Step 6: Add Cores for Advanced Processing
- **Action:** Implement `cores/api_ingress.py` for input sanitization, `cores/text_processor.py` for OTP generation, `cores/orchestrator.py` for auth workflows. Add AI core for anomaly detection using RAG concepts.
- **Why:** Handles complex auth flows like MFA verification.
- **Comments:** `# Orchestrate MFA workflow` .
- **Docstrings:** Core function docstrings.
- **Try-Except:** Handle external API calls (e.g., SMS) with retries.
- **Logging:** Log core operations.
- **Async:** Async for AI processing.

### Step 7: Update URLs and Admin
- **Action:** Clean `urls.py` with named patterns. Enhance `admin.py` with encrypted field handling and auditlog integration.
- **Why:** Maintainability and security.
- **Comments:** `# Auth URL patterns` .
- **Docstrings:** URL pattern docstrings.
- **Try-Except:** N/A.
- **Logging:** N/A.
- **Async:** N/A.

### Step 8: Comprehensive Testing
- **Action:** Write tests in `tests/` for models, services, views with 95%+ coverage.
- **Why:** Ensures reliability.
- **Comments:** `# Test auth service` .
- **Docstrings:** Test method docstrings.
- **Try-Except:** In test assertions.
- **Logging:** N/A.
- **Async:** Async tests for views.

### Step 9: Integrate Logging Library
- **Action:** Install and configure `drf-api-logger` and `django-auditlog` in settings, add to middleware.
- **Why:** Tracks all API interactions for debugging/production.
- **Comments:** `# API logging middleware` .
- **Docstrings:** Middleware docstrings.
- **Try-Except:** N/A.
- **Logging:** Self-explanatory.
- **Async:** N/A.

### Step 10: Frontend Alignment (Brief)
- **Action:** Update frontend auth components with typed interfaces, error handling. Use Next.js SSR with NextAuth for Google OAuth.
- **Why:** Consistency.
- **Comments:** `# Auth component with error states` .
- **Docstrings:** Component docstrings.
- **Try-Except:** In API calls.
- **Logging:** Console logs for dev.
- **Async:** Async fetches.

## 5 Own Recommendations (Implemented)
1. **Implement OAuth2 Integration:** Add Google login using hybrid pattern (client gets ID token, server verifies), backed by django-allauth concepts but optimized for API-first architecture with typed schemas.
2. **Add Biometric Auth:** Integrate WebAuthn for fingerprint authentication, enhancing security for mobile users (no face ID as per request).
3. **Rate Limiting per User:** Beyond IP, limit login attempts per user to prevent brute force (e.g., 5 failed attempts locks account for 15 mins).
4. **Session Management:** Use Redis for session storage/cache with automatic expiration to reduce DB hits.
5. **Audit Trail Expansion:** Log device fingerprints and geolocation for compliance in the Audit Log.

**Next:** Proceed to implementation upon approval.
