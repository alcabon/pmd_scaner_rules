When a release branch has been moved to production and you need to bring your long-lived upstream branches (like `dev` or `int`) back in sync with the "truth" (Production/Master), the correct strategy in **sfdx-hardis** depends on whether you are managing a **team's shared branch** or an **individual developer's sandbox**.

### The Verdict: `sf hardis:project:git:retrofit`

For the specific scenario you described—updating branches like `dev` and `int` after a release—the best command is **`sf hardis:project:git:retrofit`**.

---

### Why Retrofit is the better choice here

In a standard SFDX-Hardis CI/CD flow, once `main` (Production) is updated, your other branches are technically "behind."

* **`hardis:project:git:retrofit`**: This is designed for **branch-to-branch synchronization**. It allows you to select a source branch (e.g., `main`) and "retrofit" those changes into your target branches (e.g., `dev` or `int`). It ensures that the core codebase is aligned across the entire pipeline.
* **`hardis:work:refresh`**: As your description noted, this is a **developer-centric command**. It is meant for a developer working on a *feature branch* to pull the latest changes from `int` or `dev` into their local environment and sync them to their *scratch org* or *sandbox*.

### Comparison Table

| Feature | `hardis:project:git:retrofit` | `hardis:work:refresh` |
| --- | --- | --- |
| **Primary Goal** | Back-porting Production/Release changes to upstream branches. | Keeping a local feature branch & sandbox up to date. |
| **User** | Release Manager / Lead Dev. | Individual Developer. |
| **Scope** | Git Branch  Git Branch. | Git Branch  Local Code  Org. |
| **Typical Use Case** | "I just deployed to Prod, now `dev` needs those changes." | "My colleague pushed to `int`, I need those changes in my sandbox." |

---

### Best Practice Workflow

After a production release, follow these steps to ensure a clean "refresh" of your environment:

1. **Retrofit the branches:** Run `sf hardis:project:git:retrofit`. Select `main` (or `master`) as the source and `int` as the target. Repeat for `dev`.
2. **Sync the Orgs:** Once the Git branches are updated, the CI/CD pipeline (if configured) will usually deploy these changes to the `int` and `dev` sandboxes.
3. **Developers Refresh:** Individual developers should then use `sf hardis:work:refresh` on their personal feature branches to pull those latest `dev`/`int` updates into their local workspace and sandboxes.

> **Note:** Always ensure your local git state is clean before running `retrofit`, as it will perform multiple checkouts and merges.

Would you like me to provide the specific flags for the `retrofit` command to automate this in a CI script?
