# 7. Validation & Mapping

Goal: FluentValidation (input validation) and AutoMapper (object-to-object mapping) — the two libraries that quietly make the CQRS framework from chapter 5 possible without every handler hand-writing boilerplate.

## 7.1 FluentValidation

FluentValidation replaces scattered `if (string.IsNullOrEmpty(x)) throw ...` checks with declarative, composable rule chains, one class per thing being validated — the .NET equivalent of a `zod`/`yup`/`class-validator` schema, but expressed as fluent C# instead of a schema object.

```csharp
public class CreateUserSto { public string Email { get; set; } public string Password { get; set; } }

public class CreateUserValidator : AbstractValidator<CreateUserSto>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Email).NotNull().WithMessage("Missing email").EmailAddress().WithMessage("Invalid email format");
        RuleFor(x => x.Password).NotEmpty().MinimumLength(8);
    }
}

// running it
var result = new CreateUserValidator().Validate(sto);
if (!result.IsValid) { /* result.Errors is a list of ValidationFailure */ }
```

Real, more elaborate example from this codebase, including a **conditional rule** (`.When(...)` — only apply this rule if a predicate holds, exactly like a `zod` `.refine()` with a condition):

```csharp
public class RegisterCompanyAccountsValidator : AbstractValidator<RegisterCompanyAccountQuery>
{
    public RegisterCompanyAccountsValidator()
    {
        RuleFor(x => x.PropsFilter.Email).NotNull().WithMessage("Missing Email address");
        RuleFor(x => x.PropsFilter.Email).EmailAddress().WithMessage("Wrong Email address format");
        RuleFor(x => x.PropsFilter.AccountName).NotNull().WithMessage("Missing account name").NotEmpty();
        RuleFor(x => x.PropsFilter.Password).NotNull().WithMessage("Missing password").NotEmpty();
        RuleFor(x => x.PropsFilter)
            .Must(p => p.Password == p.RepeatPassword)
            .When(x => !x.PropsFilter.EmailExists)
            .WithMessage("Passwords do not match");
    }
}
```
(`HRNS.Application/CQRS/CompanyAccounts/RegisterCompanyAccountQuery.cs:700-726`, condensed.)

### Where validators plug into the pipeline

Every `BaseHandler<TRequest, TResponse>` holds a `List<IValidator<TRequest>>` (`_validators`, `CQRS/Base/BaseHandler.cs:88`), and both `QueryBaseHandler`/`CommandBaseHandler` constructors automatically add a generic base validator on top of whatever feature-specific one exists:

```csharp
public QueryBaseHandler(...) : base(...)
{
    var validator = new QueryBaseValidator<TDTOModel, TPropsFilter>();
    _validators.Add(validator);
}
```

The base command validator (`CommandBaseValidator<TSTOModel>`, `CQRS/Base/CommandBase.cs:331-347`) is itself a good example of `RuleForEach` — validating every item in a collection with shared rules and nested per-item rules (`ChildRules`):

```csharp
public class CommandBaseValidator<TSTOModel> : AbstractValidator<CommandBase<TSTOModel>> where TSTOModel : STOModel
{
    public CommandBaseValidator()
    {
        RuleFor(x => x.Items).NotNull().NotEmpty().WithMessage("No items to save");
        RuleForEach(x => x.Items)
            .NotNull().WithMessage("Item to save is null")
            .ChildRules(items => { items.RuleFor(x => x.Id).NotEmpty().WithMessage("Missing entry ID"); });
    }
}
```

So a concrete command like `UpsertTicketingTicketTypeCommand` gets **two validators for free** just by inheriting from `CommandBase<TicketingTicketTypeSTO>`/`CommandBaseHandler<...>` — the generic "items aren't empty, every item has an ID" check, plus (if the feature needs anything more specific) its own feature-level validator added the same way. `FluentValidation.DependencyInjectionExtensions` (in the `.csproj`) also allows scanning-and-registering validators through DI (`services.AddValidatorsFromAssembly(...)`), an alternative to manual `_validators.Add(...)` you'll see used for validators consumed outside the CQRS pipeline (e.g. directly in a controller via `[ApiController]`'s automatic model validation).

## 7.2 AutoMapper

AutoMapper eliminates hand-written "copy field A to field B, field C to field D, ..." mapping code between two object shapes — the closest Node/TS analogue is a mapping library like `class-transformer` or, more honestly, a hand-rolled `mapUserToDto(user)` function, except AutoMapper generates that function for you from a declarative **profile**, by matching property names automatically and letting you override the exceptions.

```csharp
public class UserProfile : Profile
{
    public UserProfile()
    {
        CreateMap<UserEntity, UserDto>(); // same-named properties map automatically

        CreateMap<UserEntity, UserSummaryDto>()
            .ForMember(dest => dest.FullName, opt => opt.MapFrom(src => src.FirstName + " " + src.LastName))
            .ForMember(dest => dest.PasswordHash, opt => opt.Ignore()); // never leak this into a DTO
    }
}

// usage — anywhere IMapper is injected
var dto = _mapper.Map<UserDto>(userEntity);
_mapper.Map(sto, existingEntity); // map ONTO an existing object (used for updates — see chapter 5 §5.4)
```

### The base profile every entity mapping inherits

Just like `Entity`/`EntityDbConfig<T>` from chapter 4, every entity's mapping profile builds on a shared base that maps the audit fields once:

```csharp
public abstract class EntityMappingProfile : Profile
{
    public EntityMappingProfile()
    {
        CreateMap<Entity, BaseModel>().IncludeAllDerived();
        CreateMap<Entity, RefModel>().IncludeAllDerived();
        CreateMap<Entity, DTOModel>().IncludeAllDerived();

        CreateMap<STOModel, Entity>().IncludeAllDerived().MaxDepth(0)
            .ForMember(dest => dest.IsConstant, cfg => cfg.Ignore())
            .ForMember(dest => dest.CreatedByUser, cfg => cfg.Ignore())
            .ForMember(dest => dest.CreatedAt, cfg => cfg.Ignore())
            .ForMember(dest => dest.ModifiedByUser, cfg => cfg.Ignore())
            .ForMember(dest => dest.ModifiedAt, cfg => cfg.Ignore());
    }
}
```
(`HRNS.Database/Entities/Entity/EntityMappingProfile.cs`, in full.)

`.IncludeAllDerived()` is the key line: it means this base mapping (`Entity` → `DTOModel`) also applies automatically to every *derived* pair (`TicketingTicketTypeEntity` → `TicketingTicketTypeDTO`), without each feature-specific profile having to repeat the audit-field mappings. Notice the `STOModel → Entity` direction explicitly **ignores** `CreatedAt`/`CreatedByUser`/`ModifiedAt`/`ModifiedByUser` — those are intentionally *not* trusted from client input; `CommandBaseHandler.SetAdditionalFieldsOnAdd`/`SetAdditionalFieldsOnUpdate` (chapter 5 §5.4) stamp them server-side instead, right after the mapping runs. That's a real, deliberate security boundary: a malicious or buggy client sending a forged `CreatedByUser` in the request body cannot make it into the database — the mapping layer throws that field away before the entity is ever saved.

A concrete, feature-specific profile then adds only what's genuinely feature-specific — relationships that need to resolve an ID from a nested object, or vice versa:

```csharp
public class TicketingTicketTypeEntityMappingProfile : EntityMappingProfile
{
    public TicketingTicketTypeEntityMappingProfile()
    {
        CreateMap<TicketingTicketTypeSTO, TicketingTicketTypeEntity>()
            .ForMember(e => e.Contract, opt => opt.Ignore())
            .ForMember(e => e.DefaultAssignmentTeam, opt => opt.Ignore())
            .AfterMap((src, dest) =>
            {
                if (src.Contract != null) dest.ContractId = src.Contract.Id;
                if (src.DefaultAssignmentTeam != null) dest.DefaultAssignmentTeamId = src.DefaultAssignmentTeam.Id;
            });

        CreateMap<TicketingTicketTypeEntity, TicketingTicketTypeDTO>()
            .ForMember(dest => dest.Contract, opt => opt.MapFrom(src => src.Contract))
            .ForMember(dest => dest.DefaultAssignmentTeam, opt => opt.MapFrom(src => src.DefaultAssignmentTeam));
    }
}
```
(`HRNS.Database/Entities/Ticketing/TicketingTicketType/TicketingTicketTypeEntityMappingProfile.cs`, in full.) The pattern here — ignore the navigation property on the way in (STO → Entity), then resolve just its `Id` in `AfterMap` — exists because the incoming STO only carries a lightweight reference (`Contract.Id`), not a full `Contract` object EF Core could attach directly; forcing AutoMapper to "just assign" a detached, partially-populated navigation object would confuse EF Core's change tracker.

### How profiles get discovered

Every `Profile` subclass across the whole solution is picked up automatically — `services.AddAutoMapper(cfg => cfg.AddMaps(...))` in `HRNS.Application/DependencyInjection.cs` scans multiple assemblies for AutoMapper profiles the same way MediatR scans for handlers (chapter 3 §3.3). You never register a mapping profile by hand; you just create the class, and as long as it inherits from `Profile` (or, for entities, `EntityMappingProfile`) and lives somewhere in one of the scanned assemblies, it's active.

## 7.3 Why both layers matter together

Validation answers "is this input well-formed and allowed?" *before* anything touches the database. Mapping answers "how does this well-formed input become a persisted row (or a persisted row become an API response)?" *after* validation passes and authorization (chapter 5 §5.4) has already run. Keeping them as two separate, composable concerns — rather than validating inline while mapping, or mapping inline while validating — is what lets `CommandBaseHandler.DoHandle()` stay a clean, linear sequence of named steps (chapter 5 §5.4) instead of one tangled method doing five things at once.

## Checkpoint

1. Find a validator in `HRNS.Application/CQRS/` that uses `.When(...)` for a conditional rule you haven't seen yet, and explain in one sentence what real-world business rule it's enforcing.
2. Find a mapping profile that overrides a *base* mapping's behavior for one specific field with `.ForMember(...)` — what field, and why does the default (same-name-property) mapping not work for it?
3. In `EntityMappingProfile`, why is `CreatedByUser`/`CreatedAt` ignored on the `STOModel → Entity` map but *not* ignored on the `Entity → DTOModel` map? (Hint: think about which direction is "trust the client" vs. "trust the database.")
