---
layout: default
title: "The Self-Registering Factory Pattern: Making Your Code Smarter"
date: 2025-12-13 10:00:00 +0530
categories: engineering design-patterns
---

When building software, one problem comes up often: you have different types of things that do similar jobs, and you need a clean way to pick which one to use. This is where the factory pattern helps. But today, we are going to talk about something even better—the self-registering factory pattern.

If you have ever maintained a big switch statement or endless if-else chains just to create objects, this pattern will make your life much easier.

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

This works, but there is a problem. Every time you add a new payment method, you need to go back and modify this factory class. That violates the Open/Closed Principle—your code should be open for extension but closed for modification.

## Enter the Self-Registering Factory

A self-registering factory solves this problem elegantly. Instead of the factory knowing about every possible type, each type registers itself with the factory when your application starts.

Think of it like a registration desk at a conference. Instead of the organizer maintaining a master list and manually checking everyone in, attendees register themselves when they arrive. The system stays flexible and the registration desk doesn't need to know about every possible attendee in advance.

## How Does It Work?

The magic happens in two parts:

### 1. A Central Registry

First, you create a registry that holds all available implementations:

```csharp
public class PaymentProcessorFactory
{
    private static Dictionary<string, Func<IPaymentProcessor>> _processors 
        = new Dictionary<string, Func<IPaymentProcessor>>();

    public static void Register(string type, Func<IPaymentProcessor> creator)
    {
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

Notice the static constructor? That runs automatically when the class is first accessed. Each processor registers itself without any central coordinator needing to know about it.

## Why This Pattern Matters

### 1. Add Features Without Breaking Existing Code

Want to add cryptocurrency payments? Just create a new class:

```csharp
public class CryptoProcessor : IPaymentProcessor
{
    static CryptoProcessor()
    {
        PaymentProcessorFactory.Register("crypto", () => new CryptoProcessor());
    }

    public void ProcessPayment(decimal amount)
    {
        // Crypto processing logic
    }
}
```

The factory doesn't need to change. The existing payment processors don't need to change. You just drop in a new class and it works.

### 2. Better Separation of Concerns

Each payment processor is responsible only for its own logic and its own registration. The factory just manages the registry. Clean boundaries mean easier testing and maintenance.

### 3. Plugin Architecture Made Easy

This pattern is perfect for building plugin systems. Each plugin can register itself when loaded, and your core application doesn't need to know what plugins exist.

## Real World Use Cases

At Caresmartz360, we use patterns like this in several places:

**Report Generators:** Different agencies need different report formats—PDF, Excel, CSV, custom templates. Each report generator registers itself, and when an agency requests a report, the factory creates the right generator without our core code needing to know about every possible format.

**Notification Channels:** We send notifications through email, SMS, push notifications, and in-app messages. Each channel implementation registers itself, making it trivial to add new channels like WhatsApp or Slack integrations.

**EVV Integrations:** Electronic Visit Verification systems vary by state and provider. Each EVV adapter registers itself with the factory, allowing us to support new states or providers without modifying existing integration code.

## Things to Watch Out For

While this pattern is powerful, there are some considerations:

**Initialization Timing:** Static constructors run the first time a class is referenced. If your classes are never referenced, they won't register. You might need to explicitly touch each class during application startup.

**Thread Safety:** If registrations happen during multi-threaded startup, protect your registry with proper locking.

**Testing:** In unit tests, you might need to reset your registry between tests to avoid state leaking from one test to another.

**Discovery:** Without a central list, it can be harder to see what implementations exist. Good documentation and naming conventions help here.

## A More Advanced Version

For production systems, you might want more features:

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

This version adds metadata, the ability to enable or disable processors at runtime, and a way to discover what's available.

## The Bigger Picture

The self-registering factory pattern is really about embracing the Open/Closed Principle. Your system becomes modular and extensible without becoming fragile.

Instead of a central coordinator that knows everything, you build a system where components announce their capabilities. This scales better, tests better, and adapts to change better.

At Caresmartz360, patterns like this help us move fast while keeping our codebase maintainable. When a new state requires a different EVV format, or when we need to support a new payment gateway, we add new classes rather than modify existing ones. That means less risk, faster delivery, and happier developers.

## Getting Started

If you want to try this pattern in your own projects:

1. Identify places where you have multiple implementations of the same interface
2. Look for switch statements or if-else chains that pick which implementation to use
3. Create a simple registry dictionary
4. Add static constructors to your implementations
5. Replace your switch statements with factory lookups

Start small. Pick one area and refactor it. You will quickly see the benefits.

## Wrapping Up

The self-registering factory pattern might sound complex, but it's actually quite simple once you see it in action. It's a tool that makes your code more flexible and easier to extend.

Good engineering is about choosing patterns that match your problems. When you need to support multiple implementations and want to avoid tightly coupled code, this pattern is a solid choice.

Your future self, and your teammates, will thank you for it.
