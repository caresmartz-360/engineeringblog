---
layout: default
title: "The Self-Registering Factory Pattern: Making Your Code Smarter"
date: 2025-12-13 10:00:00 +0530
categories: engineering design-patterns
---

You know the pain. Every time someone needs to add a new payment method, report format, or notification channel, they open the same factory file. That file becomes a bottleneck—not just for deployment, but for confidence.

Here's what happens:

1. A factory class with a hundred-line switch statement
2. Merge conflicts pile up because everyone touches the same code
3. Bugs slip through when someone forgets to update one branch
4. Fear of breaking existing code slows down new features
5. The factory becomes the most dangerous file in your codebase

The self-registering factory pattern solves this by flipping things around: instead of the factory knowing about every option, each option tells the factory "hey, I exist."

**The trade-off:** You lose the single list where everything lives, but you gain the ability to add new features without changing old code.

If you have ever been afraid to modify a core factory class, this pattern is worth understanding.

## What is a Factory Pattern?

Before we get to self-registering factories, let's understand the basic factory pattern.

Imagine you are building a payment system. You support credit cards, PayPal, and bank transfers. Each payment method works differently, but your code needs to create the right payment processor based on what the customer chooses.

A traditional factory looks like this:

```csharp
public class PaymentFactory
{
    public IPaymentProcessor Create(string type)
    {
        if (type == "creditcard")
            return new CreditCardProcessor();
        else if (type == "paypal")
            return new PayPalProcessor();
        else if (type == "banktransfer")
            return new BankTransferProcessor();
        else
            throw new Exception("Unknown payment type");
    }
}
```

This works, but there is a problem. Every time you add a new payment method, you need to go back and modify this factory class. That breaks a basic rule: you should be able to add new features without changing existing code.

## Enter the Self-Registering Factory

A self-registering factory solves this problem elegantly. Instead of the factory knowing about every possible type, each type registers itself with the factory when your application starts.

Think of it like a registration desk at a conference. Instead of the organizer maintaining a master list and manually checking everyone in, attendees register themselves when they arrive. The system stays flexible and the registration desk doesn't need to know about every possible attendee in advance.

## How Does It Work?

It happens in two steps:

### 1. A Central Registry

First, you create a registry that keeps track of all the payment methods:

```csharp
public class PaymentProcessorFactory
{
    // Warning: Not thread-safe. In production, use ConcurrentDictionary
    // or protect with locks during registration phase.
    private static Dictionary<string, Func<IPaymentProcessor>> _processors 
        = new Dictionary<string, Func<IPaymentProcessor>>();

    public static void Register(string type, Func<IPaymentProcessor> creator)
    {
        // Silently overwrites if key exists. Consider logging or throwing
        // if duplicate registrations indicate a configuration error.
        _processors[type] = creator;
    }

    public static IPaymentProcessor Create(string type)
    {
        if (_processors.ContainsKey(type))
            return _processors[type]();
        
        throw new Exception($"No processor registered for type: {type}");
    }
}
```

### 2. Self-Registration

Each payment processor registers itself when the class is loaded:

```csharp
public class CreditCardProcessor : IPaymentProcessor
{
    static CreditCardProcessor()
    {
        PaymentProcessorFactory.Register("creditcard", () => new CreditCardProcessor());
    }

    public void ProcessPayment(decimal amount)
    {
        // Credit card processing logic
    }
}

public class PayPalProcessor : IPaymentProcessor
{
    static PayPalProcessor()
    {
        PaymentProcessorFactory.Register("paypal", () => new PayPalProcessor());
    }

    public void ProcessPayment(decimal amount)
    {
        // PayPal processing logic
    }
}
```

Notice the static constructor? That runs automatically when the class is first used. Each payment method registers itself without anyone needing to know about it.

A word of caution: this automatic registration only happens when the class gets loaded into memory. If nothing in your startup code touches the class, it never registers. This can bite you—tests might work because the test uses the class, but production breaks because nothing does. You may need to explicitly load these classes at startup. Class loading order is not deterministic, so relying on this implicitly can lead to environment-specific bugs.

## Why This Helps

### 1. Add New Features Without Touching Old Code

Want to add cryptocurrency payments? Create a new class. It registers itself. Done.

### 2. Each Part Handles Its Own Job

Each payment processor owns its logic and registration. Clear boundaries make testing and debugging faster.

### 3. Makes Plugins Easy

Each plugin announces itself when it loads. The core application stays decoupled from what exists.

## Real World Use Cases

At Caresmartz360, we use this pattern where extensibility matters more than central control:

**Report Generators:** Adding a new format means dropping in a class, not modifying core rendering logic—faster onboarding, fewer conflicts.

**Notification Channels:** New integrations (WhatsApp, Slack) ship without touching existing channels—reduced coupling, safer deploys.

**EVV Integrations:** State-specific adapters register themselves—new compliance requirements don't ripple through the system.

## When Not To Use This

Before you refactor every factory in your codebase, understand the trade-offs:

**Small, stable sets:** If you have three payment methods that rarely change, a simple switch statement is clearer. The extra indirection just makes things harder to follow.

**When you need to see everything in one place:** If new engineers need to quickly see all options, or if compliance needs to audit what exists, having registrations scattered everywhere makes discovery harder.

**Speed-critical code:** Dictionaries and function calls add overhead. If your factory runs millions of times per second, measure the cost first.

**Already confusing systems:** If your code is already hard to follow, adding automatic registration that happens behind the scenes makes debugging worse.

## Things to Watch Out For

**Startup Timing:** Classes only register when they get loaded. If nothing uses the class at startup, it never registers. You might need to explicitly load these classes early.

**Thread Safety:** If multiple threads run during startup, protect your dictionary with locks or use `ConcurrentDictionary`. Race conditions are hard to debug.

**Testing:** Clear your registry between tests, or give each test its own factory instance.

**Finding What Exists:** You lose the single list of options. Make up for it with admin tools, good logging, or a way to list registered items at runtime.

## A More Advanced Version

In real systems, you often need to turn things on and off without deploying: feature flags, emergency shutoffs, customer-specific settings, or admin pages that show what is available. That is when you add extra information:

```csharp
public class PaymentProcessorFactory
{
    private static Dictionary<string, ProcessorRegistration> _processors 
        = new Dictionary<string, ProcessorRegistration>();

    public static void Register(string type, Func<IPaymentProcessor> creator, 
                                string description, bool isEnabled = true)
    {
        _processors[type] = new ProcessorRegistration
        {
            Creator = creator,
            Description = description,
            IsEnabled = isEnabled
        };
    }

    public static IPaymentProcessor Create(string type)
    {
        if (!_processors.ContainsKey(type))
            throw new Exception($"No processor registered for type: {type}");
            
        var registration = _processors[type];
        
        if (!registration.IsEnabled)
            throw new Exception($"Processor {type} is currently disabled");
            
        return registration.Creator();
    }

    public static IEnumerable<string> GetAvailableTypes()
    {
        return _processors.Where(p => p.Value.IsEnabled).Select(p => p.Key);
    }
}
```

The `IsEnabled` flag supports things like gradual rollouts, incident response, and per-environment configuration without deploying new code.

## The Bigger Picture

This pattern is about picking the right trade-off. You give up having everything in one place. You get the ability to add new things without changing old things. That trade pays off when your system grows by adding features, not by changing what already exists.

At Caresmartz360, this helps us add new integrations without touching core code. When a new state requires a different verification format, we add a class. When an agency wants a custom report, we add a generator. The core stays stable.

But we do not use it everywhere. Simple factories stay simple until they are not.

## Getting Started

1. Start with a factory that changes every sprint because new options keep coming
2. Verify those options do not depend on each other
3. Add the registry, migrate one option over, check it works
4. Migrate the rest one by one
5. Document how registration works

Begin with one high-churn factory. If it reduces friction, expand. If it complicates things, revert.


Good engineering is knowing when to use a pattern, not just how to build it.
