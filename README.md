# Contribution 1: Support Discord as a first-class notification/alert target

**Contribution Number:** 1
**Student:** Ben Klein
**Issue:** https://github.com/graphql-hive/console/issues/130
**Status:** Phase 1 Complete

---

## Why I Chose This Issue

I chose issue #130 because it fits with my existing TS/JS experience, and will push my understanding forward just the right amount for a first issue. The issue seems like a great first issue as there is already Teams and Slack integration so it should be straight forward to follow that model to build a Discord integration.

I'm interested in this issue because:

1. I've done a lot of web development work but haven't used GraphQL yet and I want to learn a bit more about it.
2. There is already a model for alerts with other platforms.

---

## Understanding the Issue

### Problem Description

There are already alert/notification integrations for GraphQL Hive on Slack and Teams. The goal of this issue is to add integration for Discord. For example, a notification for Discord for a GraphQL Hive user might look like:

```
I found 1 change in project GraphQL Hive, target production (view details):
Breaking Changes
Field `type` was removed from object type `Organization`
```

### Expected Behavior

When a GraphQL Schema is updated with breaking changes, it should send an alert via Discord to the API maintainer.

### Current Behavior

Currently, alerts and notifications are only supported for Slack and Teams.
https://the-guild.dev/graphql/hive/docs/schema-registry/management/projects

### Affected Components

| Layer                     | Reference file(s)                                                                                                                                 |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| DB enum migration         | `packages/migrations/src/actions/2024.06.11T10-10-00.ms-teams-webhook.ts`                                                                         |
| GraphQL schema            | `packages/services/api/src/modules/alerts/module.graphql.ts`                                                                                      |
| Schema-change adapter     | `packages/services/api/src/modules/alerts/providers/adapters/msteams.ts`                                                                          |
| Adapter tests             | `packages/services/api/src/modules/alerts/providers/adapters/msteams.spec.ts`                                                                     |
| Dispatch routing          | `packages/services/api/src/modules/alerts/providers/alerts-manager.ts`                                                                            |
| DI registration           | `packages/services/api/src/modules/alerts/index.ts`                                                                                               |
| GraphQL resolver          | `packages/services/api/src/modules/alerts/resolvers/TeamsWebhookChannel.ts`                                                                       |
| Storage / Zod enum        | `packages/services/storage/src/index.ts`, `packages/services/api/src/modules/alerts/providers/metric-alert-rules-storage.ts`                      |
| Metric alerts (workflows) | `packages/services/workflows/src/lib/metric-alert-notifier.ts`, `packages/services/workflows/src/tasks/send-metric-alert-channel-notification.ts` |
| UI create channel         | `packages/web/app/src/components/project/alerts/create-channel.tsx`                                                                               |

---

## Reproduction Process

### Environment Setup

After a little troubleshooting, the project is running locally. First I tried and failed to run it in Windows, and switched to WSL (Ubuntu) as the project seems designed to run on a Unix system. I had a couple other minor issues with Docker, make sure that Docker is set up to work with WSL2. Here is the final steps needed to run the project, summarized by Cursor's agent (Some of the pathnames are mine, replace with your local path names):

1. Clone the repo into the WSL filesystem:

git clone https://github.com/ben-m-klein/console.git ~/projects/graphql_console_repo
cd ~/projects/graphql_console_repo

2. Node.js 24.14.1 via nvm:

nvm install 24.14.1
nvm use 24.14.1 # repo has .node-version, not .nvmrc

Optional: cp .node-version .nvmrc so plain nvm use works.

3. pnpm ≥ 10.33.2 (Corepack or global install):

corepack enable
corepack prepare pnpm@10.33.2 --activate

4. Docker Desktop with WSL2 integration enabled; confirm the daemon is running:

docker info

5. Free ports: 5432, 6379, 9000, 9001, 8123, 9092, 8081, 8082, 9644, 3567, 7043, 10255 (plus 3000, 3001, 3014 for app services).

# One-time setup

6. Create root .env:

echo 'ENVIRONMENT=local' > .env

7. Install dependencies (also runs env:sync to generate service .env files):

pnpm i

8. Start Docker dependencies and run migrations:

pnpm local:setup

9. Fix WSL/Linux volume permissions if containers fail to start:

# Redis (Bitnami runs as UID 1001)

sudo chown -R 1001:1001 docker/.hive-dev/redis

# Postgres (Alpine postgres runs as UID 70)

sudo chown -R 70:70 docker/.hive-dev/postgresql
docker compose -f docker/docker-compose.dev.yml up -d
Do not blanket-chown all of docker/.hive-dev/ — that breaks Postgres.

10. Generate types/codegen:

pnpm generate

11. Build workspace packages (required for @graphql-hive/laboratory and others):

pnpm build
At minimum if you want a faster start:

pnpm --filter @graphql-hive/laboratory build

# Run the dev environment

12. Start all Hive services:

cd ~/projects/graphql_console_repo
nvm use 24.14.1
pnpm dev:hive
Alternative: VS Code “Start Hive” button (with recommended extensions installed).

13. Verify services are up:

Service URL
UI
http://localhost:3000
GraphQL API / auth
http://localhost:3001/graphql
Workflows (email history)
http://localhost:3014/\_history

Quick check:

curl -s -o /dev/null -w "3000:%{http_code} 3001:%{http_code} 3014:%{http_code}\n" \
 http://localhost:3000/ \
 http://localhost:3001/graphql \
 http://localhost:3014/\_history

# Create an account and log in

14. Open http://localhost:3000 and sign up with any email/password.

15. Sign in at http://localhost:3000 with the same credentials.

# Daily workflow

Start:

cd ~/projects/graphql_console_repo
nvm use 24.14.1
docker compose -f docker/docker-compose.dev.yml up -d # if Docker stack isn't running
pnpm dev:hive

Stop:

# Ctrl+C in the dev:hive terminal

docker compose -f docker/docker-compose.dev.yml down # optional: stop Docker deps

### Steps to Reproduce

1. Produced a breaking change alert with both a test webhook sent to webhook.site, and one sent to Discord. Only the webhook.site fired, the discord webhook silently failed. The discord alert does not work on generic 'webhook' type alerts, the new alert type is needed.
   Evidence: See reproduction evidence link below for documentation and screenshots of the missing behavior.
   Root cause: no DISCORD_WEBHOOK channel type / adapter (MS Teams pattern in msteams.ts)

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/ben-m-klein/console/tree/feature/discord-webhook-alerts
- **Screenshots/logs:** https://docs.google.com/document/d/14UF1k0HbEYe6rdaET3-WEkJYu0XVz0TtVClU0Y7eShw/edit?usp=sharing
- **My findings:** Produced a breaking change alert with both a test webhook sent to webhook.site, and one sent to Discord. Only the webhook.site fired, the discord webhook silently failed. The discord alert does not work on generic 'webhook' type alerts, the new alert type is needed.

---

## Solution Approach

### Analysis

The Discord alert webhook doesn't exist. The issue is for creating a Discord compatible webhook to deliver alerts when a breaking change is made to a schema.

### Proposed Solution

Create the Discord alert as laid out in the following implementation plan section below.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The Discord alert webhook doesn't exist. The issue is for creating a Discord compatible webhook to deliver alerts when a breaking change is made to a schema.

**Match:** There are generic webhooks (confirmed working using webhook.site), MS teams webhooks and slack webhooks that can be used as reference.

**Plan:**

| Layer                     | Reference file(s)                                                                                                                                 |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| DB enum migration         | `packages/migrations/src/actions/2024.06.11T10-10-00.ms-teams-webhook.ts`                                                                         |
| GraphQL schema            | `packages/services/api/src/modules/alerts/module.graphql.ts`                                                                                      |
| Schema-change adapter     | `packages/services/api/src/modules/alerts/providers/adapters/msteams.ts`                                                                          |
| Adapter tests             | `packages/services/api/src/modules/alerts/providers/adapters/msteams.spec.ts`                                                                     |
| Dispatch routing          | `packages/services/api/src/modules/alerts/providers/alerts-manager.ts`                                                                            |
| DI registration           | `packages/services/api/src/modules/alerts/index.ts`                                                                                               |
| GraphQL resolver          | `packages/services/api/src/modules/alerts/resolvers/TeamsWebhookChannel.ts`                                                                       |
| Storage / Zod enum        | `packages/services/storage/src/index.ts`, `packages/services/api/src/modules/alerts/providers/metric-alert-rules-storage.ts`                      |
| Metric alerts (workflows) | `packages/services/workflows/src/lib/metric-alert-notifier.ts`, `packages/services/workflows/src/tasks/send-metric-alert-channel-notification.ts` |
| UI create channel         | `packages/web/app/src/components/project/alerts/create-channel.tsx`                                                                               |

---

## Plan

1. **Migration** — Add `DISCORD_WEBHOOK` to Postgres enum `alert_channel_type` (new file under `packages/migrations/src/actions/`).

2. **GraphQL** — Add `DISCORD_WEBHOOK` to `AlertChannelType`; add `DiscordWebhookChannel implements AlertChannel { id, name, type, endpoint }`. Run `pnpm graphql:generate`.

3. **Adapter** — Create `discord.ts` implementing `CommunicationAdapter`:
    - `sendSchemaChangeNotification` — build embed(s) from `changes`, `messages`, `errors`, `initial`; include links to Hive UI (use `WEB_APP_URL` like MS Teams).
    - `sendChannelConfirmation` — short “I will send notifications here” message on channel create/delete.
    - Private `sendDiscordMessage(webhookUrl, payload)` with Discord payload limits (~2000 chars content, embed limits).

4. **Register adapter** — Add to `packages/services/api/src/modules/alerts/index.ts`; inject into `AlertsManager`.

5. **Route in AlertsManager** — In `triggerSchemaChangeNotifications` and `triggerChannelConfirmation`, add `channel.type === 'DISCORD_WEBHOOK'` branch (alongside `MSTEAMS_WEBHOOK` ~lines 306–312, 378–379).

6. **Resolver** — Add `DiscordWebhookChannel.ts` (`__isTypeOf`, `endpoint` from `webhookEndpoint`).

7. **Storage** — Extend `AlertChannelModel` / DB types enum with `DISCORD_WEBHOOK`.

8. **Workflows (metric alerts)** — Add `sendDiscordNotification()` in `metric-alert-notifier.ts`; add `case 'DISCORD_WEBHOOK'` in `send-metric-alert-channel-notification.ts`.

9. **UI** — In `create-channel.tsx`: add “Discord Webhook” to type dropdown; treat as webhook-like (URL input); add Discord webhook setup doc link; extend `ChannelsTable_AlertChannelFragment`.

10. **Tests** — Add `discord.spec.ts` (Vitest, mock `fetch`): schema change with breaking field, channel confirmation, payload shape assertions.

11. **Manual verify** — Local Hive + Discord test server webhook; publish schema change via CLI; confirm embed in Discord.

**Implement:** Changes will be posted here: https://github.com/ben-m-klein/console/tree/feature/discord-webhook-alerts

**Review:** Yes, and I will confirm the guidelines are followed as the implementation proceeds.

**Evaluate:** Confirm that I am able to create a Discord alert in the UI, change the schema with a breaking change and trigger a Discord alert.

---

## Testing Strategy

### Unit Tests

Added and ran Discord adapter tests covering:

- schema change notification payloads
- channel confirmation payloads
- Discord embed formatting and truncation behavior
  Also reran the existing MS Teams webhook adapter tests to confirm the shared webhook-style behavior still passes.

### Integration Tests

Ran the relevant project checks:

- `pnpm graphql:generate`
- `pnpm --filter @hive/app typecheck`
- `pnpm --filter @hive/workflows typecheck`
- `pnpm exec vitest run packages/services/api/src/modules/alerts/providers/adapters/discord.spec.ts packages/services/api/src/modules/alerts/providers/adapters/msteams.spec.ts`
  Result: all checks passed.

### Manual Testing

Created a Discord Webhook alert channel in the local Hive UI. The confirmation message was delivered successfully to Discord. See the note doc for screenshots: https://docs.google.com/document/d/14UF1k0HbEYe6rdaET3-WEkJYu0XVz0TtVClU0Y7eShw/edit?usp=sharing

Created a schema change notification alert for the `development` target using the Discord channel. Published schema changes through the local Hive CLI and verified Discord received:

- a breaking schema change notification
- a safe schema change notification

---

## Implementation Notes

### Week 3 Progress

Implemented Discord as a alert channel type for GraphQL Hive. Added database/storage/API support for the `DISCORD_WEBHOOK` channel type, exposed it through GraphQL, created a Discord webhook adapter, routed schema-change and metric-alert notifications to Discord, and added UI support for creating Discord webhook channels.
Challenges included understanding the existing alerts architecture, especially how resolvers, providers, adapters, and workflow tasks interact. Local Docker also required troubleshooting due to a Docker Desktop port-forwarding issue with Kafka/Redpanda.

### Code Changes

- **Files modified:** migrations, storage alert channel types, alerts GraphQL schema/resolvers, Discord notification adapter and tests, alert manager routing, workflow metric alert notifier, and alert channel UI components.
- **Key commits:**
    - `762ac4efb` Add Discord webhook alert channel migration
    - `aa3a51878` Support Discord webhook alert channel type
    - `0847e1572` Add Discord webhook GraphQL channel type
    - `4c8e5afae` Add Discord webhook notification adapter
    - `0589218aa` Route alert notifications to Discord webhooks
    - `57f215b51` Route metric alerts to Discord webhooks
    - `a64836251` Add Discord webhook option to alert channel UI
- **Approach decisions:** Followed the existing MS Teams webhook pattern where possible, while formatting Discord messages with Discord webhook embeds. Added Discord as a webhook-like channel in the UI because it uses a webhook endpoint, but kept it as a distinct channel type so alerts can be formatted specifically for Discord.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**

- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
