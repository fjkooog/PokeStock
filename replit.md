# PokéStock Alerts

A Node.js and TypeScript Discord application that searches Pokémon TCG inventory, manages member watches, and prepares deduplicated restock alerts.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — start the development server
- `pnpm --filter @workspace/api-server run typecheck` — typecheck the bot server
- `pnpm --filter @workspace/api-server run test` — run unit tests
- `pnpm run typecheck` — full workspace typecheck
- `pnpm run build` — full workspace build

The server binds to `PORT` and exposes the local inspection API under `/api`. It defaults to a local SQLite database at `data/pokestock.sqlite`.

## Environment

Copy `.env.example` when running independently. Never commit `.env` or Discord credentials.

- `DISCORD_TOKEN` — bot token used for command registration and alert messages
- `DISCORD_CLIENT_ID` — Discord application ID
- `DISCORD_PUBLIC_KEY` — public key used to verify interaction webhooks
- `DISCORD_GUILD_ID` — development Discord server ID used for fast guild command registration
- `DATABASE_URL` — optional `file:` SQLite path in phase one
- `MOCK_INVENTORY` — keep `true` until a permitted live provider is implemented

## Where things live

- `artifacts/api-server/src/pokestock/types.ts` — provider and inventory contracts
- `artifacts/api-server/src/pokestock/providers/` — retailer catalog, stubs, and explicitly marked mock feed
- `artifacts/api-server/src/pokestock/inventory/` — normalization, search engine, and change detection
- `artifacts/api-server/src/pokestock/storage/` — SQLite schema and repository boundary
- `artifacts/api-server/src/pokestock/discord/` — slash commands, signature verification, REST calls, and alerts
- `artifacts/api-server/src/routes/pokestock.ts` — Discord webhook and development inspection routes

## Architecture decisions

- Retailer-specific logic is behind `RetailerProvider`; adding a permitted feed does not change the inventory engine.
- Phase one uses Node's built-in SQLite support to avoid a native dependency and keep local development free. The repository interface leaves room for a PostgreSQL/Supabase adapter.
- Mock data is a separate provider mode and is labeled in command output, API responses, and alert footers.
- Discord interactions are handled through the current HTTP interaction API with Ed25519 signature verification. No Discord gateway is required for slash commands.
- Alert history uses a unique event/channel key so the same restock is not repeatedly posted to a destination.

## Product

Available now: `/stock`, `/stores`, `/watch`, `/unwatch`, `/watchlist`, `/deals`, `/product`, `/setlocation`, `/alerts`, `/help`, `/pokestock-setup`, `/pokestock-status`, `/test-restock`, and `/simulate-restock`; provider registry; local inventory persistence; mock search; inventory transition detection; Discord alert publishing; and a polling scheduler.

Not yet live: retailer inventory integrations and production PostgreSQL migration. All 21 retailer entries are present, with explicit MOCK or STUB status.

## Gotchas

- `DISCORD_PUBLIC_KEY` is required for `/api/discord/interactions`; the server intentionally returns 503 when Discord is not configured rather than accepting unsigned requests.
- The first-phase `DATABASE_URL` adapter supports SQLite paths and `file:` URLs only. Do not point it at PostgreSQL until the PostgreSQL adapter is added.
- Do not reverse-engineer retailer sites or bypass protections. Each live provider must use a permitted official/public source.

## Connect a Discord application

Phase two is designed for testing in one real Discord server while every inventory provider remains MOCK or STUB.

### 1. Create the Discord application

1. Open the [Discord Developer Portal](https://discord.com/developers/applications).
2. Select **New Application**, give it a name such as `PokéStock Alerts`, and create it.
3. On **General Information**, copy **Application ID** into `DISCORD_CLIENT_ID`.
4. On the same page, copy **Public Key** into `DISCORD_PUBLIC_KEY`.
5. Open **Bot**, select **Add Bot**, then use **Reset Token** and copy the token into the `DISCORD_TOKEN` environment secret. Never put it in source code or commit it.
6. Copy the ID of the Discord server used for testing into `DISCORD_GUILD_ID`. Enable Developer Mode in Discord, right-click the server, and select **Copy Server ID**.

### 2. Use the minimum permissions

The bot only needs to read the configured channel and post embeds:

- **View Channel**
- **Send Messages**
- **Embed Links**

The corresponding permission integer is `19456`. Do not grant Administrator, Manage Server, or Read Message History to the bot.

When installing the app, request these OAuth2 scopes:

- `bot`
- `applications.commands`

The administrator using `/pokestock-setup`, `/pokestock-status`, `/test-restock`, and `/simulate-restock` needs the Discord **Manage Server** permission. The bot itself does not need that permission.

### 3. Install the application into the server

In the Developer Portal:

1. Open **OAuth2 → URL Generator**.
2. Select the `bot` and `applications.commands` scopes.
3. Select View Channel, Send Messages, and Embed Links under Bot Permissions.
4. Open the generated URL, choose the test server, and authorize the app.
5. Confirm the bot can see and send messages in the channel you will configure.

### 4. Configure the Interaction Endpoint URL

Discord needs a publicly reachable HTTPS URL. Set:

```text
https://YOUR_PUBLIC_HOST/api/discord/interactions
```

The exact application path is `/api/discord/interactions`. Replace `YOUR_PUBLIC_HOST` with the published HTTPS host for this project. Do not use `localhost` or an unsigned development URL in the Discord portal.

Click **Save Changes**. Discord sends a signed PING request to this endpoint. The application verifies the Ed25519 signature using `DISCORD_PUBLIC_KEY` and responds with Discord interaction type `1`. An invalid signature is rejected.

### 5. Register development slash commands

Make sure `DISCORD_TOKEN`, `DISCORD_CLIENT_ID`, and `DISCORD_GUILD_ID` are available as environment secrets, then run:

```bash
pnpm --filter @workspace/api-server run register:dev
```

This uses Discord's guild command endpoint:

```text
PUT /applications/{DISCORD_CLIENT_ID}/guilds/{DISCORD_GUILD_ID}/commands
```

Guild commands normally appear quickly. The running server also registers commands automatically; when `DISCORD_GUILD_ID` is set it uses guild registration, otherwise it uses global registration.

### 6. Configure and test the alert channel

In the test server:

1. Run `/pokestock-setup` and select the channel where alerts should go.
2. Run `/pokestock-status` to verify the configured channel, SQLite connection, Discord configuration, and polling timestamps.
3. Run `/test-restock` to post a realistic MOCK alert. It intentionally has no purchase link.
4. Run `/simulate-restock`, select a mock retailer and product, and verify:
   - one `OUT OF STOCK → IN STOCK` event is generated
   - the alert pipeline runs
   - the duplicate verification cycle generates zero additional events

Every status screen and alert includes a clear TEST or MOCK label. No retailer is represented as LIVE.