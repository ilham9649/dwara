<!--
SYNC IMPACT REPORT
==================
Version Change: Initial → 1.0.0
Rationale: Initial ratification with core principles for code quality, testing, UX, security, and performance.

Modified Principles: N/A (initial version)

Added Sections:
- Core Principles: 5 principles covering code quality, testing, UX, security, and performance
- Quality Gates: Code review, testing, linting requirements
- Standards: Documentation, error handling, logging standards

Removed Sections: N/A (initial version)

Templates Requiring Updates:
- ✅ plan-template.md - Constitution check section validated
- ✅ spec-template.md - Requirements alignment validated
- ✅ tasks-template.md - Task categorization validated
- ⚠ README.md - May need reference to constitution for development workflow

Follow-up TODOs: None

Ratification Date: 2026-03-10 (Initial)
Last Amended Date: 2026-03-10
-->

# BAMF Constitution

## Core Principles

### I. Code Quality Excellence

All code MUST adhere to established patterns and maintain high standards:

- **Simplicity First**: Prefer clear, straightforward solutions over clever tricks. Code should be readable by any team member without extensive comments. Complex logic MUST be justified in architecture reviews.

- **No Comments Policy**: Do not add code comments unless explicitly required. Self-documenting code with descriptive names is preferred. Comments that explain "what" are forbidden; comments explaining "why" (for complex business logic or algorithms) are the only acceptable exception.

- **Error Handling**: Every function that can fail MUST handle errors explicitly. Never silently ignore errors. Wrap errors with context using language-specific patterns (e.g., `fmt.Errorf` with `%w` in Go, `raise from` in Python). Error messages MUST include: what failed, why it failed, and how to fix it.

- **Dependency Management**: Use only dependencies already present in the codebase. Never introduce new libraries without explicit justification and review. Regularly audit dependencies for vulnerabilities using Trivy.

- **Type Safety**: Prefer strong typing. Use type hints in Python, interfaces in Go, and TypeScript for the web UI. Avoid `any` or `interface{}` unless absolutely necessary.

- **Naming Conventions**: Follow language-specific conventions:
  - Go: `CamelCase` for exported, `camelCase` for private, `UPPER_SNAKE` for constants
  - Python: `snake_case` for variables/functions, `PascalCase` for classes, `UPPER_SNAKE` for constants
  - TypeScript/JavaScript: `camelCase` for variables/functions, `PascalCase` for classes/interfaces

**Rationale**: High-quality code reduces bugs, improves maintainability, and enables faster development. The "no comments" policy forces better naming and structure, making the code self-explanatory. Proper error handling ensures observability and debuggability.

### II. Testing Standards (NON-NEGOTIABLE)

Testing is mandatory and follows strict discipline:

- **Test Coverage**: All code MUST have test coverage. Aim for 80%+ coverage across unit tests, integration tests, and contract tests. Critical paths (authentication, authorization, certificate issuance) MUST have 90%+ coverage.

- **Test Organization**: Organize tests by type:
  - **Unit tests**: Test individual functions/components in isolation
  - **Contract tests**: Validate interface contracts between Go, Python, and TypeScript services
  - **Integration tests**: Test end-to-end flows (e.g., login → SSH tunnel → session audit)
  - **E2E tests**: Validate complete user journeys

- **Test-First Development**: New features MUST follow TDD:
  1. Write tests first (they MUST fail)
  2. Implement feature to make tests pass
  3. Refactor without changing behavior

- **Test Isolation**: Tests MUST be independent and run in any order. Use fixtures/factories for test data. Mock external dependencies (databases, external APIs) appropriately.

- **Security Testing**: Security tests MUST run in CI on every push:
  - Semgrep (SAST) for OWASP Top 10 and BAMF-specific rules
  - Trivy for container image CVE scanning
  - Nuclei (DAST) for security validations against running stack

- **Performance Testing**: Performance tests MUST validate critical thresholds:
  - SSH tunnel establishment: <500ms p95
  - API request latency: <200ms p95
  - Bridge throughput: 1000+ concurrent tunnels
  - Certificate issuance: <100ms p95

**Rationale**: BAMF handles security-critical operations. Thorough testing prevents security vulnerabilities, ensures reliability, and catches regressions early. The non-negotiable stance on testing reflects the security-sensitive nature of the product.

### III. User Experience Consistency

All user-facing interfaces MUST provide consistent, intuitive experiences:

- **CLI UX**: The `bamf` CLI MUST follow POSIX conventions:
  - Use standard flags (`--help`, `--version`, `-v`/`--verbose`, `-q`/`--quiet`)
  - Support human-readable output by default with optional JSON via `--json`
  - Write regular output to stdout, errors to stderr
  - Use exit codes (0 for success, 1 for errors)
  - Provide clear error messages with actionable fixes
  - Support shell completion (bash, zsh, fish)

- **Web UI Consistency**: The Next.js web UI MUST use shadcn/ui components consistently:
  - Follow the component library design system
  - Use consistent color schemes and typography
  - Ensure responsive design works on mobile, tablet, desktop
  - Provide loading states and empty states
  - Show clear success/error feedback for all user actions
  - Support keyboard navigation for accessibility

- **Error Messages**: All errors MUST be user-friendly:
  - Explain what happened in plain language
  - Provide context (what operation, what resource)
  - Suggest actionable next steps
  - Include correlation IDs for support
  - Avoid technical jargon unless necessary

- **Feedback Latency**: All user interactions MUST provide feedback within 100ms:
  - Show loading indicators for operations >500ms
  - Provide optimistic UI updates where appropriate
  - Display progress for long-running operations (e.g., tunnel establishment)

**Rationale**: BAMF is used daily by developers and operators. Consistent UX reduces cognitive load, speeds adoption, and reduces support burden. The CLI and web UI are primary touchpoints and must be polished.

### IV. Security Requirements

Security is paramount; all code MUST adhere to strict security practices:

- **Certificate-Based Trust**: All trust relationships MUST use certificates, never shared secrets:
  - x509 certificates for API and bridge communication (mTLS)
  - SSH certificates for SSH access (no static keys)
  - Short-lived certificates with configurable TTL (default: 1 hour)
  - Certificate revocation support for emergency access revocation

- **No Secrets in Code**: Never hardcode secrets, API keys, or credentials:
  - Use environment variables for configuration
  - Inject secrets from Kubernetes secrets at runtime
  - Rotate certificates and keys automatically
  - Never log secrets or sensitive data

- **Audit Logging**: All security-relevant events MUST be logged:
  - Authentication successes and failures
  - Authorization decisions (allow/deny with reason)
  - Certificate issuance and revocation
  - Session creation and termination
  - Tunnel establishment and teardown
  - Structured JSON format with timestamps, user, IP, action, result
  - Exportable via REST API for SIEM integration

- **Input Validation**: All inputs MUST be validated and sanitized:
  - Validate user input at API boundaries
  - Sanitize database queries to prevent injection
  - Validate file paths to prevent directory traversal
  - Enforce size limits on uploads and inputs

- **Least Privilege**: Components MUST run with minimal permissions:
  - Containers run as non-root users
  - Use RBAC for Kubernetes API access
  - Restrict network policies to required traffic only
  - Principle of least privilege for all service accounts

- **Session Recording**: All sessions MUST be recorded when enabled:
  - SSH sessions in asciicast v2 format for playback
  - Database queries logged (PostgreSQL and MySQL wire protocol)
  - HTTP requests/responses captured for web app proxy
  - Recordings indexed by correlation ID and time

**Rationale**: BAMF is an access management system. Security failures compromise the entire trust model. These requirements protect against common vulnerabilities and provide audit trails for compliance.

### V. Performance Requirements

The system MUST meet performance targets for scale:

- **Tunnel Performance**: Tunnel operations MUST be low-latency and high-throughput:
  - SSH tunnel establishment: <500ms p95
  - TCP tunnel connection: <300ms p95
  - Maximum concurrent tunnels per bridge: 1000+
  - Tunnel throughput: 100+ Mbps per connection
  - Survive bridge pod failure with reconnection <5s

- **API Performance**: The API MUST respond quickly under load:
  - Authentication requests: <200ms p95
  - Certificate issuance: <100ms p95
  - Audit log queries: <500ms p95 (with pagination)
  - Concurrent users: 1000+ without degradation
  - 99th percentile latency targets for all endpoints

- **Resource Limits**: Components MUST operate within resource bounds:
  - Bridge memory: <512MB per 100 active tunnels
  - API memory: <512MB baseline
  - Agent memory: <256MB
  - Web UI bundle size: <2MB (gzipped)
  - Container startup time: <10s

- **Caching**: Frequently accessed data MUST be cached:
  - Role definitions cached in memory (TTL: 5 minutes)
  - User sessions cached in Redis
  - Audit log pagination state cached
  - Certificate validity cached for active sessions

- **Database Performance**: Database operations MUST be optimized:
  - Use connection pooling
  - Index all query fields used in filters
  - Avoid N+1 queries
  - Use prepared statements
  - Database query timeout: 5s

**Rationale**: BAMF serves as a critical infrastructure component. Poor performance impacts developer productivity and operational reliability. These targets ensure BAMF scales with organization growth.

## Quality Gates

Code changes MUST pass all quality gates before merging:

- **Testing**: All tests MUST pass (unit, integration, contract, E2E)
- **Linting**: All linters MUST pass with zero errors:
  - Go: `golangci-lint run`
  - Python: `ruff check` + `ruff format`
  - TypeScript: `eslint` + `prettier`
- **Type Checking**: Strict type checking MUST pass:
  - Python: `mypy src/`
  - TypeScript: `tsc --noEmit`
- **Security Scanning**: MUST pass:
  - Semgrep SAST scan
  - Trivy image CVE scan
- **Documentation**: Changes MUST update relevant documentation
- **Code Review**: At least one approval from team member required

## Standards

### Documentation Standards

- **README.md**: MUST be up-to-date with setup instructions
- **API Documentation**: OpenAPI/Swagger spec MUST match implementation
- **Inline Docs**: Use docstrings for all public functions/classes
- **Changelog**: Update CHANGELOG.md for breaking changes
- **Architecture Docs**: Document major design decisions in ADR format

### Error Handling Standards

- **Go**: Wrap errors with `fmt.Errorf("context: %w", err)` and use `errors.Is()`/`errors.As()` for checking
- **Python**: Use specific exceptions and `raise from` for chaining; never use bare `except:`
- **TypeScript**: Use custom error classes extending `Error` with error codes

### Logging Standards

- **Structured Logging**: Use JSON format for all logs
- **Log Levels**: DEBUG, INFO, WARN, ERROR (use appropriately)
- **Context**: Include correlation IDs, user IDs, request IDs in all logs
- **Sensitive Data**: Never log passwords, tokens, or secrets
- **Performance**: Log request latency for all API endpoints

## Governance

This constitution supersedes all other development practices. Amendments require:

- **Documentation**: Proposed changes MUST be documented with rationale
- **Approval**: Changes require consensus from maintainers
- **Migration Plan**: Breaking changes MUST include a migration path
- **Versioning**: Follow semantic versioning (MAJOR.MINOR.PATCH):
  - MAJOR: Backward-incompatible principle removals or redefinitions
  - MINOR: New principles or materially expanded guidance
  - PATCH: Clarifications, wording fixes, non-semantic refinements
- **Compliance Review**: All PRs and code reviews MUST verify compliance with constitution

**Development Guidance**: Refer to `docs/development.md` for runtime development workflow and `AGENTS.md` for build/lint/test commands.

---

**Version**: 1.0.0 | **Ratified**: 2026-03-10 | **Last Amended**: 2026-03-10
