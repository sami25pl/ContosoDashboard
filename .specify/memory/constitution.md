# ContosoDashboard Constitution

## Core Principles

### I. Training-Only Scope
ContosoDashboard is an educational application, not a production reference or supported product. Changes MUST preserve that scope, MUST NOT imply that mock authentication or other training shortcuts are production-ready, and MUST document material limitations introduced by a feature.

### II. Offline-First Operation
The training application MUST remain usable locally without requiring cloud services or external service accounts. When infrastructure needs a replaceable implementation, application logic SHOULD depend on an abstraction so local training implementations remain practical.

### III. Layered Architecture
User interface, business services, domain models, and data access MUST remain separated according to the existing project structure. New behavior SHOULD use dependency injection and existing service boundaries rather than embedding persistence or business rules in UI components.

### IV. Authorization Boundaries
Protected functionality MUST enforce authentication and role requirements, and access to user- or project-scoped data MUST verify resource ownership or membership. Authorization MUST be enforced at the service/data boundary as well as in the UI; hiding a page or control alone is insufficient.

### V. Simplicity and Clarity
Changes MUST favor focused, maintainable solutions over unnecessary complexity. User-visible behavior and known limitations MUST be documented where relevant, and changes MUST be validated with the narrowest applicable build or test checks.

## Additional Constraints

- Keep local training setup independent of cloud credentials and subscriptions.
- Preserve the separation between Blazor pages, application services, models, and Entity Framework Core data access.
- Do not describe mock authentication, local-only infrastructure, or training security controls as production-ready.

## Development Workflow

- New features MUST identify their user-facing behavior and relevant authorization boundary before implementation.
- Changes to data access MUST preserve the service-layer authorization checks applicable to the affected resource.
- Build and run the relevant tests or checks before considering implementation complete; report any remaining warnings or limitations.

## Governance

Maintainers may amend this constitution when they document the rationale and update the version and amendment date. This constitution guides feature specifications, plans, implementation, and reviews. Conflicts MUST be resolved by updating the conflicting artifact or documenting an approved, scoped exception; principles MUST NOT be silently ignored.

**Version**: 1.0.0 | **Ratified**: 2026-11-19 | **Last Amended**: 2026-11-19
