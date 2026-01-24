---
layout: default
title: "Beyond Performance: JSC vs Hermes Through a Security Lens (Part 1)"
date: 2026-01-17 10:00:00 +0530
categories: mobile-security react-native reverse-engineering
---

React Native is widely adopted for building cross-platform mobile applications.
Most discussions around it focus on **developer productivity and performance**.

What is rarely discussed is this:

> **Your choice of JavaScript engine directly affects how easy your app is to reverse engineer.**

In this series, we analyze **JavaScriptCore (JSC)** and **Hermes** from a **cybersecurity and mobile VAPT perspective**.

This first part lays the foundation:
- What JSC and Hermes are
- Why React Native needs a JS engine
- How engine choice impacts the attack surface
- How to reliably identify which engine an APK is using

---

## Why React Native Needs a JavaScript Engine

React Native applications do not compile JavaScript into native machine code. Instead, they:

1. Ship JavaScript logic inside the APK
2. Load that logic at runtime
3. Execute it using a JavaScript engine
4. Bridge JS calls to native Android APIs

Without a JavaScript engine:
- Business logic cannot execute
- UI state management fails
- Network requests and validation logic break

On Android, React Native primarily uses two engines:
- **JavaScriptCore (JSC)** – older / default
- **Hermes** – newer, optimized engine by Meta

---

## What Is JavaScriptCore (JSC)?

**JavaScriptCore (JSC)** is Apple’s JavaScript engine, also used in Safari.

From a **security standpoint**, JSC has a critical characteristic:

> **It executes plain JavaScript source code.**

### What This Means in Practice

- `index.android.bundle` contains **readable JavaScript**
- Code is usually **minified**, not compiled
- Entire application logic is shipped to the client

### Security Implication

This is **source-level exposure**. From a threat-modeling perspective, this is equivalent to shipping your client-side source code.

Attackers can easily see:
- API endpoints
- Feature flags
- Client-side validation logic
- Debug or test conditions
- Business rules


---

## What Is Hermes?

**Hermes** is a JavaScript engine built specifically for React Native.

Instead of shipping JavaScript source:
- JavaScript is **compiled into Hermes Bytecode (HBC)**
- This is VM bytecode (not native machine code), similar in spirit to Java bytecode

- Bytecode is executed by a custom virtual machine
- App startup time and memory usage improve

### From a Security Perspective

- Code is **not immediately readable**
- Reverse engineering requires decompilation
- Casual attackers are slowed down

**⚠️ Important:** Hermes is a **performance optimization**, not a security feature.


---

## How Engine Choice Affects the Attack Surface

In practice, this means JSC-based apps are often vulnerable to trivial static analysis (regex searches, logic patching), while Hermes requires bytecode-aware tooling and control-flow recovery.


| Aspect | JSC | Hermes |
|------|----|--------|
| Code format | Plain JavaScript | Bytecode |
| Readability | Very high | Medium |
| Reverse-engineering effort | Low | Moderate |
| Tooling required | Basic | Specialized |
| Security by design | ❌ No | ❌ No |

**Key takeaway:**
Hermes increases attacker effort, but **does not prevent reverse engineering**.

---

## Identifying the JavaScript Engine (The Correct Way)

A common mistake is assuming the engine based on the file name.

This is wrong.

Both JSC and Hermes often use:
`index.android.bundle`

The **only reliable method** is inspecting the file content.

---

> Note: JavaScript engine choice (JSC vs Hermes) is independent of React Native’s old vs new architecture.

### JSC – Example

```bash
┌──(kali㉿kali)-[~/Desktop/apk/universalcaregiver1/assets]
└─$ file index.android.bundle
index.android.bundle: React Native minified JavaScript, ASCII text, with very long lines (10980)
```
✅ **Confirmed: JSC**

![Screenshot of terminal output for JSC]({{ site.baseurl }}/assets/file_jse.png)


### Hermes – Example

```bash
┌──(kali㉿kali)-[~/Desktop/apk-env/agency1/assets]
└─$ file index.android.bundle
index.android.bundle: Hermes JavaScript bytecode, version 96
```
✅ **Confirmed: Hermes**

![Screenshot of terminal output for Hermes]({{ site.baseurl }}/assets/file_hermes.png)

### Why File Extensions Do Not Matter

React Native selects the JavaScript engine using an executor:
- `JSCExecutor`
- `HermesExecutor`

The executor decides how the file is interpreted, not the filename.

This is why:
- `.bundle` can be plain JS
- `.bundle` can also be Hermes bytecode

**Security rule:** Trust file content, not file extensions.

---

## What Comes Next

In Part 2, we move into the offensive security side:
- Reverse engineering workflows
- JSC vs Hermes tooling
- Why JSC apps are easier to hack
