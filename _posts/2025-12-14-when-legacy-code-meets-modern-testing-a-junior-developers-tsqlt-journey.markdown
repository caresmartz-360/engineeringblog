---
layout: default
title: "When Legacy Code Meets Modern Testing: A Junior Developer's tSQLt Journey"
date: 2025-12-27 00:00:00 +0530
categories: engineering
---

Testing stored procedures used to feel like trying to test a black box. You change something, run it manually against a dev database, check if the output "looks right," and hope nothing breaks in production.

Recently, I started using tSQLt, a unit testing framework for SQL Server, while working on some legacy database code. It's been a learning experience, and I wanted to share what I discovered about how automated testing can surface issues that might already be lurking in production.

---

## The Problem: Hidden Complexity in URL Parsing

I was working on `dbo.GetHHAConfig`, a stored procedure that retrieves agency configuration for our Tellus EVV integration. This procedure handles multiple configuration paths depending on whether we're fetching standard or additional EVV settings.

Buried in both the `@Additional = 0` and `@Additional = 1` branches was some complex URL parsing logic. The procedure needed to take a single stored URL field and split it into two values:

- **TokenAPIUrl**: The base URL (e.g., `https://prod-server.com`)
- **PostUrl**: The full API endpoint path

The actual logic in production looked like this:

```sql
-- For the TokenAPIUrl output
IIF(
  (CHARINDEX('/', TokenAPIUrl, 
    CHARINDEX('/', TokenAPIUrl, 
      CHARINDEX('/', TokenAPIUrl) + 1) + 1) - 1) >= 0,
  LEFT(TokenAPIURL, 
    CHARINDEX('/', TokenAPIUrl, 
      CHARINDEX('/', TokenAPIUrl, 
        CHARINDEX('/', TokenAPIUrl) + 1) + 1) - 1),
  TokenAPIUrl
) AS TokenAPIUrl,

-- For the PostUrl output
IIF(
  (CHARINDEX('/', TokenAPIUrl, 
    CHARINDEX('/', TokenAPIUrl, 
      CHARINDEX('/', TokenAPIUrl) + 1) + 1) - 1) >= 0,
  TokenAPIUrl,
  (TokenAPIUrl + '/api/v1')
) AS PostUrl
```

Nearly identical logic was repeated in the `@Additional = 1` branch using `ApiUrl` instead of `TokenAPIUrl`.

This code was solving a real problem. We had a single URL field in the database, but the EVV API workflow needed two different URL values for token authentication and batch posting.

The approach worked for the configurations that existed at the time. But it relied on undocumented assumptions about URL shape and expression evaluation order. As soon as real world variation appeared, those assumptions broke down.

That's often how legacy code evolves, solving immediate needs with the context and constraints available at the time.

---

## Enter tSQLt: Testing What We Couldn't See Before

A senior engineer suggested writing unit tests using tSQLt before making any changes to the procedure. The goal wasn't to criticize existing code. It was to create a safety net.

Setting up tSQLt was straightforward:

1. Install the framework in a test database
2. Use `tSQLt.FakeTable` to mock the configuration tables
3. Write test cases with known inputs and expected outputs
4. Run tests to verify behavior across different URL patterns

Here's what a basic test looked like:

```sql
EXEC tSQLt.NewTestClass 'GetHHAConfigTests';
GO

CREATE PROCEDURE GetHHAConfigTests.[test should parse standard https URL correctly]
AS
BEGIN
    -- Arrange
    EXEC tSQLt.FakeTable 'dbo.AgencyTellusConfiguration';
    
    INSERT INTO dbo.AgencyTellusConfiguration
    (AgencyID, OfficeId, TokenAPIUrl, EVVAggregator, IsActive, IsArchieved, Id)
    VALUES
    (1,
     'A1B2C3D4-E5F6-7890-ABCD-EF1234567890',
     'https://prod-server.com/api/token',
     '471777F6-93E7-46F0-8FB5-FC241D5589DF',
     1,
     0,
     80);
    
    -- Act
    DECLARE @Results TABLE (
        TokenAPIUrl NVARCHAR(255),
        BatchProcessUrl NVARCHAR(255),
        UserName NVARCHAR(255),
        Password NVARCHAR(255),
        ProviderTaxId NVARCHAR(255),
        ProviderNpi NVARCHAR(255),
        PostUrl NVARCHAR(255),
        StateId INT
    );
    
    INSERT INTO @Results
    EXEC dbo.GetHHAConfig
        @AgencyId = 1,
        @OfficeId = 'A1B2C3D4-E5F6-7890-ABCD-EF1234567890',
        @GetEVVConfiguration = 0,
        @Additional = 0;
    
    -- Assert
    EXEC tSQLt.AssertEqualsString
        @Expected = 'https://prod-server.com',
        @Actual   = (SELECT TokenAPIUrl FROM @Results);

    EXEC tSQLt.AssertEqualsString
        @Expected = 'https://prod-server.com/api/token',
        @Actual   = (SELECT PostUrl FROM @Results);
END;
GO
```

---

## What the Tests Revealed: A Runtime Failure

Once the tests were in place, something alarming happened immediately.

Before I even got to checking whether outputs were correct, the first test with a non-standard URL structure failed with this runtime error:

```
Conversion failed when converting the nvarchar value 'https://test/api' to data type int.
```

At first glance, this error made no sense. There is no explicit integer conversion anywhere in the stored procedure. But this turned out to be the most important clue.

### Why This Error Happens

The issue wasn't just bad string parsing. It was **implicit conversion caused by expression evaluation**.

Here's what's happening under the hood in SQL Server:

1. `CHARINDEX` returns `0` when the search string isn't found
2. The nested `CHARINDEX` calls perform arithmetic on those return values
3. `IIF()` evaluates both branches, not just the selected one
4. SQL Server attempts implicit conversions during expression evaluation
5. Result: string values are forced into integer context, causing conversion failures

This means the logic doesn't just produce wrong results. **It can completely fail at runtime depending on the URL shape**.

### Test Case 1: Simple URL Structure

```
Input:    'https://test/api'
Expected: TokenAPIUrl = 'https://test'
          PostUrl     = 'https://test/api'
```

**Result: Failed (runtime error)**

Instead of returning incorrect values, the procedure raised a conversion error. The logic attempted to locate a "third slash" using nested `CHARINDEX` calls. With a shorter URL, the calculation produced invalid positions, triggering implicit type coercion and failing at runtime.

### Test Case 2: URL with Port Number

```
Input:    'http://localhost:5000/api'
Expected: TokenAPIUrl = 'http://localhost:5000'
          PostUrl     = 'http://localhost:5000/api'
```

**Result: Failed**

The slash counting approach treated the port number as part of the path and split the URL at the wrong position, cutting it mid domain or mid port.

### Test Case 3: Subdomain URL

```
Input:    'https://api.staging.prod-server.com/v1/token'
Expected: TokenAPIUrl = 'https://api.staging.prod-server.com'
          PostUrl     = 'https://api.staging.prod-server.com/v1/token'
```

**Result: Failed**

The logic couldn't distinguish between domain structure and path segments and split inside the path instead of after the domain.

### Test Case 4: Fallback Logic Edge Case

```
Input:    'https://prod.server.com'
Expected: TokenAPIUrl = 'https://prod.server.com'
          PostUrl     = 'https://prod.server.com/api/v1'
```

**Result: Inconsistent behavior**

The `IIF` condition checks whether a `CHARINDEX` result is `>= 0`, which is always true, even when the substring isn't found. As a result, the fallback path was rarely triggered as intended.

**Here's the concerning part:** Some of these URL patterns already existed in our local and dev databases (`AgencyTellusConfiguration` and `EVVAdditionalConfiguration`). Different agencies had configured their integrations in different ways, and we hadn't realized the parsing logic could fail silently or catastrophically for certain configurations.

The tests didn't just reveal potential bugs. They exposed fundamental fragility in how the procedure handles URL variation.

**tSQLt didn't magically find the bug.** It forced the procedure to execute with realistic inputs, repeatedly and deterministically. That's what made the conversion error visible.

---

## The Real Value: Catching Production Issues Before Users Do

This experience highlighted something important about legacy code.

When code is written, you solve for the known cases. You test with the data you have. You validate against the scenarios you're aware of. But production is messy. Users configure systems in unexpected ways. Edge cases emerge over time.

Without an automated test suite when this procedure was originally written, there was no systematic way to validate these variations. The code worked for the configurations that existed at the time, and that was considered sufficient.

tSQLt gave us a way to interrogate the code with inputs we hadn't previously tested. It wasn't about proving anyone wrong. It was about discovering what the code actually does versus what we assumed it does.

---

## Beyond Correctness: Testing Query Performance

Once correctness issues were visible, performance problems became easier to notice as well.

While debugging failing tests, I noticed the procedure wasn't using indexes efficiently. Looking at the execution plan, I found this query:

```sql
SELECT ...
FROM AgencyTellusConfiguration
WHERE
-- AgencyId = @AgencyId AND  -- commented out
OfficeId = @OfficeId
AND EVVAggregator = '471777F6-93E7-46F0-8FB5-FC241D5589DF'
AND IsActive = 1
AND IsArchieved = 0
AND Id = 80
```

The `AgencyId` filter was commented out, preventing the use of an existing composite index on `(AgencyId, OfficeId)` and forcing a table scan. (It wasn’t clear whether this was deliberate or accidental, but it had real performance implications.)

You can extend tSQLt tests to act as early performance regression detectors. While they're not a replacement for proper load testing, they can catch obvious issues before they reach production.

---

## Lessons Learned

**Legacy code isn't bad code.** It's code written under different constraints.

**Tests aren't about blame.** They're about understanding behavior and creating confidence to improve.

**Production is the real test environment.** Unit tests help us approximate its messiness.

**Complex stored procedures need comprehensive coverage.** Multiple branches mean exponential test paths.

**Performance is a feature.** Correct but slow code still fails users.

**Implicit conversions are bugs waiting to happen.** Especially when combined with `IIF()` and nested expressions in SQL Server.

---

**Key Takeaways:**

- Automated testing can reveal bugs already in production
- Complex parsing logic is fragile as data patterns evolve
- Tests create safety nets for refactoring
- Stored procedures with many branches are hard to test manually but easy to test with tSQLt
- Performance regressions can be detected early
- `IIF()` evaluation combined with nested functions can trigger unexpected implicit conversions in SQL Server

---



