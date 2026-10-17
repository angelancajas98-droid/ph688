Telegram-Bot: Serverless (Deno)

The owner-controlled Telegram business automation bot from the spec in
`docs/`, built as a genuinely serverless app:

| Concern | Serverless answer |
|---|---|
| Runtime | Deno (V8), deployed to **Deno Deploy** |
| Receiving updates | Telegram **webhook** -> `Deno.serve` (no polling, no always-on server) |
| Storage | **Deno KV** (persistent data + auto-expiring workflow state) |
| Scheduled publishing | **`Deno.cron`**, every minute |
| Config | `.env` locally (`--env-file`), dashboard env vars in production |

## Features (from `docs/Telegram-Bot.docx`)

Publish to one or many destinations, inline buttons, **re-pointing buttons after
publication**, scheduled posts, destination management, approval-based access,
business assistance for approved users, `@mention` replies in groups, and
automation of all of it.

From `docs/WORKFLOW-PURPOSE.md` it also adds **activity recording** (every
operation is logged for 30 days, visible under Settings), a **Knowledge** base
that powers the assistant's answers, and **Broadcast** to approved users.
Every operation follows the doc's lifecycle: validate -> authorize -> execute ->
record -> respond.

## Quick start

1. Create a bot with [@BotFather](https://t.me/BotFather); get your numeric id from
   [@userinfobot](https://t.me/userinfobot).
2. `cp .env.example .env` and fill it in (generate the secret with
   `openssl rand -hex 24`).
3. **Try it locally, no tunnel needed** (long polling, same code, local KV file):
   ```bash
   deno task dev
   ```
   Message your bot from the owner account and send `/start`.

## Deploy to Deno Deploy

Use the current Deno Deploy at [console.deno.com](https://console.deno.com)
(Deploy Classic at dash.deno.com has shut down).

1. Push this folder to a GitHub repo.
2. In the console, create an app from that repo. Set the **entrypoint to
   `main.ts`** in the dashboard (this repo deliberately has no `deploy` block in
   `deno.json`, because such a block overrides the dashboard entirely).
3. **Provision Deno KV:** open your organization -> **Databases** -> Provision
   Database -> Deno KV, then **Assign** it to the app. Without this,
   `Deno.openKv()` fails at startup.
4. Add environment variables in the app settings: `BOT_TOKEN`, `OWNER_ID`,
   `WEBHOOK_SECRET`, `TZ_OFFSET_MINUTES`. Generate the secret with
   `openssl rand -hex 24`.
5. Deploy, then put the app URL in your local `.env` as `PUBLIC_URL`. The
   `WEBHOOK_SECRET` in `.env` **must equal** the one in the dashboard. Register:
   ```bash
   deno task webhook:set      # this is the bot's permanent address
   deno task webhook:info     # verify; shows any delivery errors
   ```
6. Send `/start` to the bot.

To go back to local polling: `deno task webhook:delete`, then `deno task dev`.

Notes:
- `Deno.cron` runs on the new Deploy (see the **Cron** tab for run history). If it
  is ever unavailable on your plan, point an external pinger at
  `GET /cron/<WEBHOOK_SECRET>` every minute; it runs the same code.
- Deno KV *queues* are not supported on the new Deploy and are not used here.
- KV data from Deploy Classic is not migrated automatically.
- Local `deno task dev` uses a local KV file, separate from production data.

## Using it

**Owner** (`/start` or `/menu`): Publish, Schedule, Buttons, Destinations,
Access, Knowledge, Broadcast, Settings.

- *Connect a channel:* make the bot an admin; it registers itself and DMs you.
- *Connect a group:* make the bot an admin, then send `/addhere` in the group.
- *Schedule times:* `2026-10-01 10:00` (in your `TZ_OFFSET_MINUTES` timezone) or
  `+30m`, `+2h`, `+1d`.
- *Compose flow:* send text -> add buttons (`/done` when finished, `/skip` for none)
  -> tick destinations -> Done.

**Everyone else:** private messages start an access request the owner approves or
declines. Approved users get answers from the Knowledge base. In groups the bot
stays silent unless the group is a connected destination **and** someone writes
`@YourBot ...`.

## Reliability

- Scheduled posts are claimed atomically; if every destination fails they are
  re-queued (3 attempts, 5 minutes apart) instead of being lost.
- Broadcasts are queued and sent by the cron tick, never inside a webhook
  request, so a large audience cannot cause Telegram to retry and duplicate.

## Privacy model

- Persistent: destinations, posts and their message ids, buttons, schedules, access
  decisions, knowledge entries.
- Ephemeral: multi-step workflow state expires from KV after 1 hour (`expireIn`).
- Ordinary conversations are never stored. Activity log entries hold *what operation
  happened*, not message content, and expire after 30 days.

## Layout

```
main.ts                 serverless entry: webhook route + Deno.cron
src/bot.ts              owner control center, workflows, user + group handling
src/store.ts            Deno KV storage layer (all keys documented inline)
src/features/           publish, buttons, schedule, broadcast, assistant
src/dev-polling.ts      local dev runner (long polling)
scripts/                set / inspect / delete the Telegram webhook
docs/                   the two spec documents this was built from
```

## Security notes

- The webhook path contains `WEBHOOK_SECRET` **and** Telegram sends it back as the
  `X-Telegram-Bot-Api-Secret-Token` header; requests without both are rejected.
- Owner identity is your numeric Telegram id (`OWNER_ID`), never a username.

## Extending

The workflow doc also names Support tickets, Tasks, Polls and Approvals flows.
They aren't built. Each would follow the pattern used here: a store module +
a feature module + a menu entry in `src/bot.ts` and `src/keyboards.ts`
.
