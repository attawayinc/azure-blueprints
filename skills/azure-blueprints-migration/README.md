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

> This skill is guidance only. Always verify commands against the latest Microsoft Learn documentation before running them.
