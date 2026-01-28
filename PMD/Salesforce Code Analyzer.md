To bridge the gap between **Static Analysis (PMD/Salesforce Code Analyzer)** and **Generative AI (GitHub Copilot)**, you need to explicitly translate rigid XML rules into natural language instructions.

Here are the main PMD rules for Salesforce translated into **"System Instructions"** or **"Custom Instructions"** you can feed into GitHub Copilot to ensure generated code passes your scanner automatically.

### 1. Naming Conventions (The "Style" Rules)

PMD enforces strict casing to ensure readability. Copilot often defaults to generic Java styles or random casing unless instructed otherwise.

| PMD Rule (XML) | Logic | **Copilot Prompt Instruction** |
| --- | --- | --- |
| `ClassNamingConventions` | Classes must be `UpperCamelCase`. | "All Apex class names must use **UpperCamelCase**." |
| `MethodNamingConventions` | Methods must be `lowerCamelCase` and often implies no underscores. | "Method names must use **lowerCamelCase**. Do not use underscores in method names." |
| `LocalVariableNamingConventions` | Variables must be `lowerCamelCase`. | "Local variables and parameters must use **lowerCamelCase**." |
| `FieldNamingConventions` | specific prefixes (like `FINAL_` or `k_`). | "Constants (static final) must be **UPPER_SNAKE_CASE**. Instance variables must be **lowerCamelCase**." |

> **Copy-Paste Prompt:**
> "Adopt Salesforce Apex naming standards: Class names in UpperCamelCase, method and variable names in lowerCamelCase. Avoid underscores in method names."

---

### 2. Documentation (The "ApexDoc" Rules)

The `ApexDoc` rule is one of the most common failures in CI/CD pipelines. Copilot tends to write simple comments (`//`) instead of formal documentation.

| PMD Rule (XML) | Logic | **Copilot Prompt Instruction** |
| --- | --- | --- |
| `ApexDoc` | Requires formal JavaDoc-style comments. | "Generate **ApexDoc** for every class and method." |
| *(Property)* `reportMissingDescription` | The `@description` tag is mandatory. | "The `@description` tag is **mandatory** for all ApexDoc headers." |
| *(Property)* `reportMissingParam` | Every method parameter must have a `@param`. | "Include `@param` for every input parameter explaining its purpose." |
| *(Property)* `reportMissingReturn` | Non-void methods must have `@return`. | "Include `@return` for non-void methods." |

> **Copy-Paste Prompt:**
> "Strictly enforce ApexDoc standards. Every class and method must have an ApexDoc header containing a `@description`. All methods must document every `@param` and the `@return` value if applicable."

---

### 3. Security (The "Trust" Rules)

These are critical. If Copilot generates insecure code, PMD (and the Security Review) will flag it immediately.

| PMD Rule (XML) | Logic | **Copilot Prompt Instruction** |
| --- | --- | --- |
| `ApexSharingViolations` | Classes must define sharing rules. | "Always declare `with sharing` or `inherited sharing` on classes. **Never** omit the sharing declaration." |
| `ApexSOQLInjection` | Prevents dynamic SOQL strings. | "Prevent SOQL injection: Always use **bind variables** (e.g., `:variableName`) instead of string concatenation." |
| `ApexCRUDViolation` | Checks field-level security. | "Before performing DML (Insert/Update), ensure you check `Schema.sObjectType` permissions or use `Security.stripInaccessible`." |

> **Copy-Paste Prompt:**
> "Security First: Always declare `with sharing` on classes. Never use string concatenation for SOQL; use bind variables. Check field-level security before any DML operation."

---

### 4. Code Structure & Performance (The "Quality" Rules)

These rules prevent "spaghetti code" and governor limit hits.

| PMD Rule (XML) | Logic | **Copilot Prompt Instruction** |
| --- | --- | --- |
| `AvoidSoqlInLoops` | No queries inside `for` loops. | "Bulkification: **NEVER** place a SOQL query or DML statement inside a loop. Collect IDs in a Set and query outside." |
| `IfStmtsMustUseBraces` | `if (x) return;` is forbidden. | "Always use curly braces `{}` for `if`, `for`, `while`, and `else` blocks, even for single lines." |
| `AvoidDeeplyNestedIfStmts` | Complexity check. | "Avoid deep nesting. Use **guard clauses** (early returns) to flatten logic complexity." |

---

### ⚡ The "Master Prompt" for GitHub Copilot

You can paste this entire block into your **Copilot Custom Instructions** (in VS Code settings) or at the start of a chat session to cover 90% of PMD violations:

```text
You are an expert Salesforce Developer using the SF Code Scanner (PMD) ruleset. 
When generating Apex code, you must strictly adhere to the following rules:

1. **Naming:** Use UpperCamelCase for classes and lowerCamelCase for methods/variables. No underscores in method names.
2. **Documentation:** Add ApexDoc to ALL classes and methods. You MUST include the @description tag, @param for all parameters, and @return for non-void methods.
3. **Security:** - Always define the class as 'with sharing' or 'inherited sharing'.
   - Use bind variables (e.g., :myVar) for SOQL. NEVER concatenate strings into queries.
   - Enforce FLS (Field Level Security) before DML.
4. **Performance:** NEVER put SOQL or DML inside loops. Use Sets and Maps for bulkification.
5. **Formatting:** Always use curly braces {} for control structures (if/else/for), even for single-line statements.

```

### Visual Aid: From PMD Failure to Success

---

**Would you like me to generate a specific `.ruleset.xml` file for you that matches these prompts so you can enforce them locally?**

... [Improve Developer Productivity with Github Copilot](https://www.youtube.com/watch?v=e4xIovF1lEQ)

I selected this video because it specifically demonstrates how to customize Copilot for Salesforce development, including generating unit tests and refactoring code to meet specific standards like the ones PMD enforces.
