---
layout: default
title: "Domain-Driven Design Building Blocks: A Practical Guide with C#"
date: 2026-01-17 10:00:00 +0530
categories: engineering architecture design-patterns
---

Domain-Driven Design (DDD) can sound intimidating, but it's really about one thing: organizing your code around your business problems, not your database tables or frameworks.

If you've ever opened a codebase and struggled to figure out where the business logic lives—buried in controllers, scattered across services, or mixed with database code—DDD offers a better way.

This post breaks down the core DDD building blocks with practical C# examples. No theory overload. Just the components you need to know and how to implement them.

## Why DDD Matters

Before we dive into components, let's address the "why."

Traditional layered architecture often leads to:
- Business logic scattered everywhere
- Data models that don't reflect business concepts
- Code that's hard to change when business rules evolve
- Developers who can't explain the code without a database diagram

DDD solves this by putting your business domain at the center. Your code speaks the language of your business stakeholders. When a product manager says "aggregate," "entity," or "value object," developers know exactly what they mean.

## The Core Building Blocks

### 1. Entities

**What they are:** Objects that have a unique identity that persists over time. Two entities are the same if they have the same ID, even if their properties differ.

**Simple rule:** If you need to track something over time and distinguish it from other similar things, it's an entity.

**Example:** A caregiver, a care recipient, a visit, a shift.

```csharp
// Base entity that all entities inherit from
public abstract class Entity<TId> where TId : notnull
{
    public TId Id { get; protected set; }
    
    protected Entity(TId id) => Id = id;
    
    public override bool Equals(object obj)
    {
        if (obj is not Entity<TId> other)
            return false;
            
        if (ReferenceEquals(this, other))
            return true;
        
        // Don't compare transient entities (those without persisted IDs)
        if (IsTransient() || other.IsTransient())
            return false;
            
        // Entities are equal if they have the same ID
        return Id.Equals(other.Id);
    }
    
    // Virtual to allow derived classes to define what "transient" means
    protected virtual bool IsTransient()
    {
        // For Guid-based IDs, check for empty
        if (Id is Guid guidId)
            return guidId == Guid.Empty;
            
        return Id.Equals(default(TId));
    }
    
    public override int GetHashCode() => Id.GetHashCode();
    
    public static bool operator ==(Entity<TId> left, Entity<TId> right) =>
        left?.Equals(right) ?? right is null;
    
    public static bool operator !=(Entity<TId> left, Entity<TId> right) =>
        !(left == right);
}

public class CareVisit : Entity<Guid>
{
    public string VisitNumber { get; private set; }
    public DateTime ScheduledStartTime { get; private set; }
    public VisitStatus Status { get; private set; }
    
    private readonly List<CareTask> _careTasks = new();
    public IReadOnlyCollection<CareTask> CareTasks => _careTasks.AsReadOnly();
    
    public CareVisit(string visitNumber, DateTime scheduledStart) 
        : base(Guid.NewGuid())
    {
        VisitNumber = visitNumber;
        ScheduledStartTime = scheduledStart;
        Status = VisitStatus.Scheduled;
    }
    
    public void AddTask(string serviceName, string notes)
    {
        if (Status == VisitStatus.Completed)
            throw new InvalidOperationException("Cannot modify completed visit");
            
        _careTasks.Add(new CareTask(serviceName, notes));
    }
    
    public void CompleteVisit()
    {
        if (Status != VisitStatus.InProgress)
            throw new InvalidOperationException("Only in-progress visits can be completed");
            
        Status = VisitStatus.Completed;
    }
}

public enum VisitStatus
{
    Scheduled,
    InProgress,
    Completed,
    Cancelled
}
```

**Key points:**
- ID never changes
- Private setters protect invariants
- Business methods (AddLine, Submit) enforce rules
- State changes happen through methods, not property setters

### 2. Value Objects

**What they are:** Objects without identity. Two value objects are the same if all their properties match.

**Simple rule:** If you only care about the values and don't need to track it over time, it's a value object.

**Example:** An address, a time range, a billing rate, vitals measurement.

```csharp
// Using C# record types for value objects - immutability and value equality built-in
public record TimeRange
{
    public DateTime StartTime { get; init; }
    public DateTime EndTime { get; init; }
    public TimeSpan Duration => EndTime - StartTime;
    
    public TimeRange(DateTime startTime, DateTime endTime)
    {
        if (endTime <= startTime)
            throw new ArgumentException("End time must be after start time");
            
        StartTime = startTime;
        EndTime = endTime;
    }
    
    public bool Overlaps(TimeRange other) =>
        StartTime < other.EndTime && other.StartTime < EndTime;
}

public record BillingRate
{
    public decimal HourlyRate { get; init; }
    public string ServiceType { get; init; }
    
    public BillingRate(decimal hourlyRate, string serviceType)
    {
        if (hourlyRate <= 0)
            throw new ArgumentException("Hourly rate must be positive");
            
        HourlyRate = hourlyRate;
        ServiceType = serviceType ?? throw new ArgumentNullException(nameof(serviceType));
    }
    
    public decimal CalculateCost(TimeSpan duration) => HourlyRate * (decimal)duration.TotalHours;
}

// For simple value objects, use record struct for better performance
public readonly record struct Address
{
    public string Street { get; init; }
    public string City { get; init; }
    public string ZipCode { get; init; }
    
    public Address(string street, string city, string zipCode)
    {
        Street = street ?? throw new ArgumentNullException(nameof(street));
        City = city ?? throw new ArgumentNullException(nameof(city));
        ZipCode = zipCode ?? throw new ArgumentNullException(nameof(zipCode));
    }
}
```

**Key points:**
- No ID property
- Immutable (no setters)
- Value equality (override Equals and GetHashCode)
- Validation in constructor
- Operations return new instances

### 3. Aggregates

**What they are:** A cluster of entities and value objects treated as a single unit. One entity is the "root" that controls access to everything inside.

**Simple rule:** Think of it as a consistency boundary. Everything inside an aggregate must be consistent at all times. Changes to anything inside go through the root.

**Example:** A Shift (root) containing CareVisits and CaregiverAssignments.

```csharp
// Shift is our aggregate root - it's the consistency boundary
// We keep Shift and its assignments together because:
// - Cost must be consistent with assignments
// - Status changes affect all assignments
// - We always load/save them as one unit
public class Shift : Entity<ShiftId>
{
    public string ShiftNumber { get; private set; }
    public CareRecipientId CareRecipientId { get; private set; }
    public TimeRange ScheduledTime { get; private set; }
    public ShiftStatus Status { get; private set; }
    
    private readonly List<CaregiverAssignment> _assignments = new();
    public IReadOnlyCollection<CaregiverAssignment> Assignments => _assignments.AsReadOnly();
    
    private readonly List<DomainEvent> _domainEvents = new();
    public IReadOnlyCollection<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    // All cost calculation happens in the aggregate root to maintain invariants
    public decimal TotalCost => _assignments.Sum(a => CalculateAssignmentCost(a));
    
    public Shift(string shiftNumber, CareRecipientId recipientId, TimeRange scheduledTime)
        : base(new ShiftId(Guid.NewGuid()))
    {
        ShiftNumber = shiftNumber;
        CareRecipientId = recipientId;
        ScheduledTime = scheduledTime;
        Status = ShiftStatus.Draft;
    }
    
    public void AssignCaregiver(CaregiverId caregiverId, BillingRate rate)
    {
        // Only Draft and Confirmed shifts can have caregivers assigned
        if (Status == ShiftStatus.Completed || Status == ShiftStatus.Cancelled)
            throw new InvalidOperationException("Cannot modify completed or cancelled shift");
            
        if (_assignments.Any(a => a.CaregiverId == caregiverId))
            throw new InvalidOperationException("Caregiver already assigned");
            
        _assignments.Add(new CaregiverAssignment(caregiverId, rate));
    }
    
    public void ConfirmShift()
    {
        if (!_assignments.Any())
            throw new InvalidOperationException("Cannot confirm shift without caregivers");
        
        if (Status != ShiftStatus.Draft)
            throw new InvalidOperationException("Only draft shifts can be confirmed");
            
        Status = ShiftStatus.Confirmed;
        _domainEvents.Add(new ShiftConfirmedEvent(Id.Value, CareRecipientId, 
            _assignments.Select(a => a.CaregiverId).ToList()));
    }
    
    public void StartShift()
    {
        if (Status != ShiftStatus.Confirmed)
            throw new InvalidOperationException("Only confirmed shifts can be started");
            
        Status = ShiftStatus.InProgress;
    }
    
    public void CompleteShift()
    {
        if (Status != ShiftStatus.InProgress)
            throw new InvalidOperationException("Only in-progress shifts can be completed");
            
        Status = ShiftStatus.Completed;
        _domainEvents.Add(new ShiftCompletedEvent(Id.Value));
    }
    
    public void ClearDomainEvents() => _domainEvents.Clear();
    
    // Invariant logic stays in the aggregate root
    private decimal CalculateAssignmentCost(CaregiverAssignment assignment) =>
        assignment.Rate.CalculateCost(ScheduledTime.Duration);
}

public enum ShiftStatus { Draft, Confirmed, InProgress, Completed, Cancelled }

// Child entity - no business logic, just data holder
public class CaregiverAssignment
{
    public CaregiverId CaregiverId { get; }
    public BillingRate Rate { get; }
    
    internal CaregiverAssignment(CaregiverId caregiverId, BillingRate rate)
    {
        CaregiverId = caregiverId;
        Rate = rate;
    }
}

// Strongly-typed IDs using record for value semantics
public record ShiftId(Guid Value);
public record CareRecipientId(Guid Value);
public record CaregiverId(Guid Value);
```

**Key points:**
- External code can only reference the aggregate root
- The root enforces all business rules
- All changes maintain consistency
- Child entities have internal constructors—only the root can create them
- One transaction per aggregate
- Cost calculation is an invariant protected by the aggregate

### 4. Domain Services

**What they are:** Operations that don't naturally belong to a single entity or value object. They involve multiple objects or external concerns.

**Simple rule:** If the operation involves coordination between multiple aggregates or doesn't fit neatly into one object, it's probably a domain service.

**Example:** Matching caregivers to shifts, calculating visit costs with insurance coverage, scheduling optimization.

```csharp
// Pure domain service - no repository dependencies
// Pass data in, get results out. Keeps domain layer clean.
public class CaregiverMatchingService
{
    public List<CaregiverMatch> FindBestMatches(
        List<Caregiver> availableCaregivers, 
        CareRecipient recipient, 
        Shift shift)
    {
        var matches = availableCaregivers
            .Select(caregiver => new CaregiverMatch(
                caregiver.Id,
                caregiver.FullName,
                CalculateScore(caregiver, recipient)))
            .Where(m => m.Score > 50)
            .OrderByDescending(m => m.Score)
            .ToList();
        
        return matches;
    }
    
    private int CalculateScore(Caregiver caregiver, CareRecipient recipient)
    {
        var score = 0;
        
        // Domain logic for matching
        if (caregiver.HasWorkedWith(recipient.Id)) score += 50;
        if (caregiver.Languages.Intersect(recipient.Languages).Any()) score += 30;
        score += (int)(caregiver.Rating * 10);
        
        return score;
    }
}

// Note: In practice, you might need repositories in domain services for complex scenarios.
// This is a trade-off. If you do, keep them behind domain-focused interfaces,
// not EF-specific ones.

public class CaregiverMatch
{
    public CaregiverId CaregiverId { get; }
    public string Name { get; }
    public int Score { get; }
    
    public CaregiverMatch(CaregiverId caregiverId, string name, int score)
    {
        CaregiverId = caregiverId;
        Name = name;
        Score = score;
    }
}

// Pure domain service - all data passed in
public class VisitBillingService
{
    public BillingBreakdown CalculateBilling(Shift shift, InsuranceCoverage coverage)
    {
        var totalCost = shift.TotalCost;
        var insurancePays = coverage.IsActive ? totalCost * coverage.Percentage / 100 : 0;
        var patientPays = totalCost - insurancePays;
        
        return new BillingBreakdown(totalCost, insurancePays, patientPays);
    }
}

public class BillingBreakdown
{
    public decimal TotalCost { get; }
    public decimal InsurancePays { get; }
    public decimal PatientPays { get; }
    
    public BillingBreakdown(decimal total, decimal insurance, decimal patient)
    {
        TotalCost = total;
        InsurancePays = insurance;
        PatientPays = patient;
    }
}
```

**Key points:**
- Stateless operations
- Named after the domain operation (not "OrderService" but "OrderPricingService")
- Coordinates between multiple aggregates
- Can interact with external systems

### 5. Repositories

**What they are:** Abstractions that provide access to aggregates. They handle loading and saving aggregates to storage.

**Simple rule:** Think of it as a collection of aggregates in memory. You get, add, or remove aggregates. The repository handles the database details.

```csharp
public interface IShiftRepository
{
    Task<Shift> GetByIdAsync(Guid id);
    Task<IEnumerable<Shift>> GetShiftsForCareRecipientAsync(CareRecipientId recipientId, DateTime date);
    void Add(Shift shift);  // Note: No SaveChanges here
    void Update(Shift shift);
}

public class ShiftRepository : IShiftRepository
{
    private readonly CareDbContext _context;
    
    public ShiftRepository(CareDbContext context) => _context = context;
    
    public async Task<Shift> GetByIdAsync(Guid id)
    {
        // Note: Using EF's Include is a pragmatic compromise.
        // Ideally, repositories hide all persistence details, but eager loading
        // is often necessary for performance. Keep this in infrastructure layer.
        return await _context.Shifts
            .Include(s => s.Assignments)
            .FirstOrDefaultAsync(s => s.Id == id);
    }
    
    public async Task<IEnumerable<Shift>> GetShiftsForCareRecipientAsync(
        CareRecipientId recipientId, DateTime date)
    {
        var startOfDay = date.Date;
        var endOfDay = startOfDay.AddDays(1);
        
        return await _context.Shifts
            .Include(s => s.Assignments)
            .Where(s => s.CareRecipientId == recipientId.Value 
                     && s.ScheduledTime.StartTime >= startOfDay 
                     && s.ScheduledTime.StartTime < endOfDay)
            .ToListAsync();
    }
    
    // No SaveChanges - let Unit of Work/Application layer control transactions
    public void Add(Shift shift) => _context.Shifts.Add(shift);
    
    public void Update(Shift shift) => _context.Shifts.Update(shift);
}

// Unit of Work pattern - controls transaction boundaries
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

public class EfUnitOfWork : IUnitOfWork
{
    private readonly CareDbContext _context;
    
    public EfUnitOfWork(CareDbContext context) => _context = context;
    
    public async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        return await _context.SaveChangesAsync(cancellationToken);
    }
}
```

**Key points:**
- One repository per aggregate root
- Methods return domain objects, not data transfer objects
- Hide all database/ORM details
- Repository interface lives in the domain layer
- Implementation lives in the infrastructure layer
- **Transaction boundaries controlled by Unit of Work, not repositories**
- Repositories register changes; Unit of Work commits them

### 6. Domain Events

**What they are:** Something significant that happened in your domain that other parts of the system might care about.

**Simple rule:** If you find yourself saying "when X happens, we also need to do Y," you probably need a domain event.

**Important:** Domain events are dispatched **after** the aggregate is persisted. Aggregates collect events during their lifecycle, but don't handle side effects themselves. This ensures consistency.

```csharp
public abstract class DomainEvent
{
    public DateTime OccurredOn { get; } = DateTime.UtcNow;
}

public class ShiftConfirmedEvent : DomainEvent
{
    public Guid ShiftId { get; }
    public CareRecipientId RecipientId { get; }
    public List<CaregiverId> AssignedCaregivers { get; }
    
    public ShiftConfirmedEvent(Guid shiftId, CareRecipientId recipientId, List<CaregiverId> caregivers)
    {
        ShiftId = shiftId;
        RecipientId = recipientId;
        AssignedCaregivers = caregivers;
    }
}

public class ShiftCompletedEvent : DomainEvent
{
    public Guid ShiftId { get; }
    public ShiftCompletedEvent(Guid shiftId) => ShiftId = shiftId;
}

// The Shift aggregate (defined earlier in Aggregates section) collects events:
// - ConfirmShift() adds ShiftConfirmedEvent
// - CompleteShift() adds ShiftCompletedEvent
// Events are stored in _domainEvents collection and dispatched after persistence.

// Event handler example
public class ShiftConfirmedEventHandler : IEventHandler<ShiftConfirmedEvent>
{
    private readonly INotificationService _notificationService;
    
    public ShiftConfirmedEventHandler(INotificationService notificationService)
    {
        _notificationService = notificationService;
    }
    
    public async Task Handle(ShiftConfirmedEvent @event)
    {
        foreach (var caregiverId in @event.AssignedCaregivers)
        {
            await _notificationService.NotifyCaregiverAsync(caregiverId, @event.ShiftId);
        }
    }
}

public interface IEventHandler<T> where T : DomainEvent
{
    Task Handle(T domainEvent);
}
```

**Key points:**
- Named in past tense (ShiftConfirmed, not ConfirmShift)
- Immutable
- Contain all information about what happened
- Raised by aggregates during state changes
- Handlers are separate from the aggregate
- **Dispatched after persistence to ensure consistency**
- **Outbox pattern**: For reliability, persist events in the same transaction as the aggregate. A separate process reads the outbox table and dispatches events. This ensures events are never lost even if the dispatcher fails.

```csharp
// Example outbox table
public class OutboxEvent
{
    public Guid Id { get; set; }
    public string EventType { get; set; }
    public string EventData { get; set; }  // Serialized JSON
    public DateTime OccurredOn { get; set; }
    public bool Dispatched { get; set; }
}
```

### 7. Factories

**What they are:** Objects responsible for creating complex aggregates or entities when the construction logic is complicated.

**Simple rule:** If creating an object involves multiple steps, validation, or complex logic, use a factory.

```csharp
public class ShiftFactory
{
    private readonly ICareRecipientRepository _recipientRepository;
    
    public ShiftFactory(ICareRecipientRepository recipientRepository)
    {
        _recipientRepository = recipientRepository;
    }
    
    public async Task<Shift> CreateShiftAsync(
        CareRecipientId recipientId, 
        DateTime scheduledDate)
    {
        var recipient = await _recipientRepository.GetByIdAsync(recipientId);
        
        if (recipient == null || !recipient.IsActive)
            throw new InvalidOperationException("Invalid care recipient");
        
        var shiftNumber = $"SH-{scheduledDate:yyyyMMdd}-{Guid.NewGuid().ToString("N")[..6].ToUpper()}";
        var timeRange = new TimeRange(
            scheduledDate.Date.AddHours(9),
            scheduledDate.Date.AddHours(13)
        );
        
        return new Shift(shiftNumber, recipientId, timeRange);
    }
    
    public async Task<Shift> CreateRecurringShiftAsync(Shift template, DateTime newDate)
    {
        var newShift = await CreateShiftAsync(template.CareRecipientId, newDate);
        
        // Copy assignments if caregivers are available
        foreach (var assignment in template.Assignments)
        {
            newShift.AssignCaregiver(assignment.CaregiverId, assignment.Rate);
        }
        
        return newShift;
    }
}
```

**Key points:**
- Encapsulate complex creation logic
- Validate preconditions before creation
- Keep constructors simple, move complexity to factories
- Can interact with repositories to fetch related data

## Putting It All Together

Here's how these components work together in a real application flow:

```csharp
public class ShiftApplicationService
{
    private readonly IShiftRepository _shiftRepository;
    private readonly ICareRecipientRepository _recipientRepository;
    private readonly ICaregiverRepository _caregiverRepository;
    private readonly ShiftFactory _shiftFactory;
    private readonly CaregiverMatchingService _matchingService;
    private readonly IUnitOfWork _unitOfWork;
    private readonly DomainEventDispatcher _eventDispatcher;
    
    public async Task<Guid> CreateAndConfirmShiftAsync(
        CareRecipientId recipientId, 
        DateTime scheduledDate)
    {
        // 1. Load necessary data
        var recipient = await _recipientRepository.GetByIdAsync(recipientId);
        if (recipient == null)
            throw new InvalidOperationException($"Care recipient {recipientId} not found");
        
        // 2. Use factory to create the aggregate
        var shift = await _shiftFactory.CreateShiftAsync(recipientId, scheduledDate);
        
        // 3. Use domain service with data fetched from infrastructure
        var availableCaregivers = await _caregiverRepository
            .GetAvailableCaregiversAsync(shift.ScheduledTime);
        
        var matches = _matchingService.FindBestMatches(
            availableCaregivers, 
            recipient, 
            shift);
        
        if (!matches.Any())
            throw new InvalidOperationException("No suitable caregivers available");
        
        // 4. Let aggregate make domain decisions
        // Note: Picking "First" is a policy decision. In production, this might be:
        // - A domain service: CaregiverSelectionPolicy.SelectBest(matches)
        // - Configuration: Settings.CaregiverSelectionStrategy
        // - User choice: Let coordinator pick from top 3
        var bestMatch = matches.First();
        var rate = new BillingRate(25.00m, "Personal Care");
        shift.AssignCaregiver(bestMatch.CaregiverId, rate);
        shift.ConfirmShift();
        
        // 5. Save aggregate
        _shiftRepository.Add(shift);
        
        // 6. Commit transaction
        await _unitOfWork.SaveChangesAsync();
        
        // 7. Dispatch domain events AFTER successful persistence
        foreach (var domainEvent in shift.DomainEvents)
            await _eventDispatcher.DispatchAsync(domainEvent);
        
        shift.ClearDomainEvents();
        return shift.Id;
    }
}
```

**Key architectural decisions:**
1. Application service orchestrates, but doesn't make domain decisions
2. Domain logic stays in aggregates and domain services
3. Transaction boundaries are explicit (Unit of Work)
4. Events dispatched after persistence ensures consistency
5. Infrastructure concerns (repository queries) stay in application layer

## Common Mistakes to Avoid

**1. Anemic Domain Models**
Don't create entities that are just bags of properties with no behavior. Business logic belongs in the domain, not in services.

**Bad:**
```csharp
public class Shift
{
    public Guid Id { get; set; }
    public ShiftStatus Status { get; set; }
    public List<CaregiverAssignment> Assignments { get; set; }
}

public class ShiftService
{
    public void ConfirmShift(Shift shift)
    {
        if (shift.Assignments.Count == 0)
            throw new Exception("Cannot confirm shift without caregivers");
        shift.Status = ShiftStatus.Confirmed;
    }
}
```

**Good:** Put the logic where it belongs—in the aggregate.

**2. Exposing Collections Without Protection**
```csharp
// Bad - external code can modify directly
public List<CaregiverAssignment> Assignments { get; set; }

// Good - readonly view, modifications through methods
private readonly List<CaregiverAssignment> _caregiverAssignments = new();
public IReadOnlyCollection<CaregiverAssignment> CaregiverAssignments => _caregiverAssignments.AsReadOnly();
```

**3. Violating Aggregate Boundaries**
Don't reference entities from one aggregate directly from another. Use IDs instead.

```csharp
// Bad - direct reference across aggregates
public class Shift
{
    public Caregiver Caregiver { get; set; }  // Wrong
}

// Good - reference by ID
public class Shift
{
    public CaregiverId CaregiverId { get; private set; }  // Correct
}
```

**4. Large Aggregates**
Keep aggregates small. If an aggregate has too many entities, split it into multiple aggregates.

## Bounded Contexts: The Big Picture

Before we wrap up, let's address how all these DDD components fit into larger systems.

### What is a Bounded Context?

A **Bounded Context** is a boundary within which a domain model is defined and applicable. It's where specific terms have precise meanings, and different contexts can have different models for the same concepts.

**Simple rule:** If the same word means different things to different teams, you probably need separate bounded contexts.

### Example in Home Care Software

In a system like CareSmartz or AlayaCare, you might have these bounded contexts:

```csharp
// Scheduling Context - cares about time and availability
namespace HomeCare.Scheduling
{
    public class Shift
    {
        public CaregiverId AssignedCaregiver { get; }
        public TimeRange ScheduledTime { get; }
        public ShiftStatus Status { get; }
        // Focus: when, who, conflicts
    }
}

// Billing Context - cares about money and insurance
namespace HomeCare.Billing
{
    public class BillableVisit
    {
        public decimal TotalAmount { get; }
        public InsuranceClaim Claim { get; }
        public PaymentStatus Status { get; }
        // Focus: cost, payment, claims
    }
}

// Care Management Context - cares about patient health and care plans
namespace HomeCare.CareManagement
{
    public class CareEpisode
    {
        public List<CareGoal> Goals { get; }
        public List<Assessment> Assessments { get; }
        public CareRecipientId RecipientId { get; }
        // Focus: health outcomes, care plans
    }
}
```

**The same "shift" concept exists in all three contexts, but the model is different:**
- **Scheduling** needs to know availability and conflicts
- **Billing** needs to know costs and insurance
- **Care Management** needs to know what services were provided

### Context Mapping

Contexts don't exist in isolation. They need to integrate:

```csharp
// Integration event - crosses context boundaries
public class ShiftCompletedIntegrationEvent
{
    public Guid ShiftId { get; }
    public DateTime CompletedAt { get; }
    public decimal Duration { get; }
}

// Scheduling context publishes
// Billing context subscribes to create invoice
// Care Management context subscribes to update care record
```

**Context Integration Patterns:**
- **Shared Kernel**: Small shared code between contexts (use sparingly)
- **Customer-Supplier**: One context depends on another's API
- **Anti-Corruption Layer**: Translate between contexts to keep your model clean
- **Published Language**: Use events/messages for loose coupling

### Practical Guidelines

1. **Start with one context**: Don't over-engineer. Split when you feel the pain.
2. **Team boundaries = context boundaries**: If separate teams own different areas, that's a natural split.
3. **Different change rates**: If billing rules change weekly but scheduling rarely changes, separate them.
4. **Different data needs**: If contexts need different persistence strategies, split them.

**Reality check:** Bounded contexts are messy in practice. You'll have overlapping concepts, temporary coupling during migrations, and teams that don't align perfectly with context boundaries. That's normal. DDD gives you a vocabulary to discuss and evolve these boundaries, not a perfect blueprint.

**When to split Shift into multiple contexts:****
- Scheduling team can't deploy without waiting for billing team → split
- The Shift class has 50 properties trying to serve everyone → split  
- Different parts have different SLAs or scaling needs → split

## Strategic Design vs Tactical Design

What we've covered so far (entities, aggregates, repositories) is **tactical DDD**—how to structure code within a bounded context.

**Strategic DDD** is about the big picture:
- Identifying bounded contexts
- Understanding how contexts relate
- Mapping out the domain landscape
- Deciding what to build vs. buy

Both matter. Tactical patterns without strategic thinking leads to a well-structured mess. Strategic thinking without tactical patterns leads to good intentions with poor execution.

## When NOT to Use DDD

DDD is powerful but not always necessary:
- Simple CRUD applications
- Small projects with straightforward logic
- Tight deadlines with simple requirements
- When your team is unfamiliar with DDD and doesn't have time to learn

Use DDD when:
- Complex business rules exist
- The domain logic is the core competitive advantage
- Multiple teams work on the same codebase
- Requirements change frequently

## Final Thoughts

Domain-Driven Design isn't about using every pattern all the time. It's about organizing code around business concepts so it's easier to understand, maintain, and evolve.

**For tactical patterns** (entities, aggregates, value objects):
1. Use base classes to enforce entity equality by ID
2. Prefer C# records for value objects—they communicate intent
3. Keep aggregates small and focused on consistency boundaries
4. Push all invariant logic into aggregate roots
5. Make domain services pure when possible
6. Control transactions explicitly with Unit of Work
7. Dispatch events after persistence

**For strategic design** (bounded contexts):
1. Start with one context, split when you feel the pain
2. Align contexts with team boundaries
3. Use integration events for loose coupling
4. Be explicit about context relationships

The goal is code that business stakeholders can read and developers can change confidently. If your code achieves that, you're on the right track.

Remember: DDD is a journey, not a destination. Start with the basics, learn from your mistakes, and refine as you go.

---

**Note:** This post focuses on tactical DDD patterns.
