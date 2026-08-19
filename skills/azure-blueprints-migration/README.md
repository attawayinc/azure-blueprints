# Azure Blueprints → Template Specs + Deployment Stacks — GitHub Copilot skill

A GitHub Copilot **skill** that guides customers and partners through migrating off
**Azure Blueprints** to **Template Specs** + **Deployment Stacks**.

> [!IMPORTANT]
> Azure Blueprints (preview) is being retired: **phased retirement begins July 31, 2026**, and the service is fully **retired on January 31, 2027**. Migrate before then and export your definitions early.

Drop [`SKILL.md`](./SKILL.md) into your repository's skill/prompt location (or reference it in
GitHub Copilot) and ask Copilot to help you inventory, export, convert, deploy, validate, and
cut over your blueprints.

## Official Microsoft Learn documentation

- **Migrate blueprints to deployment stacks** — <https://learn.microsoft.com/azure/azure-resource-manager/bicep/migrate-blueprint>
- **Migrate blueprints to template specs** — <https://learn.microsoft.com/azure/governance/blueprints/migrate-to-template-specs>
- **Azure Blueprints retirement (dates, phased timeline, FAQ)** — <https://aka.ms/AzureBlueprintsRetirement>

## What the skill covers

- Inventory — definitions **and** assignments, at **subscription** and **management-group** scope
- Export/archive, convert artifacts to Bicep/ARM, and decouple policy assignments
- Publish **Template Specs**, deploy **Deployment Stacks** with `denySettings` (lock parity)
- Validate, cut over, and decommission — with the common **caveats and limitations** called out (for example, role-assignment adoption, tag preservation, and management-group permissions)

## Legal notices and disclaimers

> [!CAUTION]
> **This skill generates commands that can permanently delete or alter Azure resources.** Read the full [Legal notices and disclaimers](./SKILL.md#legal-notices-and-disclaimers) in `SKILL.md` before using it.

In summary:

- **Provided "AS IS."** MIT-licensed sample content (see [`LICENSE`](../../LICENSE)) — **no warranty of any kind**, express or implied, and **no liability** for any damages or data loss arising from its use, to the maximum extent permitted by law.
- **Not a Microsoft product or service.** Not covered by any SLA, support agreement, or product lifecycle policy. File issues in this repository; do not open a Microsoft support case.
- **Guidance, not advice.** Not legal, regulatory, compliance, audit, or security advice, and it certifies no compliance posture. Consult your own advisors before changing regulated or production workloads.
- **AI output may be wrong.** Commands an AI assistant generates from this skill are suggestions, not validated instructions. Review, understand, and test every command yourself.
- **Destructive and irreversible.** Deleting blueprint definitions/assignments, deploying stacks with `deleteResources`, and applying deny-settings can cause permanent loss. **Export and back up first, and validate in a non-production subscription.** You are solely responsible for changes made in your own environment.
- **Dates and roadmap may change.** Nothing here is a commitment by Microsoft. The authoritative sources are <https://aka.ms/AzureBlueprintsRetirement> and [Microsoft Learn](https://learn.microsoft.com/azure/governance/blueprints/overview).
- **Don't paste secrets into an AI chat.** Credentials, keys, tokens, personal data, and confidential information may be processed by the AI provider under its own terms.
