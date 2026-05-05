## Project Overview
This project is an **ASP.NET Core 10 REST API** built with **Minimal APIs** and organized using **Vertical Slice Architecture**.

The codebase is organized by **business feature / use case**, not by technical layers such as Controllers, Services, or Repositories.

Primary goals:
- keep each use case self-contained
- prefer simple, explicit code over heavy abstraction
- keep business rules separate from HTTP and persistence concerns
- optimize for maintainability, readability, and safe refactoring

---

## Architecture Style

### Chosen architecture
Use:
- **ASP.NET Core 10**
- **Minimal APIs**
- **Vertical Slice Architecture**
- **EF Core** for persistence
- **feature-local request/response models**
- **domain policy / calculator classes** for business rules

Do **not** organize the project primarily by:
- Controllers
- Services
- Repositories
- DTOs
- Validators

Instead, organize by **feature**.

---

## Core Architecture Decisions

### 1. Feature-first organization
Each business use case must live in its own folder under `Features/`.

Examples:
- `Employees/GetEmployees`
- `Employees/GetEmployeeById`
- `Dependents/GetEmployeeDependents`
- `Paychecks/GetEmployeePaycheck`

Each feature folder should contain only the files needed for that use case.

Typical contents:
- `Endpoint.cs` — route registration
- `Handler.cs` — use case logic
- `Request.cs` — request contract if needed
- `Response.cs` — response contract
- `Mapping.cs` — DTO/domain mapping if needed
- `Validator.cs` — validation when needed

### 2. Thin endpoints
Endpoint classes should:
- define the route
- define metadata
- delegate immediately to the handler

Do not put business logic directly in endpoint registration.

### 3. Focused handlers
Each handler should implement one use case only.

Handlers may:
- query the database
- call domain policies/calculators
- return result DTOs

Handlers should not become generic utility classes shared across unrelated features.

### 4. Business rules belong in domain policy/calculator classes
Complex business calculations must not be embedded directly in endpoints.

Examples:
- paycheck calculation
- benefits deduction rules
- age-based cost calculation
- high-income adjustment logic

Put such logic in domain-level classes like:
- `BenefitsCostPolicy`
- `PaycheckCalculator`

### 5. Infrastructure stays isolated
Persistence and technical setup belong in `Infrastructure/`.

Examples:
- `AppDbContext`
- EF Core configuration
- seed data
- dependency injection registration

Avoid leaking infrastructure concerns into every feature.

### 6. Local DTOs
Request and response models should usually stay inside the feature that owns them.

Do not create a giant shared DTO folder unless there is a clear proven need.

### 7. Prefer explicit code over premature abstraction
Avoid:
- generic repository pattern
- generic service base classes
- premature mediator abstractions unless required later
- excessive inheritance
- “helper” dumping grounds

Prefer:
- explicit handlers
- explicit queries
- explicit response models

### 8. Async by default
Use async I/O for database access and external calls.

Prefer:
- `ToListAsync`
- `FirstOrDefaultAsync`
- `SingleOrDefaultAsync`

Always pass `CancellationToken` through async flows.

### 9. Use REST-style routes
Routes should model resources clearly.

Preferred examples:
- `GET /employees`
- `GET /employees/{employeeId}`
- `GET /employees/{employeeId}/dependents`
- `GET /employees/{employeeId}/paycheck`

### 10. Shared folder is only for cross-cutting concerns
Use `Shared/` only for small reusable technical pieces.

Good candidates:
- endpoint registration abstraction
- common extensions
- error handling helpers
- route constants

Do not put feature/business logic into `Shared/`.

---

## Project Folder Structure

```text
src/
└── BenefitsApi/
    ├── Program.cs
    ├── Features/
    │   ├── Employees/
    │   │   ├── GetEmployees/
    │   │   │   ├── Endpoint.cs
    │   │   │   ├── Handler.cs
    │   │   │   ├── Response.cs
    │   │   │   └── Mapping.cs
    │   │   └── GetEmployeeById/
    │   │       ├── Endpoint.cs
    │   │       ├── Handler.cs
    │   │       ├── Response.cs
    │   │       └── Mapping.cs
    │   ├── Dependents/
    │   │   └── GetEmployeeDependents/
    │   │       ├── Endpoint.cs
    │   │       ├── Handler.cs
    │   │       ├── Response.cs
    │   │       └── Mapping.cs
    │   └── Paychecks/
    │       └── GetEmployeePaycheck/
    │           ├── Endpoint.cs
    │           ├── Handler.cs
    │           ├── Response.cs
    │           ├── Calculator.cs
    │           └── Mapping.cs
    ├── Domain/
    │   ├── Employees/
    │   │   ├── Employee.cs
    │   │   ├── Dependent.cs
    │   │   └── DependentType.cs
    │   └── Paychecks/
    │       ├── PaycheckBreakdown.cs
    │       └── BenefitsCostPolicy.cs
    ├── Infrastructure/
    │   ├── Persistence/
    │   │   ├── AppDbContext.cs
    │   │   └── SeedData.cs
    │   └── DependencyInjection.cs
    ├── Shared/
    │   ├── Abstractions/
    │   │   └── IEndpoint.cs
    │   ├── Extensions/
    │   │   └── EndpointRegistrationExtensions.cs
    │   └── Constants/
    │       └── ApiRoutes.cs
    └── appsettings.json
```

---

## Feature Breakdown for This Project

### Employees
Responsibilities:
- view all employees
- view one employee
- expose employee basic data

Expected slices:
- `GetEmployees`
- `GetEmployeeById`

### Dependents
Responsibilities:
- view dependents for an employee
- expose relationship type and age-related data

Expected slices:
- `GetEmployeeDependents`

### Paychecks
Responsibilities:
- calculate employee paycheck
- calculate benefit deductions
- return deduction breakdown

Expected slices:
- `GetEmployeePaycheck`

---

## Business Rules for This Project

The API must support these calculation rules:

- 26 paychecks per year
- employee base cost: `$1,000` per month
- each dependent adds `$600` per month
- employees with salary greater than `$80,000` incur an additional `2%` of yearly salary in benefits costs
- dependents older than `50` add `$200` per month each

These rules should live in a dedicated policy/calculation class, not inline in the endpoint.

---

## Minimal API Conventions

### Endpoint registration
Each feature should expose an endpoint mapping class implementing a shared contract, for example `IEndpoint`.

### Preferred pattern
- `Endpoint.cs` contains only route mapping and metadata
- `Handler.cs` contains the use case logic
- `Response.cs` contains output DTO
- `Request.cs` exists only when the endpoint accepts a body/query contract

### Example responsibilities
`Endpoint.cs`
- map route
- attach tags
- declare response types
- delegate to handler

`Handler.cs`
- load domain data
- call business policy
- build response DTO
- return `Results.Ok(...)`, `Results.NotFound()`, etc.

---

## Domain Modeling Conventions

### Employee
Represents an employee with:
- identity
- name
- annual salary
- collection of dependents

### Dependent
Represents:
- spouse
- domestic partner
- child

Use an enum such as:
- `Spouse`
- `DomesticPartner`
- `Child`

### Domain rules
Even if some rules are not yet enforced by write endpoints, the model must reflect them clearly for future evolution.

Example:
- an employee may only have one spouse or one domestic partner, not both
- an employee may have unlimited children

---

## Persistence Conventions

Use EF Core directly through `AppDbContext`.

Guidelines:
- query the DbContext from handlers
- avoid introducing a generic repository layer
- keep EF configuration in Infrastructure/Persistence
- use async queries
- use includes only when needed by the use case

Example:
- `GetEmployeeById` may include dependents
- `GetEmployees` may return lightweight summary data without unnecessary graph loading

---

## Dependency Injection Conventions

Register:
- DbContext
- domain policies/calculators
- infrastructure services
- endpoint discovery/registration helpers

Do not register large “manager” or “service” classes unless a real need emerges.

---

## Validation Conventions

Validate input close to the feature.

Preferred options:
- feature-local validator class
- Minimal API validation support
- endpoint filters for cross-cutting validation if needed

Do not centralize unrelated validation rules into one large validation folder.

---

## Error Handling Conventions

Use consistent HTTP responses:
- `200 OK` for successful reads
- `404 Not Found` when resource does not exist
- `400 Bad Request` for invalid input
- `500` only for unhandled server errors

Prefer standardized problem details for API errors.

---

## Naming Conventions

### Folder names
Use business-oriented names:
- `Employees`
- `Dependents`
- `Paychecks`

### Slice names
Use action-oriented names:
- `GetEmployees`
- `GetEmployeeById`
- `GetEmployeePaycheck`

### Class names
Prefer simple explicit names inside each slice:
- `Endpoint`
- `Handler`
- `Request`
- `Response`
- `Validator`

Because the folder already provides context.

---

## What Copilot Should Prefer When Generating Code

When generating new code for this project, prefer the following:

1. Add new endpoints as **new vertical slices** under `Features/`
2. Keep route mapping inside `Endpoint.cs`
3. Keep logic inside `Handler.cs`
4. Keep DTOs local to the slice
5. Put reusable business rules into `Domain/`
6. Put persistence concerns into `Infrastructure/`
7. Use async EF Core APIs
8. Accept and pass `CancellationToken`
9. Return typed, clear API responses
10. Avoid unnecessary abstractions

---

## What Copilot Should Avoid

Avoid generating:
- MVC Controllers unless explicitly requested
- generic repositories
- massive service layers
- one shared DTO folder for the whole project
- business logic inside Program.cs
- business logic directly inside endpoint mapping
- utility classes that mix unrelated concerns
- static helper classes for core business rules unless clearly appropriate

---

## Preferred Example Flow for a New Feature

If a new use case is added, for example:
- “Get employee annual benefits summary”

Copilot should generate:

```text
Features/
└── Paychecks/
    └── GetEmployeeAnnualBenefitsSummary/
        ├── Endpoint.cs
        ├── Handler.cs
        ├── Response.cs
        └── Mapping.cs
```

And only add shared/domain/infrastructure files if there is a real architectural reason.

---

## Testing Guidance

When generating tests, prefer:
- feature-focused tests
- handler unit tests for business behavior
- endpoint/integration tests for HTTP behavior
- business rule tests for calculation policies

Do not overuse broad end-to-end tests for simple calculation scenarios.

Important scenarios for this project:
- employee with no dependents
- employee with one spouse
- employee with multiple children
- employee salary above 80,000
- dependent older than 50
- combinations of the above

---

## Summary

This project follows these principles:
- Minimal APIs
- Vertical Slice Architecture
- feature-first organization
- thin endpoints
- focused handlers
- isolated business rules
- isolated infrastructure
- explicit code over premature abstraction

Copilot should generate code that keeps the project simple, explicit, and organized around business use cases.
