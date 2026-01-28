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

---

To improve your guidelines for generating Apex code with GitHub Copilot while strictly adhering to PMD rules, you should shift from simple prompts to a **structured "System Instruction" approach**.

The most effective way to enforce these rules is to create a `.github/copilot-instructions.md` file in your repository. This file acts as a permanent context for Copilot, ensuring every suggestion it makes considers your PMD constraints without you needing to repeat them.

Here are the improved guidelines, broken down by strategy and specific rule implementations.

### 1. The Core Strategy: Use `copilot-instructions.md`

Create a file named `copilot-instructions.md` inside the `.github/` folder of your repository. Copilot will read this file before generating any code.

**Copy and paste the following content into that file.** It translates specific PMD rules into natural language instructions Copilot can understand.

```markdown
# Apex Generation Guidelines (PMD Compliant)

You are an expert Salesforce Apex developer. When generating code, you must strictly adhere to the following PMD static analysis rules.

## 1. Security & Sharing (Critical)
* **Enforce Sharing:** Every class must explicitly declare `with sharing` or `inherited sharing`. Never use `without sharing` unless explicitly requested for a system-level process. (Rule: `ApexSharingViolations`)
* **CRUD/FLS Checks:** Before any DML (Insert, Update, Delete) or SOQL query, you must check for field-level security and object permissions using `Schema.DescribeSObjectResult`. (Rule: `ApexCRUDViolation`)
* **Avoid SOQL Injection:** Always use variable binding (`:variableName`) in SOQL queries. Never use string concatenation to build dynamic queries. (Rule: `ApexSOQLInjection`)

## 2. Performance & Limits (Critical)
* **Bulkification:** All code must be bulk-safe. Triggers and methods must handle `List<SObject>` rather than single records.
* **No Loops for SOQL/DML:** NEVER perform DML statements or SOQL queries inside a `for` or `while` loop. Gather data in collections (Lists/Sets/Maps) and perform the operation once outside the loop. (Rules: `AvoidSoqlInLoops`, `AvoidDmlStatementsInLoops`)
* **Map Usage:** specific IDs should be collected in a `Set<Id>` or `List<Id>` and used in a single SOQL query with `WHERE Id IN :set`.

## 3. Error Handling & Reliability
* **No Empty Catch Blocks:** Never leave a `catch` block empty. At a minimum, log the error to the debug log or a custom error logging framework. (Rule: `EmptyCatchBlock`)
* **Avoid Hardcoding:** Never hardcode IDs (e.g., RecordTypeIds, ProfileIds). Use `Schema.DescribeSObjectResult` or Custom Metadata/Settings to retrieve them dynamically. (Rule: `AvoidHardcodingId`)
* **One Trigger Per Object:** Assume a "One Trigger Per Object" architecture using a handler class pattern.

## 4. Code Style & Maintainability
* **Naming Conventions:** * Classes: `PascalCase`
    * Methods/Variables: `camelCase`
    * Constants: `UPPER_SNAKE_CASE` (Rule: `ClassNamingConventions`, `MethodNamingConventions`)
* **Braces are Mandatory:** Always use braces `{}` for `if`, `else`, `for`, and `while` loops, even for single-line statements. (Rules: `IfStmtsMustUseBraces`, `ForLoopsMustUseBraces`)
* **Asserts in Tests:** All unit tests must contain at least one `System.assert` or `System.assertEquals`. (Rule: `ApexUnitTestClassShouldHaveAsserts`)

```

### 2. Interactive Prompting Strategy

When asking Copilot to generate code interactively, use specific "Trigger Phrases" that reference these categories.

**Instead of:**

> "Write a trigger to update the contact address when the account address changes."

**Use this (PMD-Optimized):**

> "Write a bulkified `Account` trigger handler method to update related `Contact` addresses. Ensure strict CRUD/FLS checks are performed before the update, and use a Map to avoid SOQL inside loops."

### 3. Review Checklist (The "Human in the Loop")

Even with improved guidelines, you should use this quick checklist to review Copilot's output against common PMD violations:

* [ ] **Class Declaration**: Does it say `with sharing`?
* [ ] **Loops**: Is there a `[` (query) or `insert`/`update` keyword *inside* a `{` `}` block of a loop? (Major violation).
* [ ] **Security**: Are variables in queries preceded by `:`? (e.g., `WHERE Name = :inputName`).
* [ ] **Hardcoding**: Do you see any 15 or 18-character strings (IDs) directly in the code?
* [ ] **Tests**: Did it generate `System.assert` statements? (Copilot often skips these).

### 4. Advanced: Use "Fix" Prompts for Existing Code

If you are refactoring legacy code, you can use Copilot to fix specific PMD issues by naming the rule directly:

* *"Refactor this method to fix the **AvoidDeeplyNestedIfStmts** PMD violation by using guard clauses."*
* *"Rewrite this loop to resolve the **AvoidSoqlInLoops** violation. Use a Map to pre-fetch the data."*

... [Improve Developer Productivity with Github Copilot](https://www.youtube.com/watch?v=e4xIovF1lEQ)

This video is relevant because it specifically covers Salesforce development with GitHub Copilot, demonstrating how to use it for Apex, LWC, and SOQL while maintaining productivity and best practices.

---

You are absolutely right. The previous guidelines touched on naming but missed the **strict enforcement** required by PMD's `ApexDoc` and specific naming pattern rules (like avoiding underscores).

For a truly "PMD-compliant" setup, you need to explicitly instruct Copilot to handle **ApexDoc headers** (`@description`) and enforce specific casing rules that standard Copilot suggestions often ignore.

Here is the updated, specific configuration for your `.github/copilot-instructions.md` file to target these rules.

### Updated Strategy: Documentation & Naming

Add this specific section to your instructions file. It maps directly to PMD rules `ApexDoc`, `ClassNamingConventions`, `MethodNamingConventions`, and `VariableNamingConventions`.

#### Add this to `copilot-instructions.md`

```markdown
## 5. Documentation Standards (PMD Rule: ApexDoc)
* **Mandatory ApexDoc:** Every Class, Interface, and Public Method must have a formal ApexDoc header.
* **Required Tags:** You must include the `@description` tag in every header.
* **Format:**
    ```apex
    /**
     * @description [Clear, concise description of functionality]
     * @author [User Context]
     * @group [Functional Area]
     */
    ```
* **Missing Docs:** Do not generate public methods without describing their parameters (`@param`) and return values (`@return`).

## 6. Naming Conventions (Strict)
* **Classes:** Use `PascalCase`. strictly NO underscores (`_`) allowed in class names (Rule: `ClassNamingConventions`).
    * *Bad:* `Account_Service`, `Trigger_Handler`
    * *Good:* `AccountService`, `TriggerHandler`
* **Methods:** Use `camelCase`. Begin with a verb (e.g., `getAccounts`, `updateStatus`). (Rule: `MethodNamingConventions`).
* **Variables:** Use `camelCase`. Boolean variables should start with `is`, `has`, or `should` (e.g., `isActive`). (Rule: `VariableNamingConventions`).
* **Constants:** Use `UPPER_SNAKE_CASE` for `static final` variables only.
* **Test Classes:** Must end with the suffix `Test` (e.g., `AccountServiceTest`).

```

---

### Why this matters (The PMD Rules)

To get Copilot to respect these, you have to understand exactly what PMD flags. Here is the breakdown of the specific rules these instructions are patching:

| PMD Rule | Common Copilot Mistake | The Fix in Instruction |
| --- | --- | --- |
| **ApexDoc** | Generates `//` single line comments or no comments at all. | Explicitly demand `/** ... */` blocks with `@description`. |
| **ClassNamingConventions** | Often suggests snake_case for utilities (e.g., `Http_Utils`). | Explicitly forbid underscores in class names. |
| **VariableNamingConventions** | Uses short names like `acc` or `lst`. | Implicitly handled by "Clean Code" but reinforced by asking for descriptive `camelCase`. |
| **MethodNamingConventions** | Sometimes suggests `Submit_Form()` style. | Enforce `camelCase` starting with a verb. |

### Interactive Prompting for Refactoring

If you are using Copilot Chat to fix existing code that violates these rules, use this precise prompt:

> "Refactor this class to satisfy PMD documentation and naming rules. Rename the class to remove underscores (PascalCase), rename methods to camelCase, and add a formal ApexDoc header with `@description` to the class and all public methods."

### Example of PMD-Compliant Output

With the instructions above, when you ask: *"Create a service to calculate tax,"* Copilot should now generate this (instead of a bare class):

```apex
/**
 * @description Service class to handle tax calculations for Opportunity Line Items.
 * @author Copilot
 */
public with sharing class TaxCalculationService {

    private static final Decimal DEFAULT_TAX_RATE = 0.20;

    /**
     * @description Calculates the total tax for a list of amounts.
     * @param amounts List of Decimals representing line item prices.
     * @return Decimal The total calculated tax.
     */
    public static Decimal calculateTotalTax(List<Decimal> amounts) {
        // Implementation...
    }
}

```

**Would you like me to generate a `ruleset.xml` file configured with these specific naming and doc rules so you can run a local PMD scan to verify Copilot's output?**
