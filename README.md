# ASP.NET Core Entity, DTO, Mapping, and Repository Development Guide

## Purpose

This guide defines a consistent workflow for building ASP.NET Core APIs with:

* Entity Framework Core
* DTOs
* Mapster
* Repository pattern
* Service layer
* Controller layer
* Related/nested response objects

The main goal is to prevent a common problem:

> Creating a Response DTO with related objects but forgetting to load or map the corresponding navigation properties.

For example:

```text
Entity
    ↓
Repository
    ↓
Service
    ↓
DTO
    ↓
Controller
    ↓
Frontend
```

The Entity, Repository, Mapping, and DTO must be designed together.

---

# 1. Core API Design Rule

Use this pattern:

### Request DTOs use IDs

Create and Update requests should normally receive foreign key IDs.

Example:

```csharp
public class OpportunityCreateDto
{
    public Guid CustomerId { get; set; }
    public Guid PipelineId { get; set; }
    public Guid StageId { get; set; }
    public Guid? AssignedUserId { get; set; }

    public string Name { get; set; } = null!;
    public decimal Value { get; set; }
}
```

The frontend sends:

```json
{
    "customerId": "guid",
    "pipelineId": "guid",
    "stageId": "guid",
    "assignedUserId": "guid",
    "name": "New Opportunity",
    "value": 50000
}
```

---

### Response DTOs use readable related objects

Instead of returning only:

```csharp
public Guid CustomerId { get; set; }
public Guid PipelineId { get; set; }
public Guid StageId { get; set; }
public Guid? AssignedUserId { get; set; }
```

return:

```csharp
public CustomerMinimalDto Customer { get; set; } = null!;
public PipelineMinimalDto Pipeline { get; set; } = null!;
public PipelineStageMinimalDto Stage { get; set; } = null!;
public UserMinimalDto? AssignedUser { get; set; }
```

The frontend can then display:

```text
Customer: ABC Corporation
Pipeline: Sales Pipeline
Stage: Negotiation
Assigned User: John Doe
```

without displaying GUIDs.

---

# 2. Development Order

When creating a new feature, follow this order:

```text
1. Database schema
        ↓
2. Entity
        ↓
3. Entity relationships
        ↓
4. Minimal DTOs
        ↓
5. Create DTO
        ↓
6. Update DTO
        ↓
7. Response DTO
        ↓
8. Mapster mapping
        ↓
9. Repository query
        ↓
10. Repository CRUD methods
        ↓
11. Service
        ↓
12. Controller
        ↓
13. Swagger testing
        ↓
14. Frontend
```

Do not jump directly to the frontend before verifying the API response.

---

# 3. Step 1 — Create the Entity

Start by defining the entity and all of its relationships.

Example:

```csharp
public class Opportunity
{
    public Guid Id { get; set; }

    public Guid CustomerId { get; set; }
    public Guid PipelineId { get; set; }
    public Guid StageId { get; set; }
    public Guid? AssignedUserId { get; set; }

    public string Name { get; set; } = null!;
    public decimal Value { get; set; }

    public Customer Customer { get; set; } = null!;
    public Pipeline Pipeline { get; set; } = null!;
    public PipelineStage Stage { get; set; } = null!;
    public ApplicationUser? AssignedUser { get; set; }
}
```

Immediately identify the relationships:

```text
Opportunity
├── Customer
├── Pipeline
├── Stage
└── AssignedUser
```

This list becomes the basis for the Response DTO and repository query.

---

# 4. Step 2 — Configure Entity Relationships

Make sure EF Core knows the relationships.

For example:

```text
Opportunity
    │
    ├── CustomerId → Customer
    ├── PipelineId → Pipeline
    ├── StageId → PipelineStage
    └── AssignedUserId → ApplicationUser
```

Do not proceed until the database relationships are correct.

---

# 5. Step 3 — Create Minimal DTOs

Minimal DTOs are used when another entity is included inside a Response DTO.

Example:

```csharp
public class CustomerMinimalDto
{
    public Guid Id { get; set; }
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public string? CompanyName { get; set; }
}
```

Pipeline:

```csharp
public class PipelineMinimalDto
{
    public Guid Id { get; set; }
    public string Name { get; set; } = null!;
}
```

Pipeline Stage:

```csharp
public class PipelineStageMinimalDto
{
    public Guid Id { get; set; }
    public string Name { get; set; } = null!;
}
```

User:

```csharp
public class UserMinimalDto
{
    public Guid Id { get; set; }
    public string FirstName { get; set; } = null!;
    public string LastName { get; set; } = null!;
}
```

Minimal DTOs should contain only information necessary for identifying/displaying the related entity.

---

# 6. Step 4 — Create Request DTOs

Create DTO:

```csharp
public class OpportunityCreateDto
{
    public Guid CustomerId { get; set; }
    public Guid PipelineId { get; set; }
    public Guid StageId { get; set; }
    public Guid? AssignedUserId { get; set; }

    public string Name { get; set; } = null!;
    public decimal Value { get; set; }
}
```

Update DTO:

```csharp
public class OpportunityUpdateDto
{
    public Guid CustomerId { get; set; }
    public Guid PipelineId { get; set; }
    public Guid StageId { get; set; }
    public Guid? AssignedUserId { get; set; }

    public string Name { get; set; } = null!;
    public decimal Value { get; set; }
}
```

The important rule:

```text
Create/Update
    ↓
Use FK IDs
```

---

# 7. Step 5 — Create the Response DTO

The Response DTO should represent what the frontend actually needs to display.

Example:

```csharp
public class OpportunityResponseDto
{
    public Guid Id { get; set; }

    public CustomerMinimalDto Customer { get; set; } = null!;
    public PipelineMinimalDto Pipeline { get; set; } = null!;
    public PipelineStageMinimalDto Stage { get; set; } = null!;
    public UserMinimalDto? AssignedUser { get; set; }

    public string Name { get; set; } = null!;
    public decimal Value { get; set; }
}
```

Now create a relationship checklist:

```text
OpportunityResponseDto

Customer
    ↓
Opportunity.Customer

Pipeline
    ↓
Opportunity.Pipeline

Stage
    ↓
Opportunity.Stage

AssignedUser
    ↓
Opportunity.AssignedUser
```

This checklist tells you exactly what the repository must load.

---

# 8. Step 6 — Create Mapster Mapping

Create your mapping configuration at the same time as the Response DTO.

Normal matching properties may be mapped automatically:

```csharp
config.NewConfig<Opportunity, OpportunityResponseDto>();
```

If property names are different, explicitly map them.

Example:

```csharp
config.NewConfig<Lead, LeadResponseDto>()
    .Map(dest => dest.Customer, src => src.ConvertedCustomer);
```

Entity:

```csharp
public Customer? ConvertedCustomer { get; set; }
```

DTO:

```csharp
public CustomerMinimalDto? Customer { get; set; }
```

Because the names differ, explicitly tell Mapster how to map them.

---

# 9. Step 7 — Register Mapster

Use a centralized mapping configuration.

Example:

```csharp
using CRMSystem.Models.DTOs;
using CRMSystem.Models.Entities;
using Mapster;

namespace CRMSystem.Mapping
{
    public class MappingConfig : IRegister
    {
        public void Register(TypeAdapterConfig config)
        {
            config.NewConfig<Opportunity, OpportunityResponseDto>();

            config.NewConfig<Lead, LeadResponseDto>()
                .Map(dest => dest.Customer, src => src.ConvertedCustomer);
        }
    }
}
```

Then register the mappings in `Program.cs`:

```csharp
using Mapster;
```

```csharp
TypeAdapterConfig.GlobalSettings.Scan(Assembly.GetExecutingAssembly());
```

This allows Mapster to discover every `IRegister` implementation.

---

# 10. Step 8 — Create the Repository Query

The Response DTO determines what the repository needs to load.

If the Response DTO contains:

```text
Customer
Pipeline
Stage
AssignedUser
```

the repository should contain:

```csharp
private IQueryable<Opportunity> OpportunityQuery() =>
    _context.Opportunities
        .Include(x => x.Customer)
        .Include(x => x.Pipeline)
        .Include(x => x.Stage)
        .Include(x => x.AssignedUser);
```

This is the critical relationship:

```text
Response DTO
      ↓
Related properties
      ↓
Repository Includes
```

---

# 11. Use a Reusable Query Method

Do not duplicate your Includes.

Instead of:

```csharp
GetAll()
    → Include Customer
    → Include Pipeline
    → Include Stage

GetById()
    → Include Customer
    → Include Pipeline
    → Include Stage
```

create:

```csharp
private IQueryable<Opportunity> OpportunityQuery() =>
    _context.Opportunities
        .Include(x => x.Customer)
        .Include(x => x.Pipeline)
        .Include(x => x.Stage)
        .Include(x => x.AssignedUser);
```

Then:

```csharp
public async Task<IEnumerable<Opportunity>> GetAllOpportunity(Guid organizationId)
{
    return await OpportunityQuery()
        .Where(x => x.OrganizationId == organizationId)
        .ToListAsync();
}
```

And:

```csharp
public async Task<Opportunity?> GetOpportunityById(Guid id, Guid organizationId)
{
    return await OpportunityQuery()
        .FirstOrDefaultAsync(x => x.Id == id && x.OrganizationId == organizationId);
}
```

This keeps both endpoints consistent.

---

# 12. The Include Checklist

Whenever you create a Response DTO, immediately create this checklist.

Example:

```text
OpportunityResponseDto
```

| Response Property | Entity Navigation | Repository Include              |
| ----------------- | ----------------- | ------------------------------- |
| Customer          | Customer          | `.Include(x => x.Customer)`     |
| Pipeline          | Pipeline          | `.Include(x => x.Pipeline)`     |
| Stage             | Stage             | `.Include(x => x.Stage)`        |
| AssignedUser      | AssignedUser      | `.Include(x => x.AssignedUser)` |

Another example:

```text
ActivityResponseDto
```

| Response Property | Entity Navigation | Repository Include             |
| ----------------- | ----------------- | ------------------------------ |
| User              | User              | `.Include(x => x.User)`        |
| Customer          | Customer          | `.Include(x => x.Customer)`    |
| Lead              | Lead              | `.Include(x => x.Lead)`        |
| Opportunity       | Opportunity       | `.Include(x => x.Opportunity)` |

This prevents forgotten Includes.

---

# 13. Step 9 — Create and Update

When creating an entity, the newly created object normally contains FK IDs but not populated navigation properties.

Avoid:

```csharp
await _repository.Create(entity);

return entity.Adapt<ResponseDto>();
```

Instead:

```csharp
await _repository.Create(entity);

var created = await _repository.GetById(entity.Id, organizationId);

return created!.Adapt<ResponseDto>();
```

The flow becomes:

```text
POST
 ↓
Create entity
 ↓
Save
 ↓
Get entity with Includes
 ↓
Map
 ↓
Response DTO
```

---

# 14. Update Pattern

Use the same pattern for updates.

```csharp
await _repository.Update(existingEntity);

var updated = await _repository.GetById(id, organizationId);

return updated!.Adapt<ResponseDto>();
```

This ensures the response contains the same related objects as a normal GET request.

For example:

```text
PUT /opportunities/{id}
```

and:

```text
GET /opportunities/{id}
```

should return a consistent response structure.

---

# 15. EF Core Tracking

If you retrieve an entity through EF Core:

```csharp
var existing = await _repository.GetById(id, organizationId);
```

the entity is normally already tracked by the DbContext.

Therefore, after changing properties:

```csharp
existing.Name = dto.Name;
existing.Value = dto.Value;
```

you can generally save the changes without unnecessarily calling `Update()` on an already-tracked graph.

If your repository supports detached entities, use:

```csharp
if (_context.Entry(entity).State == EntityState.Detached)
    _context.Entities.Update(entity);
```

This avoids unnecessarily marking an entire loaded object graph as modified.

---

# 16. Step 10 — Service Layer

The service should coordinate the workflow.

Example:

```csharp
public async Task<OpportunityResponseDto> UpdateOpportunity(
    Guid id,
    OpportunityUpdateDto dto)
{
    var existing = await _repository.GetOpportunityById(
        id,
        _currentUser.OrganizationId);

    if (existing == null)
        throw new Exception("Opportunity not found.");

    existing.CustomerId = dto.CustomerId;
    existing.PipelineId = dto.PipelineId;
    existing.StageId = dto.StageId;
    existing.AssignedUserId = dto.AssignedUserId;
    existing.Name = dto.Name;
    existing.Value = dto.Value;

    await _repository.UpdateOpportunity(existing);

    var updated = await _repository.GetOpportunityById(
        id,
        _currentUser.OrganizationId);

    return updated!.Adapt<OpportunityResponseDto>();
}
```

The service should not manually construct the nested DTO unless there is a specific reason.

---

# 17. Step 11 — Controller

The controller should remain thin.

Example:

```csharp
[HttpGet]
public async Task<IActionResult> GetAll()
{
    var result = await _service.GetAllOpportunity();

    return Success(
        "Opportunities retrieved successfully.",
        result);
}
```

The controller should not be responsible for:

* EF Core Includes
* Mapster configuration
* Loading navigation properties
* Building complex DTOs

Those belong in the appropriate layers.

---

# 18. Step 12 — Swagger Testing

Before starting the frontend, test the API.

For GET:

```http
GET /api/opportunities
```

Verify that the response contains:

```json
{
    "id": "...",
    "customer": {
        "id": "...",
        "companyName": "ABC Corporation"
    },
    "pipeline": {
        "id": "...",
        "name": "Sales Pipeline"
    },
    "stage": {
        "id": "...",
        "name": "Negotiation"
    },
    "assignedUser": {
        "id": "...",
        "firstName": "John",
        "lastName": "Doe"
    }
}
```

If you get:

```json
{
    "customer": null,
    "pipeline": null,
    "stage": null,
    "assignedUser": null
}
```

do not move to the frontend yet.

Check:

```text
1. Does the entity have the navigation property?
2. Does the repository have Include()?
3. Is the correct entity being queried?
4. Is Mapster mapping the property?
5. Are the related database records actually present?
```

---

# 19. Common Problem: DTO Changed but Repository Didn't

Bad workflow:

```text
Entity
    ↓
Repository
    ↓
Response DTO changed later
    ↓
Nested properties are null
```

Example:

```csharp
public CustomerMinimalDto Customer { get; set; }
```

but repository:

```csharp
_context.Opportunities
    .ToListAsync();
```

EF Core did not load Customer.

Correct:

```csharp
_context.Opportunities
    .Include(x => x.Customer)
    .ToListAsync();
```

---

# 20. Common Problem: Entity and DTO Names Differ

Entity:

```csharp
public Customer? ConvertedCustomer { get; set; }
```

DTO:

```csharp
public CustomerMinimalDto? Customer { get; set; }
```

Mapster cannot assume that:

```text
ConvertedCustomer → Customer
```

is intentional.

Use:

```csharp
.Map(dest => dest.Customer, src => src.ConvertedCustomer);
```

---

# 21. Common Problem: Create Response Doesn't Have Related Objects

Bad:

```csharp
await _repository.Create(entity);

return entity.Adapt<ResponseDto>();
```

Possible result:

```json
{
    "customer": null,
    "pipeline": null,
    "stage": null
}
```

Correct:

```csharp
await _repository.Create(entity);

var created = await _repository.GetById(entity.Id, organizationId);

return created!.Adapt<ResponseDto>();
```

---

# 22. Common Problem: Update Response Doesn't Have Related Objects

Bad:

```csharp
await _repository.Update(entity);

return entity.Adapt<ResponseDto>();
```

Correct:

```csharp
await _repository.Update(entity);

var updated = await _repository.GetById(id, organizationId);

return updated!.Adapt<ResponseDto>();
```

---

# 23. Recommended Project Checklist

Use this checklist for every new entity.

## Entity

```text
[ ] Entity created
[ ] Primary key created
[ ] Foreign keys created
[ ] Navigation properties created
[ ] EF relationships configured
```

## DTO

```text
[ ] Minimal DTO created
[ ] Create DTO created
[ ] Update DTO created
[ ] Response DTO created
[ ] Response relationships identified
```

## Mapping

```text
[ ] Entity → Response mapping created
[ ] Different property names explicitly mapped
[ ] Mapping registered with Mapster
```

## Repository

```text
[ ] Base query created
[ ] Required Include() added
[ ] GetAll uses base query
[ ] GetById uses base query
[ ] Organization filtering applied where required
```

## Service

```text
[ ] Create implemented
[ ] Create reloads entity before response
[ ] Update implemented
[ ] Update reloads entity before response
[ ] Delete implemented
[ ] Business rules implemented
```

## Controller

```text
[ ] GET all
[ ] GET by ID
[ ] POST
[ ] PUT/PATCH
[ ] DELETE
[ ] Proper status responses
```

## Testing

```text
[ ] GET tested in Swagger
[ ] GET by ID tested
[ ] POST tested
[ ] PUT/PATCH tested
[ ] DELETE tested
[ ] Related objects are populated
[ ] No unexpected null navigation properties
```

---

# 24. The "One Feature" Workflow

For every new feature, use this exact process:

```text
ENTITY
  ↓
Identify FKs + Navigation Properties
  ↓
MINIMAL DTOs
  ↓
CREATE / UPDATE DTOs
  ↓
RESPONSE DTO
  ↓
Write relationship checklist
  ↓
MAPSTER
  ↓
REPOSITORY QUERY
  ↓
GET ALL / GET BY ID
  ↓
SERVICE
  ↓
CREATE → SAVE → RELOAD → MAP
  ↓
UPDATE → SAVE → RELOAD → MAP
  ↓
CONTROLLER
  ↓
SWAGGER
  ↓
FRONTEND
```

---

# 25. Golden Rule

Whenever you add something to a Response DTO, immediately ask:

```text
"Where does this data come from?"
```

For example:

```csharp
public UserMinimalDto AssignedUser { get; set; }
```

Ask:

```text
Where does AssignedUser come from?
```

Answer:

```text
Entity:
ApplicationUser AssignedUser

Repository:
.Include(x => x.AssignedUser)

Mapping:
User → UserMinimalDto

Service:
Reload entity using the populated query

Response:
AssignedUser
```

Another example:

```csharp
public PipelineMinimalDto Pipeline { get; set; }
```

Immediately identify:

```text
Entity:
Pipeline Pipeline

Repository:
.Include(x => x.Pipeline)

Mapping:
Pipeline → PipelineMinimalDto

Service:
Reload after create/update

Response:
Pipeline
```

---

# Final Architecture

The complete relationship should always look like this:

```text
                     DATABASE
                         │
                         ▼
                     ENTITY
                         │
             ┌───────────┴───────────┐
             │                       │
          FK IDs              Navigation Properties
             │                       │
             │                       ▼
             │                    Include()
             │                       │
             ▼                       ▼
       Request DTO              EF Core Query
             │                       │
             │                       ▼
             │                 Loaded Entity
             │                       │
             │                       ▼
             │                    Mapster
             │                       │
             │                       ▼
             └──────────────► Response DTO
                                     │
                                     ▼
                                  FRONTEND
```

The most important rule to remember is:

> **If a Response DTO contains a related object, the corresponding Entity navigation property, EF Core `Include()`, and Mapster mapping must all be considered together.**

And:

> **Request DTOs identify related records with IDs; Response DTOs provide readable related objects.**

Following this workflow from the beginning will prevent most of the DTO/`Include()`/Mapster adjustments you had to make in the CRM project.
