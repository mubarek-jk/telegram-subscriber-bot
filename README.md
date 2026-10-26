# Telegram Daily Subscriber Tracker

Track daily channel joins, departures and repeat joins in MongoDB. Export a selected date as CSV or Excel, optionally showing only returning users or users still subscribed. Management happens in the bot's private chat and is restricted to configured admin IDs.

## What is included

- Node.js 24, TypeScript, grammY, MongoDB transactions and persistent event history.
- Daily reports in `Africa/Addis_Ababa` by default, with a configurable IANA timezone.
- One row per user per date, plus separate repeat-event counts.
- CSV and Excel exports, split into parts under the upload cap.
- Durable MongoDB inbox before Telegram acknowledgement, recovery after restarts, deduplication, and a worker lease to prevent overlapping polling instances during deployment.
- Render background-worker Blueprint, Dockerfile and local Compose replica set.

## Before deploying

You need:

1. A Telegram bot token from [@BotFather](https://t.me/BotFather).
2. Your channel's numeric ID, starting with `-100`.
3. The numeric Telegram user IDs allowed to manage the bot.
4. A MongoDB Atlas cluster or another authenticated MongoDB replica set. A standalone MongoDB server is insufficient because this application uses transactions.
5. A Git repository connected to Render and a Render account with a paid background worker available.

Use a dedicated bot and database for this channel. Do not run another polling client with the same token. The configured database is for one bot/channel pair; do not repoint an existing database at a different channel or bot.

## Telegram setup

1. Send `/newbot` to BotFather and save the token privately.
2. In your channel, add the bot as an administrator. The tracker needs administrator status to receive `chat_member` updates. Posting, deletion and banning permissions are not required for tracking.
3. Open the bot privately from each admin account and press **Start**.
4. Obtain the channel and admin IDs. You can use Telegram's API or the included helper below.

### ID helper

Install dependencies and place only `BOT_TOKEN` in `.env` initially:

```bash
npm ci
npm run inspect-ids
```

While the helper is waiting, send `/start` privately to your bot and add the bot to your channel as an administrator. If it was already added, change one administrator permission to generate a new status event. The helper prints pending private sender IDs and channel IDs, without dumping full Telegram updates.

Only your own ID should go in `ADMIN_USER_IDS`. Run this helper before starting the worker. It does not pass an offset, so it does not acknowledge pending updates. If there are many older updates, use another trusted way to obtain your IDs rather than dropping pending events.

## Configuration

Copy `.env.example` to `.env` for local use. On Render, enter values in its Environment settings.

| Variable | Purpose |
| --- | --- |
| `BOT_TOKEN` | BotFather token, kept secret |
| `MONGODB_URI` | MongoDB replica-set/Atlas URI, kept secret |
| `MONGODB_DB` | Dedicated database name, default `subscriber_tracker` |
| `CHANNEL_ID` | Numeric target channel ID |
| `ADMIN_USER_IDS` | Comma-separated numeric admin user IDs |
| `REPORT_TIMEZONE` | Default `Africa/Addis_Ababa` |
| `EXPORT_MAX_BYTES` | Default `45000000`, conservative file-size cap |

Passwords containing special characters must be URL-encoded in MongoDB connection strings. Never commit `.env` or put secrets in `render.yaml`.

## Deploy on Render

The included `render.yaml` provisions **one paid background worker** using the `0.5c-512mb` plan in Frankfurt. This plan is listed at approximately **USD 7/month** at the time of preparation; verify the price shown by Render before creation. MongoDB hosting and any additional usage are separate. A free sleeping web service is unsuitable for this polling worker.

1. Create an Atlas cluster and a database user scoped to `readWrite` on your dedicated database. Obtain the driver's `mongodb+srv://` connection string.
2. Push this project, including `package-lock.json` and `render.yaml`, to your Git repository. Keep secrets out of the repository.
3. In Render, choose **New → Blueprint**, connect the repository, and review the worker configuration and cost.
4. Set `BOT_TOKEN`, `MONGODB_URI`, `CHANNEL_ID`, and `ADMIN_USER_IDS` when prompted. Review the timezone.
5. Add the worker's Render outbound IP ranges to the Atlas project IP access list. Obtain the actual ranges from Render's dashboard. Avoid opening the database to every IP address.
6. Deploy. If the initial startup could not reach Atlas before the IP ranges were allowed, restart the worker once networking is configured.
7. Verify the logs show `Tracker running` and send `/today` privately to the bot.
8. Have a test account join, leave and rejoin. Check `/today`, then `/export_today` in both formats.

If creating the worker manually instead of using a Blueprint:

- Service type: **Background Worker**, not Web Service.
- Runtime: Node, version `24.22.1`.
- Build command: `npm ci --include=dev && npm run build && npm prune --omit=dev`.
- Start command: `npm start`.
- Instance count: **1**.
- Add the environment variables listed above and set `MONGOMS_DISABLE_POSTINSTALL=1` during builds to avoid downloading test-only MongoDB binaries.

MongoDB is external to the Render worker. The worker does not require a persistent Render disk; database records persist in Atlas. Export files are temporary and removed after sending.

If your bot already has a webhook, startup stops instead of silently changing it. Remove the webhook deliberately through Telegram's `deleteWebhook` method with `drop_pending_updates=false`, then restart.

## Run locally

With an existing reachable Atlas/replica-set database:

```bash
npm ci
npm run build
npm start
```

For development:

```bash
npm run dev
```

### Local Docker Compose

Create `.env` with real bot/channel/admin values, then:

```bash
docker compose up --build -d
docker compose logs -f bot
```

Compose starts a private MongoDB replica set, initializes it, and starts the bot. The `mongo-data` volume preserves data across container recreation. MongoDB has no published host port. This unauthenticated database configuration is for a private local development environment only; use authenticated Atlas for Render.

Stop without deleting data:

```bash
docker compose down
```

Do not use `docker compose down -v` unless you intend to delete the database volume.

## Commands

| Command | Result |
| --- | --- |
| `/today` | Today's subscriber statistics |
| `/date 2026-10-26` | Statistics for that local calendar date |
| `/export_today` | Export today's joining users |
| `/export_date 2026-10-26` | Export users joining on that date |
| `/stats` | Overall tracked statistics and queued update count |
| `/export` | All tracked subscriber records |
| `/help` | Instructions and limitations |

Daily exports ask for membership status, join category and format. “Still subscribed” uses the latest processed status **at export time**, including for historical dates. It is not an end-of-day historical snapshot.

Daily categories are mutually exclusive, based on the user's first join event on that day. Someone first observed joining and then returning on the same day appears in the first-observed category, with the later return recorded in their return-event count. Someone whose first event of the day is a return appears in the returning category. The category filters select these daily categories, not everyone who had any return event that day.

## Database and recovery

Collections:

- `subscribers`: one document per channel/user, current observed status and lifetime counters.
- `membership_events`: immutable join/departure/status events with original UTC Telegram timestamps and event-time join classification.
- `inbox`: pending relevant Telegram updates, removed after successful processing.
- `polling_state`: committed fetch offset and ingestion sequence.
- `worker_leases`: active worker ownership and expiry.

The polling loop first commits each batch and its cursor to MongoDB with majority write concern. Only the next `getUpdates` call acknowledges that batch to Telegram. A separate sequential processor consumes the saved inbox. Subscriber changes, event insertion and inbox deletion occur in one transaction. On failure they roll back together, and the inbox item remains available for retry.

During long exports, fetching continues into the database; membership processing resumes after the export. Reports therefore describe the latest **processed** events, not updates still waiting in the inbox. Membership counts are deduplicated. Admin replies and file deliveries are at least once and may repeat if a crash happens immediately after Telegram accepts a message but before the inbox item is completed. Rerun a failed export command if needed.

A lease heartbeat fences database writes from a stale worker. On deployment, a new worker may wait up to approximately 60 seconds for the previous lease to expire. Keep Render at one instance. On a database connection/lease failure, the worker stops so the platform can restart it. Do not reset `polling_state` or delete `membership_events` to resolve routine errors.

Startup and operational logs deliberately omit raw exceptions because Telegram HTTP errors can embed tokens. If startup fails, check the bot's admin status, existing webhook, environment values, database connectivity, replica-set support and competing polling clients.

### Backup and restore

Use Atlas managed backups where your plan supports them. Otherwise use MongoDB Database Tools from a trusted machine. Prompting for the URI avoids putting it in shell history; do not paste it into logs or shared commands.

```bash
read -r -s -p 'MongoDB URI: ' TRACKER_BACKUP_URI
export TRACKER_BACKUP_URI
mongodump --uri="$TRACKER_BACKUP_URI" --db=subscriber_tracker --archive=tracker-backup.archive --gzip
unset TRACKER_BACKUP_URI
```

For a consistent manual backup of the full application state, pause the worker first, back up all collections, then resume promptly. Protect and encrypt the archive. A paused worker will not fetch events, and Telegram's retention window still applies.

Restore into an empty database with the worker stopped:

```bash
read -r -s -p 'MongoDB URI: ' TRACKER_RESTORE_URI
export TRACKER_RESTORE_URI
mongorestore --uri="$TRACKER_RESTORE_URI" --archive=tracker-backup.archive --gzip
unset TRACKER_RESTORE_URI
```

An old backup can lose already-acknowledged activity that Telegram no longer retains. Restoring it cannot reconstruct that missing activity. Do not merge old summaries with newer events or manually decrement the polling offset.

## Tracking limitations

- Telegram's standard Bot API does not list all existing channel subscribers.
- No join/departure history from before tracking can be reconstructed.
- An existing subscriber may first become known through a departure or privilege change.
- “First observed join” does not prove first-ever membership.
- Pending updates are retained by Telegram for no more than **24 hours**. Extended downtime before fetching can cause permanent gaps. Updates already in MongoDB's inbox survive worker restarts.
- Current tracked membership may differ from Telegram's total channel subscriber count.
- Multiple accounts may belong to one person; this bot counts Telegram user IDs, not verified people.
- Names and usernames reflect the latest processed membership event, not continuous profile monitoring.
- The reporting timezone is applied at query time. Changing it changes which daily report contains an event; stored UTC history remains unchanged.

## Validation

```bash
npm run check
npm run build
npm run test:integration
npm audit --omit=dev
```

`npm run check` runs strict type checking and tests for membership classification, date boundaries, authorization, export menus, CSV/Excel correctness, file splitting and cleanup. Integration tests start a temporary real MongoDB replica set using `mongodb-memory-server`. They require permission to start local processes/listening sockets and download the MongoDB binary on first run. They cover durable cursor/inbox recovery, deduplication, transaction rollback, daily aggregation and filtering, and worker fencing.

Set `MONGODB_TEST_URI` to use an existing dedicated test replica set instead of a local temporary server. The integration suite creates randomly named test databases and removes only those databases afterward. Use a test account with permissions to create and drop test databases.

The Excel dependency's UUID implementation is overridden to the patched CommonJS-compatible version `11.1.1`. The lockfile pins dependencies. Run checks after dependency upgrades.

Actual Telegram delivery and Render deployment require your configured accounts and credentials. Complete the live join/leave/rejoin smoke test after deployment.

## Official references

- https://core.telegram.org/bots/api#chatmemberupdated
- https://core.telegram.org/bots/api#getupdates
- https://core.telegram.org/bots/faq
- https://grammy.dev/guide/
- https://www.mongodb.com/docs/drivers/node/current/crud/transactions/
- https://www.mongodb.com/docs/atlas/security/ip-access-list/
- https://render.com/docs/background-workers
- https://render.com/docs/blueprint-spec
- https://render.com/pricing
