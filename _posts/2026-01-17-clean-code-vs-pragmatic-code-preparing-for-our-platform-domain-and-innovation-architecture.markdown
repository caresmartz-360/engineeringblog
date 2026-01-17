---
layout: default
title: "Clean Code vs Pragmatic Code: Preparing for Our Platform, Domain, and Innovation Architecture"
date: 2026-01-17 14:30:00 +0530
categories: engineering architecture clean-code
---

Early in my career, I assumed growth meant writing more code. Recently, I learned something different.

On a complex issue, I didn't touch the code at all. Instead, I observed senior engineers discuss trade-offs, risks, and long-term impact. Watching those decisions unfold taught me more than any commit could have.

It reminded me of two influential books:

- **Clean Code** – Robert C. Martin
- **The Pragmatic Programmer** – Andrew Hunt & David Thomas

What struck me is how relevant these philosophies become as our organization prepares to move toward a more layered architecture:

**Platform → Domain → Innovation**

We're not fully there yet, but the direction is clear.
And this shift changes how we should think about engineering decisions.

---

## The Architectural Direction We're Moving Toward

The proposed model introduces three layers:

### 1. Platform Layer (Foundational)

- Shared services
- Infrastructure
- Security
- Reliability

### 2. Domain Layer (Business Core)

- Workflows
- Business rules
- Orchestration

### 3. Innovation Layer (Velocity)

- Frontend
- Experiments
- UX
- Rapid iteration

Each layer serves a different business objective.
And that means each layer deserves a different engineering mindset.

```
[ Innovation Layer ]
        ↓
    Promote
        ↓
  [ Domain Layer ]
        ↓
    Harden
        ↓
 [ Platform Layer ]
```

---

## Platform Layer → Clean Code Discipline

Robert Martin says:

> "Clean code reads like well-written prose."

As we move toward a formal Platform layer, this becomes critical:

**Platform systems:**

- Live the longest
- Are reused across teams
- Carry high operational risk

Here, Clean Code should be non-negotiable.

```csharp
public decimal CalculateInvoiceTotal(Invoice invoice)
{
    var subtotal = CalculateSubtotal(invoice.Items);
    var tax = CalculateTax(subtotal);
    var discount = CalculateDiscount(invoice);

    return subtotal + tax - discount;
}
```

**Lesson:** Platform code should be self-documenting and built to last.

**Why this matters:**

- Predictable behavior
- Safe refactoring
- Easier onboarding
- **Lower long-term cost**

Observing platform discussions, one pattern becomes clear:

**Platform changes move slowly, reviews are deep, and shortcuts are discouraged.**

That's not bureaucracy, it's risk management.

---

## Innovation Layer → Pragmatic Execution

The Pragmatic Programmer teaches:

> "Don't build what you don't need."

The Innovation layer exists to:

- Test ideas
- Learn quickly
- Iterate with users

```csharp
public decimal CalculatePrice(List<Item> items)
{
    var subtotal = items.Sum(x => x.Price);
    return subtotal + (subtotal * 0.18m);
}
```

**Lesson:** In innovation, working code beats perfect code.

Is this perfect? No.
Is it useful? Yes.

And in innovation:

**Speed beats elegance.**

Over-engineering here:

- **Delays feedback**
- **Increases cost**
- **Reduces learning velocity**

---

## Domain Layer → Where Judgment Lives

The Domain layer is where:

- Business rules evolve
- Multiple teams collaborate
- Requirements change frequently

This is the balance zone.

Here, the guiding principle is:

**Domain code should express business language and resist infrastructure concerns.**

```csharp
public decimal CalculateOrderTotal(Order order)
{
    var subtotal = order.Items.Sum(i => i.Price);
    var tax = CalculateTax(subtotal);

    if (!order.HasDiscount)
        return subtotal + tax;

    return ApplyDiscount(subtotal, tax);
}
```

**Lesson:** Domain code speaks business language, not infrastructure language.

Notice:

- **Method names mirror business vocabulary**
- No mention of databases, APIs, or infrastructure
- Logic stays readable without over-engineering
- **Room to evolve as requirements change**

This is what I now think of as:

**Pragmatic Clean Code**  
Disciplined, but not rigid.  
**Optimized for change, not reuse.**

---

## Putting This Into Practice: A Small POC

To test this thinking, I built a small proof of concept, a simple order processing service with three modules representing each layer.

### The Platform Module

This handled core operations: logging, caching, and configuration.

```csharp
public interface ILoggingService
{
    void LogInfo(string message);
    void LogError(string message, Exception ex);
}

public class LoggingService : ILoggingService
{
    public void LogInfo(string message)
    {
        // Structured, predictable, testable
        Console.WriteLine($"[INFO] {DateTime.UtcNow}: {message}");
    }

    public void LogError(string message, Exception ex)
    {
        Console.WriteLine($"[ERROR] {DateTime.UtcNow}: {message} | Exception: {ex.Message}");
    }
}
```

**What I learned:** Even in a POC, writing clean interfaces for platform code felt right. It's code I'd trust to reuse.

### The Domain Module

This contained business logic: order validation and pricing rules.

```csharp
public class OrderService
{
    private readonly ILoggingService _logger;

    public OrderService(ILoggingService logger)
    {
        _logger = logger;
    }

    public decimal CalculateTotal(Order order)
    {
        _logger.LogInfo($"Calculating total for order {order.Id}");
        
        var subtotal = order.Items.Sum(i => i.Price * i.Quantity);
        var tax = subtotal * 0.18m;
        
        if (order.IsDiscountEligible)
            return ApplyDiscount(subtotal + tax, order.DiscountCode);
            
        return subtotal + tax;
    }

    private decimal ApplyDiscount(decimal total, string code)
    {
        // Room to grow, but clear enough for now
        return code == "SAVE10" ? total * 0.9m : total;
    }
}
```

**What I learned:** I didn't over abstract. The discount logic is simple, but I can extend it later without rewriting everything.

### The Innovation Module

This was a quick API endpoint to test the flow.

```csharp
app.MapPost("/api/orders/calculate", (OrderRequest req, OrderService svc) =>
{
    var order = new Order 
    { 
        Id = req.OrderId, 
        Items = req.Items,
        IsDiscountEligible = req.HasDiscount,
        DiscountCode = req.Code
    };
    
    var total = svc.CalculateTotal(order);
    return Results.Ok(new { Total = total });
});
```

**What I learned:** This layer doesn't need to be perfect. It just needs to work so I can test assumptions.

### The Outcome

This POC took about a hour to build and taught me more than weeks of reading:

- **Platform code benefits from discipline**
- **Domain code needs clarity, not complexity**
- **Innovation code ships fast and teaches you what matters**

The architecture wasn't theoretical anymore. **It was tangible.**

---

## An Example: Learning from Platform Foundations

I was asked to look into .NET Zero as a potential platform foundation. Exploring it through this three-layer lens helped me understand what makes a good platform.

Looking at the framework, I noticed:

- Clear separation of concerns
- Strong modular structure
- Opinionated conventions
- Well-defined extension points

**What you want from a Platform layer:**

- Projects are easy to navigate
- Responsibilities are clearly scoped
- Patterns are consistent
- Predictability and stability
- Long-term maintainability

The best platforms encourage good engineering behavior by default, which is exactly what infrastructure should do.

---

## Mapping Philosophy to Our Future Architecture

| Layer | Philosophy | Business Outcome |
|-------|-----------|------------------|
| Platform | Clean Code | Stability & scale |
| Domain | Balanced | Business agility |
| Innovation | Pragmatic | Speed & learning |

---

## What I'm Still Learning

This journey isn't finished. Here's what I'm grappling with:

**When does "pragmatic" become "technical debt"?**

In the Innovation layer, I've seen fast code become permanent code. The line between "good enough for now" and "we'll fix it later" is **thinner than I thought**.

### The Pragmatic Shortcut in My College Project

I learned this lesson during my major project in college. I was building an image classification model for plant disease detection. To meet the demo deadline, I hardcoded several assumptions:

```python
# Hardcoded image preprocessing (not actual project code just top of my head)
def preprocess_image(image_path):
    img = cv2.imread(image_path)
    img = cv2.resize(img, (224, 224))  # Hardcoded dimensions
    img = img / 255.0  # Hardcoded normalization
    return img

# Hardcoded class mapping
DISEASE_CLASSES = {
    0: "Healthy",
    1: "Bacterial Spot",
    2: "Late Blight"
}

def predict(image_path):
    processed = preprocess_image(image_path)
    prediction = model.predict(np.array([processed]))
    class_id = np.argmax(prediction)
    return DISEASE_CLASSES[class_id]
```

It worked perfectly during my practice runs.

### What Went Wrong

Then came the evaluation. The evaluator asked:

- "Can you show me predictions for these higher-resolution images?"
- "What if we want to add a new disease category?"

My model broke:

- **Higher resolution images crashed** the hardcoded 224x224 resize logic
- **Adding a new class required retraining** and manual code changes
- **No configuration layer** meant I couldn't adjust anything without editing code
### How I Should Have Done It
**What I should have done:**

```python
# Configuration-driven approach
class ModelConfig:
    def __init__(self, config_path):
        with open(config_path, 'r') as f:
            config = json.load(f)
        self.input_size = tuple(config['input_size'])
        self.normalization = config['normalization']
        self.class_mapping = config['classes']

def preprocess_image(image_path, config):
    img = cv2.imread(image_path)
    
    # Preserve aspect ratio while resizing
    h, w = img.shape[:2]
    target_size = config.input_size[0]
    scale = target_size / max(h, w)
    img = cv2.resize(img, (int(w * scale), int(h * scale)))
    
    # Pad to target size
    pad_h = (target_size - img.shape[0]) // 2
    pad_w = (target_size - img.shape[1]) // 2
    img = cv2.copyMakeBorder(img, pad_h, pad_h, pad_w, pad_w, 
                             cv2.BORDER_CONSTANT)
    
    # Configurable normalization
    if config.normalization == 'standard':
        img = img / 255.0
    elif config.normalization == 'imagenet':
        img = (img - [0.485, 0.456, 0.406]) / [0.229, 0.224, 0.225]
    
    return img
```

**Lesson:** Hardcoded assumptions can meet short-term goals but create long-term debt.

The "pragmatic" shortcuts that got me to the demo became **technical debt the moment flexibility was needed**.

> **Key Insight:**  
> The cost of "pragmatic" isn't always obvious upfront. Sometimes it shows up exactly when you can't afford it.

**How do you refactor across layers?**

Moving code from Innovation to Domain to Platform isn't just copy-paste. It's a mindset shift, and I'm still learning how to identify when that transition should happen.

**Who decides the boundaries?**

Is it the architect? The team lead? The person writing the code?

I don't have the answer yet, but I'm learning to ask the question.

---

## Final Reflection

Being kept in the loop, even without writing code, showed me something important:

- **Architecture shapes behavior**
- **Engineering is contextual**
- **Good decisions aren't binary**

But more than that: **building the POC made it real.**

Reading about these concepts is one thing. **Applying them, even in a small project, changed how I think about every line of code I write.**

Now, when I open a file, I ask:

*What layer does this belong to?*

And that question alone makes me a better engineer.

> **Key Takeaway:**  
> The real lesson wasn't Clean Code vs Pragmatic Code, it was learning **where each one belongs**.

---

## Closing Thought

Clean Code provides discipline.
Pragmatic Code provides velocity.
Architecture tells us where each belongs.

**And building something, even something small, teaches us how to decide.**
