---
name: azure-blueprints-migration
description: Use when a user needs to migrate off Azure Blueprints (definitions and/or assignments) to Template Specs and Deployment Stacks before the January 31, 2027 retirement. Covers inventory, export, conversion to Bicep, policy decoupling, Template Spec publishing, Deployment Stack deployment with deny-settings, validation, and cutover.
---

# Azure Blueprints → Template Specs + Deployment Stacks Migration Skill

**Audience:** Any AI assistant (for example, GitHub Copilot) helping an Azure customer or partner migrate off Azure Blueprints.
**Status:** Azure Blueprints (preview) is **retiring on January 31, 2027**, with **phased retirement beginning July 31, 2026**. Migrate blueprint definitions **and** assignments to **Deployment Stacks** (recommended) + **Template Specs** before then. Authoritative dates, phased timeline, and FAQ: <https://aka.ms/AzureBlueprintsRetirement>.

---

## When to use this skill

Invoke this skill whenever the user:

- Mentions Azure Blueprints, blueprint definitions, blueprint assignments, blueprint locks, or the Blueprints retirement.
- Asks for a migration plan, "what replaces Blueprints?", or how to move governance from Blueprints to ARM/Bicep tooling.
- Needs to export, convert, or re-deploy resources currently managed by Blueprints.
- Asks about Template Specs or Deployment Stacks in the context of replacing Blueprints.

If the user is asking a general ARM/Bicep question with no Blueprints context, do **not** load this skill.

---

## Operating principles (read first)

1. **Stacks over Template Specs alone.** Use Template Specs as the storage/versioning layer; deploy via Deployment Stacks to preserve assignment-like lifecycle behavior. Recommend Template Spec–only deployment only when the user explicitly has no need for resource tracking or deny-settings.
2. **Do not auto-migrate.** Always confirm scope, parameters, and lock parity with the user before producing or running deployment commands. Migration is customer-driven.
3. **Cutover is destructive.** Removing a Blueprint assignment can remove embedded policy assignments. Always sequence: deploy new Stack → verify → re-assign decoupled policies if needed → only then remove the Blueprint assignment.
4. **Never invent IDs.** If the user hasn't provided a management group ID, subscription ID, blueprint name, or version, ask before generating commands.
5. **Surface the retirement dates.** When users plan timelines, remind them Azure Blueprints (preview) **retires January 31, 2027**, with **phased retirement starting July 31, 2026** — so complete migration and export definitions before then. Link to <https://aka.ms/AzureBlueprintsRetirement> for the phased timeline.
6. **Definitions count too, not just assignments.** The retirement recommendation flags **every** blueprint definition and published version — not only assignments. Unassigned definitions deploy nothing, but they still must be handled (convert to a Template Spec, or export-and-archive) and then **deleted** to clear the recommendation. Inventory with `Get-AzBlueprint` at **each subscription and management group**, not just `Get-AzBlueprintAssignment`.

---

## Inputs to gather from the user

Before producing any migration plan or commands, collect:

- **Scope** — management group ID, subscription IDs, and/or resource group names involved.
- **Blueprint inventory** — definition names + versions, and which subscriptions/MGs they're assigned to. Offer to help inventory via `Get-AzBlueprint` / `Get-AzBlueprintAssignment` if unknown.
- **Lock mode in use** — `None` / `AllResourcesReadOnly` / `AllResourcesDoNotDelete`. This drives the Stack `denySettings` choice.
- **Identity** — system-assigned vs. user-assigned managed identity used by the assignment.
- **Policy posture** — whether policies embedded in the blueprint are app-specific (keep in template) or baseline (lift to a standalone Initiative).
- **Target tooling** — Bicep (recommended) or ARM JSON.
- **Production tolerance** — sandbox available? Side-by-side acceptable? Or strict adopt-in-place required?

If any of these are unknown, ask the user before proceeding.

---

## TL;DR — What replaces what

| Blueprints concept | Recommended replacement | Why |
|---|---|---|
| **Blueprint Definition** (package of artifacts: ARM templates, policy assignments, RBAC, resource groups) | **Template Spec** | Stores and versions ARM/Bicep templates as first-class Azure resources; closest parity to a Blueprint Definition. |
| **Blueprint Assignment** (deployment instance + lifecycle binding to a subscription/MG) | **Deployment Stack** | Tracks the resources it creates, supports deny-settings (parity to Blueprint locks/deny-assignments), and lets you update/delete the whole set as one unit. **Preferred end-state.** |
| **Blueprint locks / deny-assignments** | **Deployment Stack `denySettings`** | `denyDelete` or `denyWriteAndDelete` modes reproduce the protective behavior with first-class tooling. |
| **Blueprint parameters** | Template Spec / Stack parameters | Direct 1:1; pass the same values through the new deployment call. |
| **Sequenced artifact ordering** | Bicep `dependsOn` / ARM `dependsOn` | Express ordering in the template rather than via Blueprint artifact sequence. |
| **Policy assignments inside a Blueprint** | Standalone policy assignment **or** Azure Policy initiative (preferred for shared baselines) | Decouple policy from app/infra deployment so it can be managed independently. |

---

## Decision tree

```mermaid
flowchart TD
  A[Have an Azure Blueprint today?] --> B{Definition or<br/>Assignment?}
  B -->|Definition only<br/>(reusable template)| C[Migrate to<br/>Template Spec]
  B -->|Assignment<br/>(deployed instance)| D{Need lifecycle<br/>tracking + deny-locks?}
  D -->|Yes| E[Deploy via<br/>Deployment Stack<br/>with denySettings]
  D -->|No, one-shot| F[Deploy from<br/>Template Spec directly]
  C --> G[Done — versioned,<br/>shareable template]
  E --> H[Done — tracked,<br/>protected, updatable]
  F --> H

  style E fill:#cfe8ff,stroke:#0078d4,stroke-width:2px
  style C fill:#e6f4ea,stroke:#188038,stroke-width:2px
```

---

## End-to-end migration flow

```mermaid
flowchart LR
  subgraph Today["Today — Azure Blueprints"]
    BD[Blueprint<br/>Definition]
    BA[Blueprint<br/>Assignment]
    BL[Locks / Deny<br/>Assignments]
    BD --> BA
    BA --> BL
  end

  subgraph Migration["Migration steps"]
    INV[1. Inventory<br/>blueprints]
    EXT[2. Export<br/>definition JSON]
    CONV[3. Convert artifacts<br/>to Bicep/ARM]
    POL[4. Split policies<br/>into Policy / Initiative]
    TS[5. Publish<br/>Template Spec]
    STK[6. Deploy via<br/>Deployment Stack]
    VAL[7. Validate<br/>in non-prod]
    CUT[8. Cut over<br/>+ remove BP assignment]
  end

  subgraph Future["Future state"]
    TSF[Template Spec<br/>(versioned package)]
    STKF[Deployment Stack<br/>(tracked resources)]
    DEN[denySettings<br/>(deny-delete)]
    POLF[Policy Assignment<br/>/ Initiative]
    TSF --> STKF
    STKF --> DEN
  end

  BD -.->|maps to| TSF
  BA -.->|maps to| STKF
  BL -.->|maps to| DEN

  INV --> EXT --> CONV --> POL --> TS --> STK --> VAL --> CUT
  CUT --> STKF
```

---

## How to drive a migration (step-by-step instructions for the AI)

For each step, **(a)** explain to the user what you're about to do, **(b)** ask any missing inputs, **(c)** produce the command or artifact, **(d)** wait for the user to run it and report results before moving on.

### Step 1 — Inventory existing Blueprints

Ask for the management group ID and subscription ID, then produce:

```powershell
Get-AzBlueprint -ManagementGroupId "<mg-id>" | Select-Object Name, Id, Versions
Get-AzBlueprintAssignment -SubscriptionId "<sub-id>"
```

For each blueprint, capture: definition ID, latest published version, scopes assigned, assignment parameter values, lock mode, identity.

### Step 2 — Export the Blueprint definition

```powershell
$bp = Get-AzBlueprint -ManagementGroupId "<mg-id>" -Name "<blueprint-name>" -LatestPublished
Export-AzBlueprintWithArtifact -Blueprint $bp -OutputPath ".\exports\<blueprint-name>"
```

Output is `blueprint.json` plus one JSON file per artifact. Confirm the export folder structure with the user before continuing.

### Step 3 — Convert artifacts to Bicep (or keep as ARM)

For each `template` artifact:
1. Save the embedded template to a `.json` file.
2. Decompile to Bicep (recommended): `bicep decompile .\template.json`. Decompiled output is a starting point — review and refactor it, since it is not always idiomatic.
3. Combine related artifacts into a single `main.bicep`, or keep modular with `module` references.
4. Preserve the same parameters at the top level (`@description`, `@allowed`, defaults).

For RBAC artifacts: add `Microsoft.Authorization/roleAssignments` resources with a deterministic `guid()` name.
For resource-group artifacts: model as `Microsoft.Resources/resourceGroups` at subscription scope, or skip if the target RG already exists.

### Step 4 — Decouple policy assignments

- **Shared baseline** → publish as an Azure Policy **Initiative** at the MG and assign once. The Stack/template should **not** re-assign baseline policies.
- **App-specific policy** → keep inside the Bicep, assign at resource-group scope so it doesn't conflict with broader inheritance.

Warn the user: removing the Blueprint assignment later will remove any policies embedded in it. Re-assign decoupled policies **before** removing the Blueprint assignment.

### Step 5 — Publish a Template Spec

```powershell
New-AzTemplateSpec `
  -ResourceGroupName "rg-templatespecs" `
  -Name "landing-zone" `
  -Version "1.0.0" `
  -Location "eastus" `
  -TemplateFile ".\main.bicep"
```

This is the versioned, shareable, RBAC-scoped artifact that replaces the Blueprint definition as the **package**.

### Step 6 — Deploy via a Deployment Stack

```powershell
# Retrieve the Template Spec version resource ID
$ts = Get-AzTemplateSpec -ResourceGroupName "rg-templatespecs" -Name "landing-zone" -Version "1.0.0"

New-AzSubscriptionDeploymentStack `
  -Name "landing-zone-stack" `
  -Location "eastus" `
  -TemplateSpecId $ts.Versions[0].Id `
  -TemplateParameterFile ".\params.json" `
  -DenySettingsMode "denyDelete" `
  -ActionOnUnmanage "detachAll"
```

Decision points to surface to the user:
- `-DenySettingsMode` → `none` | `denyDelete` | `denyWriteAndDelete`. Map to the old lock mode (see mapping table below).
- `-ActionOnUnmanage` → controls what happens to resources when you *delete* the stack, or when a resource drops out of the template. Use `detachAll` during pilot; switch to `deleteResources` for full-lifecycle parity with a Blueprint assignment delete.
- For management-group scope use `New-AzManagementGroupDeploymentStack` instead.

> **Note — What-If:** Deployment stacks do not yet support a native What-If preview. To preview changes, run `New-AzSubscriptionDeployment -WhatIf` (or `az deployment sub what-if`) against the underlying template before creating or updating the stack. The stack additionally applies the deny-settings, which won't appear in the template What-If.

### Step 7 — Validate in non-production

Deploy to a sandbox subscription first. Confirm with the user:
- Resources created as expected.
- Deny-settings enforced (try a manual delete — it should fail).
- Parameter values applied correctly.
- Diff against the original Blueprint assignment: any drift?

### Step 8 — Cut over

For each existing Blueprint assignment:
1. Deploy the matching Deployment Stack to the same scope.
2. Choose one strategy and confirm with the user:
   - **Side-by-side then swap** — Stack provisions new resources, redirect dependents, then remove old Blueprint-managed resources.
   - **Adopt existing** — the stack begins managing resources whose IDs match resources declared in the template on first successful deployment (there is no separate import command). Resources at the scope that are **not** in the template are left untouched under `detachAll`.
3. Re-assign any decoupled policies at the right scope.
4. Remove the Blueprint assignment:
   ```powershell
   Remove-AzBlueprintAssignment -Name "<assignment>" -SubscriptionId "<sub>"
   ```
5. Confirm the Stack is now the single source of truth: `Get-AzSubscriptionDeploymentStack`.

> **Sequencing gotcha — embedded role/policy assignments.** While the Blueprint assignment still exists, it *owns* the role and policy assignments it created. If your Stack template tries to create a role assignment for the **same principal + role + scope**, Azure rejects it with `409 RoleAssignmentExists`; re-creating an embedded policy under a new name silently produces a **duplicate**. Removing the Blueprint assignment does **not** reliably delete the role assignments it created — they often **persist** and keep triggering `409`. The most reliable pattern is to **adopt** the existing assignments: query the existing name (`Get-AzRoleAssignment`, or the REST role-assignments list filtered by `principalId`) and set that exact GUID as the resource `name` so the deployment is an idempotent no-op the stack then manages.

### Step 9 — Cleanup

Once **all** assignments for a Definition are migrated, delete the definition. **Note:** `Az.Blueprint` has **no `Remove-AzBlueprint` cmdlet** (only `Remove-AzBlueprintAssignment`). Delete definitions via Azure CLI or REST, deleting published versions first:

```powershell
# Azure CLI
az blueprint delete --name "<blueprint-name>" --management-group "<mg-id>"   # or --subscription "<sub-id>"

# REST fallback (PowerShell) — published versions first, then the definition
$path = "/subscriptions/<sub-id>/providers/Microsoft.Blueprint/blueprints/<name>"  # or /providers/Microsoft.Management/managementGroups/<mg-id>/providers/Microsoft.Blueprint/blueprints/<name>
$api  = "api-version=2018-11-01-preview"
(Invoke-AzRestMethod -Method GET -Path "$path/versions?$api").Content |
  ConvertFrom-Json | Select-Object -Expand value |
  Where-Object { -not [string]::IsNullOrWhiteSpace($_.name) } |   # never issue a DELETE with an empty version ID
  ForEach-Object { Invoke-AzRestMethod -Method DELETE -Path "$path/versions/$($_.name)?$api" }
Invoke-AzRestMethod -Method DELETE -Path "${path}?$api"
```

Deleting a **management-group-scoped** definition requires Blueprint Contributor/Owner **at that management group** — a subscription-level role returns `403`. Stop publishing new versions of the Blueprint Definition.

---

## Mapping reference

| Blueprint setting | Stack equivalent |
|---|---|
| `properties.parameters` | `-TemplateParameterFile` / `-Parameter` |
| `properties.resourceGroups` | `Microsoft.Resources/resourceGroups` in Bicep |
| `locks.mode = AllResourcesDoNotDelete` | `DenySettingsMode = denyDelete` |
| `locks.mode = AllResourcesReadOnly` | `DenySettingsMode = denyWriteAndDelete` |
| `locks.excludedPrincipals` | `-DenySettingsExcludedPrincipal` (max 5) |
| `locks.excludedActions` | `-DenySettingsExcludedAction` (max 200) |
| Identity (`SystemAssigned` / `UserAssigned`) | Stack runs with the caller's identity; deployments use the template's own identity blocks |
| Artifact sequence | Bicep `dependsOn` |
| Versioning (definition versions) | Template Spec `version` |
| Assignment scope (Sub / MG) | `New-AzSubscriptionDeploymentStack` / `New-AzManagementGroupDeploymentStack` |

---

## Guardrails — things the AI must do

- **Do not generate or run any `Remove-AzBlueprintAssignment` command until** the user confirms the replacement Stack is deployed and validated, and any decoupled policies are re-assigned.
- **Do not recommend `denyWriteAndDelete`** unless the user explicitly needs write-protection — it blocks normal RBAC-allowed updates. Default to `denyDelete`.
- **Do not recommend `-ActionOnUnmanage deleteResources` during pilot.** Use `detachAll` until the user is confident.
- **Do not invent management group IDs, subscription IDs, blueprint names, or parameter values.** Ask.
- **Do not claim Microsoft will auto-migrate.** It will not. This is customer-driven.
- **Do not skip the policy-decoupling step** if the blueprint embeds policy assignments. This is the #1 source of post-migration regressions.

---

## Common pitfalls

- **Trying to one-shot adopt existing resources.** Stacks track resources they create or that already match resources declared in the template; other pre-existing resources may need a brief side-by-side period.
- **Forgetting to update policy ownership.** If a Blueprint assigned policy that lived inside the definition, removing the assignment also removes that policy. Re-assign at the right scope *before* removing the Blueprint assignment.
- **Wrong deny-settings mode.** `denyWriteAndDelete` blocks normal RBAC-allowed updates — pick `denyDelete` unless you truly need write protection.
- **`ActionOnUnmanage = deleteResources` during pilot.** Use `detachAll` until you're confident; flip later.
- **Deny-assignment limits and gaps.** Deny-assignments don't apply at management-group scope (only when a stack deploys into a child subscription), don't cover data-plane operations, and don't protect implicitly created child resources (for example, an AKS node resource group). Add explicit `Microsoft.Authorization/locks` where those gaps matter.
- **Blueprint-applied tags get stripped.** Blueprints stamp managed resources/resource groups with a `Blueprint` tag (and similar metadata). A migrated Stack whose template omits those tags will **remove them** on first deployment (visible as a `~ Modify` in template What-If). Carry over any tags you want to keep into the converted Bicep, or accept their removal deliberately.
- **Role assignment `409` / duplicate policy on cutover.** See the sequencing gotcha in Step 8 — defer embedded role/policy assignments until after the Blueprint assignment is removed, or adopt them by exact existing name.
- **Inherited governance noise.** At resource-group scope you may see many *inherited* management-group policy assignments. Don't mistake them for the blueprint's own artifacts when validating.
- **`Remove-AzBlueprint` doesn't exist.** `Az.Blueprint` ships `Remove-AzBlueprintAssignment` but **no** cmdlet to delete a *definition*. Use `az blueprint delete` or REST `DELETE` (see Step 9); delete published versions first. MG-scoped deletes need MG-level permissions or they return `403`.
- **Unassigned definitions still count.** The retirement recommendation lists every definition and version, not just assignments — and the same test definitions are often replicated across every subscription and the management group. Inventory each scope and archive/convert then delete them to reach zero.
- **`Get-AzBlueprint -SubscriptionId` includes inherited MG definitions.** Its result mixes subscription-scoped definitions with management-group definitions inherited into that subscription's view. Distinguish by the resource `Id` (`/subscriptions/…` vs `/managementGroups/…`). Don't multiply the same MG set across every subscription when counting, and note that deleting an inherited MG definition via a *subscription* path is a no-op (idempotent `204`) — delete it once at the MG scope (with MG permissions).
- **Never delete a version with an empty ID.** When removing a blueprint's published versions, filter out any null/blank version names *before* issuing the `DELETE` — a `DELETE …/versions/` with an empty version ID can trigger a service-side error. Enumerate versions, skip empties, delete each real version, then delete the definition.
- **Export before retirement.** Export your blueprint definitions and assignments before the **January 31, 2027** retirement (phased changes begin **July 31, 2026**). After retirement the Blueprints API stops serving requests, so `Export-AzBlueprintWithArtifact` may no longer work — capture everything you need ahead of time.

---

## Output artifacts the AI should produce

By the end of a migration session, the user should have:

1. **Inventory list** — definitions, assignments, scopes, lock modes, parameters.
2. **Export folder** per blueprint — `blueprint.json` + artifact JSON files.
3. **Converted Bicep templates** — `main.bicep` per migrated definition.
4. **Template Spec resource(s)** — published and versioned.
5. **Deployment Stack(s)** — deployed at the correct scope with the right `denySettings`.
6. **Decoupled policy assignments / initiatives** — assigned at the appropriate MG/sub/RG scope.
7. **Cutover checklist** — one row per former Blueprint assignment, with status: deployed / validated / cut over / Blueprint removed.

---

## FAQ

**Q: Do I have to use Stacks? Can I just keep using Template Spec deployments?**
A: Yes, for stateless deployments. But you lose lifecycle tracking and deny-protection. Use Stacks when you used Blueprint *assignments* for governance.

**Q: What about Deployment Stacks at the management-group scope?**
A: Supported (`New-AzManagementGroupDeploymentStack`). This is the right replacement for MG-scoped Blueprint assignments that fan out to many subs. Note that deny-assignments apply at the child-subscription scope the stack deploys into, not at the management-group scope itself.

**Q: Will Microsoft auto-migrate my Blueprints?**
A: No. Migration is customer-driven — there is no automatic conversion. Use this guide to migrate on your own timeline before retirement.

**Q: What happens on the retirement dates?**
A: Phased retirement begins **July 31, 2026**, and the service is fully **retired January 31, 2027** — after which create/update operations stop and definitions/assignments are no longer serviced. Export definitions and complete migration before then. See <https://aka.ms/AzureBlueprintsRetirement>.

**Q: I only have unassigned blueprint *definitions* left. Do those need migrating?**
A: Yes — the retirement recommendation counts every definition and published version. If a definition has no assignment it deploys nothing, so either convert it to a Template Spec (if you still want the template) or simply **export it for archive and delete it**. Deleting definitions (and their versions) is what clears them from the retirement recommendation.

**Q: Where do I file questions or issues?**
A: Use the public [`Azure/azure-blueprints`](https://github.com/Azure/azure-blueprints) GitHub repository and the official retirement notice on Microsoft Learn.

---

## Resources

- **Azure Blueprints retirement (hub — dates, phased timeline, FAQ)** — <https://aka.ms/AzureBlueprintsRetirement>
- **Migrate blueprints to deployment stacks (official guide)** — <https://learn.microsoft.com/azure/azure-resource-manager/bicep/migrate-blueprint>
- **Migrate blueprints to template specs** — <https://learn.microsoft.com/azure/governance/blueprints/migrate-to-template-specs>
- **Azure Blueprints overview** — <https://learn.microsoft.com/azure/governance/blueprints/overview>
- **Template Specs overview** — <https://learn.microsoft.com/azure/azure-resource-manager/templates/template-specs>
- **Deployment Stacks overview** — <https://learn.microsoft.com/azure/azure-resource-manager/bicep/deployment-stacks>
- **Bicep decompile** — <https://learn.microsoft.com/azure/azure-resource-manager/bicep/decompile>
- **Deny assignments** — <https://learn.microsoft.com/azure/role-based-access-control/deny-assignments>
- **Az.Blueprint reference (export)** — <https://learn.microsoft.com/powershell/module/az.blueprint>

---

*Skill version: 2.1 · Maintained by Microsoft. Verify all commands against the latest Microsoft Learn documentation before running.*
