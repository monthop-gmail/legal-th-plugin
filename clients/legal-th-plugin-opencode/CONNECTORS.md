# Connectors

## How tool references work

Plugin files use `~~category` as a placeholder for whatever tool the user connects in that category. For example, `~~cloud storage` might mean Box, Google Drive, or any other storage provider with an MCP server.

Plugins are **tool-agnostic** — they describe workflows in terms of categories rather than specific products. The `.mcp.json` pre-configures specific MCP servers, but any MCP server in that category works.

## Connectors for this plugin

| Category | Placeholder | Included servers | Other options |
|----------|-------------|-----------------|---------------|
| Thai Legal KB | `~~legal-th` | Legal-TH MCP Server | — |
| Chat | `~~chat` | — | Slack, Microsoft Teams, LINE |
| Cloud storage | `~~cloud storage` | — | Box, Google Drive, SharePoint, Egnyte |
| Email | `~~email` | — | Gmail, Microsoft 365 |
| Calendar | `~~calendar` | — | Google Calendar, Microsoft 365 |
| E-signature | `~~e-signature` | — | DocuSign, Adobe Sign |
| Office suite | `~~office suite` | — | Microsoft 365, Google Workspace |
| Project tracker | `~~project tracker` | — | Jira, Linear, Asana |
