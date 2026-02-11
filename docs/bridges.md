# 🌉 Discord ↔ Matrix Bridging Guide

> *A portal between two cozy rooms — so nobody gets left behind during the move.* ✨

---

## 📖 Table of Contents

- [What Is a Bridge?](#-what-is-a-bridge)
- [Architecture](#-architecture)
- [Bridge Modes](#-bridge-modes)
- [What Gets Bridged?](#-what-gets-bridged)
- [Setting Up mautrix-discord](#%EF%B8%8F-setting-up-mautrix-discord)
- [Homeserver Configuration](#-homeserver-configuration)
- [Security Considerations](#-security-considerations)
- [Migration Strategy](#-migration-strategy)
- [Troubleshooting](#-troubleshooting)
- [Resources](#-resources)

---

## 🤔 What Is a Bridge?

A **bridge** is a service that relays messages, media, reactions, and events between two different chat platforms in real time. Users on either side see messages as if everyone is in the same room.

Matrix bridges are built on the [Application Service API](https://spec.matrix.org/latest/application-service-api/) — a privileged integration layer that lets the bridge create and manage virtual users, join rooms, and relay events on behalf of remote platform users.

The recommended bridge for Discord is [**mautrix-discord**](https://github.com/mautrix/discord), maintained by the mautrix team (the same folks behind many of Matrix's most reliable bridges).

---

## 🏗 Architecture

```
┌──────────────────┐                                    ┌──────────────────┐
│                  │                                    │                  │
│  Matrix          │     ┌────────────────────────┐     │  Discord         │
│  Homeserver      │◄───►│   mautrix-discord       │◄───►│  API             │
│  (Continuwuity)  │     │   (bridge service)      │     │  (Bot Gateway)   │
│                  │     └───────────┬────────────┘     │                  │
└──────────────────┘                │                   └──────────────────┘
                             ┌──────┴──────┐
                             │  Database   │
                             │             │
                             │ • Room maps │
                             │ • User maps │
                             │ • Msg IDs   │
                             │ • Puppets   │
                             └─────────────┘
```

### How It Works

1. **Discord → Matrix:** The bridge listens to Discord events via a bot token. When someone sends a message in a bridged Discord channel, the bridge converts it (markdown, mentions, emoji, attachments) and posts it into the corresponding Matrix room — either as a bot or as a "puppet" user representing the Discord sender.

2. **Matrix → Discord:** The bridge watches Matrix rooms via the Application Service API. When a Matrix user sends a message, the bridge converts it and posts it to Discord via the bot API (or the user's puppet token).

3. **Mapping Database:** A PostgreSQL or SQLite database tracks which Discord channels map to which Matrix rooms, which Discord users map to which virtual Matrix users, and which message IDs correspond across platforms (so edits, deletes, and reactions can find the right target).

---

## 🎭 Bridge Modes

### Relay / Bot Mode

A single Discord bot relays all messages. Matrix users' display names are prefixed to each message.

**Discord sees:**
```
NeoBot: [PixelKougra] Hey everyone, who wants to trade?
NeoBot: [GlitterAisha] Me! I have a Rainbow Paint Brush ✨
```
**Pros:** Simple setup, no user auth needed
**Cons:** All messages come from one bot — less personal, can look spammy

### Puppet Mode (Double Puppeting)

Users link their Discord account to the bridge. Messages appear *as themselves* on both platforms — no bot prefix, no indication it's bridged.

**Discord sees:**
```
PixelKougra: Hey everyone, who wants to trade?
GlitterAisha: Me! I have a Rainbow Paint Brush ✨
```
**Pros:** Seamless experience, messages look native
**Cons:** Each user must authenticate their Discord account with the bridge

---

## 📬 What Gets Bridged?

| Feature | Discord → Matrix | Matrix → Discord | Notes |
|---|---|---|---|
| **Text messages** | ✅ | ✅ | Markdown, mentions, formatting converted |
| **Images / files** | ✅ | ✅ | Re-uploaded across platforms |
| **GIFs** | ✅ | ✅ | Sent as animated images |
| **Message edits** | ✅ | ✅ | Edit synced to the other side |
| **Message deletes** | ✅ | ✅ | Delete synced to the other side |
| **Reactions (emoji)** | ✅ | ✅ | Standard and custom emoji |
| **Typing indicators** | ✅ | ✅ | If enabled in config |
| **Reply threads** | ⚠️ Partial | ⚠️ Partial | Basic threading; not a 1:1 match |
| **Embeds** | ⚠️ Partial | — | Converted to plain text/links on Matrix |
| **Stickers** | ⚠️ Partial | — | Sent as images |
| **Voice / video calls** | ❌ | ❌ | Fundamentally different tech (WebRTC vs. proprietary) |
| **Discord slash commands** | ❌ | — | Discord-only feature |

---

## ⚙️ Setting Up mautrix-discord

### Prerequisites

| Requirement | Details |
|---|---|
| **Matrix homeserver** | Continuwuity, Synapse, or Dendrite with Application Service support |
| **Discord bot** | Created at [Discord Developer Portal](https://discord.com/developers/applications) |
| **Database** | PostgreSQL (recommended) or SQLite |
| **Runtime** | Docker (recommended) or Go 1.21+ |

### Step 1: Create a Discord Bot

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications)
2. Click **"New Application"** → name it (e.g., `NeoClubBridge`)
3. Go to **Bot** → click **"Add Bot"**
4. Copy the **Bot Token** (keep this secret!)
5. Enable these **Privileged Gateway Intents:**
   - ✅ Message Content Intent
   - ✅ Server Members Intent
   - ✅ Presence Intent
6. Go to **OAuth2 → URL Generator:**
   - Scopes: `bot`
   - Bot Permissions: `Send Messages`, `Read Message History`, `Manage Webhooks`, `Add Reactions`, `Attach Files`, `Embed Links`
7. Use the generated URL to invite the bot to your Discord server

### Step 2: Deploy mautrix-discord

**Docker (recommended):**

```yaml
# docker-compose.yml
version: "3"
services:
  mautrix-discord:
    container_name: mautrix-discord
    image: dock.mau.dev/mautrix/discord:latest
    restart: unless-stopped
    volumes:
      - ./mautrix-discord:/data
    depends_on:
      - db

  db:
    container_name: mautrix-discord-db
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: mautrix_discord
      POSTGRES_PASSWORD: your-secure-db-password
      POSTGRES_DB: mautrix_discord
    volumes:
      - ./mautrix-discord-db:/var/lib/postgresql/data
```

**Generate the config:**

```bash
docker compose run --rm mautrix-discord
# This creates a config.yaml in ./mautrix-discord/
# Edit it, then restart
```

### Step 3: Configure the Bridge

Edit `./mautrix-discord/config.yaml`:

```yaml
# Homeserver connection
homeserver:
    address: https://matrix.neopets.club
    domain: neopets.club

# Application Service settings (auto-generated registration file)
appservice:
    address: http://mautrix-discord:29334
    hostname: 0.0.0.0
    port: 29334
    database:
        type: postgres
        uri: postgres://mautrix_discord:your-secure-db-password@db/mautrix_discord?sslmode=disable

# Bridge settings
bridge:
    # Who can use the bridge
    permissions:
        "*": relay
        "neopets.club": user
        "@admin:neopets.club": admin

    # Relay mode settings
    relay:
        enabled: true
        message_formats:
            m.text: "<b>{{ .Sender.Displayname }}</b>: {{ .Message }}"
            m.notice: "<b>{{ .Sender.Displayname }}</b>: {{ .Message }}"
            m.emote: "* <b>{{ .Sender.Displayname }}</b> {{ .Message }}"
```

### Step 4: Register with Your Homeserver

The bridge generates a `registration.yaml` file. This needs to be registered with Continuwuity.

In your Continuwuity config (from the [appservice configuration](https://github.com/continuwuity/continuwuity/blob/f5f3108d5f97d24535643a114f646bde4b136ea3/conduwuit-example.toml#L371-L469)):

```toml
# Point to the bridge's registration file
appservice_timeout = 35
appservice_idle_timeout = 300
```

Place the `registration.yaml` in Continuwuity's appservice directory and restart both services.

### Step 5: Start Bridging!

```bash
docker compose up -d
```

In Matrix, DM the bridge bot (`@discordbot:neopets.club`) and use:

```
# Login with your Discord token (puppet mode)
login-token YOUR_DISCORD_TOKEN

# Or use the bridge in relay mode — just invite the bot to rooms
# and link them to Discord channels
```

---

## 🔐 Homeserver Configuration

Continuwuity handles Application Service connections natively. Key config options from the [example config](https://github.com/continuwuity/continuwuity/blob/f5f3108d5f97d24535643a114f646bde4b136ea3/conduwuit-example.toml#L371-L469):

```toml
# Appservice URL request connection timeout (seconds)
# Appservices are usually on the same network, so 35s is plenty
appservice_timeout = 35

# Appservice idle connection pool timeout (seconds)
appservice_idle_timeout = 300
```

The Application Service API allows the bridge to:
- Create virtual ("ghost") users for each Discord member
- Join/leave rooms on behalf of those users
- Send and receive events in bridged rooms
- Manage room state (names, avatars, topics)

For reference, here's how Continuwuity handles the TURN secret for the bridge's associated voice features — from [`src/service/globals/mod.rs`](https://github.com/continuwuity/continuwuity/blob/f5f3108d5f97d24535643a114f646bde4b136ea3/src/service/globals/mod.rs#L27-L118):

```rust
let turn_secret = config.turn_secret_file.as_ref().map_or_else(
    || config.turn_secret.clone(),
    |path| match std::fs::read_to_string(path) {
        | Ok(secret) => secret.trim().to_owned(),
        | Err(e) => {
            error!("Failed to read the TURN secret file: {e}");
            config.turn_secret.clone()
        },
    },
);
```

---

## 🛡 Security Considerations

### What the Bridge Can See

| Data | Accessible? | Notes |
|---|---|---|
| **Unencrypted Matrix messages** | ✅ Yes | The bridge must read messages to relay them |
| **E2EE Matrix messages** | ⚠️ Only if configured | Bridge needs encryption keys to decrypt — adds complexity |
| **Discord messages** | ✅ Yes | Via the bot token or puppet tokens |
| **Discord tokens (puppet mode)** | ✅ Yes | Stored in the bridge database — protect it! |

### Best Practices

- **🔒 Protect the bridge database** — it contains Discord tokens if using puppet mode. Encrypt at rest.
- **🔑 Use a dedicated Discord bot account** — don't use your personal Discord token for the bot.
- **🚪 Limit bridge permissions** — only give `admin` to trusted Matrix users in the bridge config.
- **📝 Only bridge public/semi-public rooms** — don't bridge E2EE rooms unless you understand the implications (the bridge effectively becomes a participant that can read messages).
- **🕐 Plan to retire the bridge** — it's a migration tool, not a permanent fixture. The longer it runs, the less incentive people have to switch.
- **🔄 Keep it updated** — `docker compose pull && docker compose up -d` regularly.

### E2EE Rooms & Bridging

Bridging an E2EE room means the bridge bot must be a trusted member with access to encryption keys. This effectively breaks the "only participants can read messages" guarantee, since the bridge relays content to Discord (which has no E2EE for text). **We recommend only bridging unencrypted public rooms.**

---

## 🦋 Migration Strategy

Here's a cozy, low-pressure migration plan:

### Phase 1: 🌱 Soft Launch (Weeks 1–2)
- Set up Matrix server and bridge
- Invite early adopters / mods to test
- Bridge 2–3 key Discord channels (e.g., `#general`, `#trading`, `#art`)
- Let people explore at their own pace

### Phase 2: 🌿 Grow the Garden (Weeks 3–4)
- Announce the move to the wider community
- Share the [main README guide](../README.md) for joining
- Bridge remaining active channels
- Start posting announcements on Matrix first, mirrored to Discord

### Phase 3: 🌸 Bloom (Weeks 5–8)
- Most activity happens on Matrix
- Discord becomes the "read-only mirror" — new conversations happen on Matrix
- Help stragglers migrate; offer 1:1 setup help if needed

### Phase 4: 🏡 Settle In (Week 9+)
- Retire the bridge
- Archive or close the Discord server (or keep it as a read-only archive)
- Celebrate! You're home. 🐾✨

> *"A migration isn't a sprint — it's a gentle stroll to a cozier spot."*

---

## 🔧 Troubleshooting

<details>
<summary><strong>Messages aren't relaying from Discord to Matrix</strong></summary>

- Check the bridge logs: `docker compose logs mautrix-discord`
- Verify the bot is in the Discord channel and has `Read Message History` + `Send Messages` permissions
- Confirm the Application Service registration is loaded by Continuwuity (restart if needed)
- Make sure `Message Content Intent` is enabled in the Discord Developer Portal

</details>

<details>
<summary><strong>Messages aren't relaying from Matrix to Discord</strong></summary>

- Check that the bridge bot is joined to the Matrix room
- Verify the room is linked to a Discord channel (`!discord link` or check bridge status)
- Check bridge permissions in `config.yaml` — the user's homeserver needs at least `user` level
- Check Discord bot permissions — needs `Send Messages`, `Manage Webhooks`

</details>

<details>
<summary><strong>Images/files aren't bridging</strong></summary>

- Check that the Discord bot has `Attach Files` and `Embed Links` permissions
- Verify your Matrix homeserver's media upload limits aren't being hit
- Large files may fail silently — check bridge logs for errors

</details>

<details>
<summary><strong>Puppet mode: "invalid token" error</strong></summary>

- Discord tokens expire or get rotated — re-authenticate with `login-token`
- Make sure you're using a user token, not a bot token, for puppet mode
- Check that your Discord account hasn't been flagged for automation

</details>

<details>
<summary><strong>Bridge seems slow or laggy</strong></summary>

- Check database performance (PostgreSQL recommended over SQLite for production)
- Monitor the bridge's memory/CPU usage
- Discord rate limits may be throttling the bot — check logs for 429 errors
- Consider increasing `appservice_timeout` in Continuwuity config if the bridge is slow to respond

</details>

---

## 📚 Resources

| Resource | Link |
|---|---|
| **mautrix-discord** | [GitHub](https://github.com/mautrix/discord) |
| **mautrix-discord docs** | [docs.mau.fi/bridges/go/discord](https://docs.mau.fi/bridges/go/discord/) |
| **Matrix Application Service spec** | [spec.matrix.org](https://spec.matrix.org/latest/application-service-api/) |
| **All mautrix bridges** | [docs.mau.fi/bridges](https://docs.mau.fi/bridges/) |
| **Discord Developer Portal** | [discord.com/developers](https://discord.com/developers/applications) |
| **Continuwuity appservice config** | [conduwuit-example.toml](https://github.com/continuwuity/continuwuity/blob/f5f3108d5f97d24535643a114f646bde4b136ea3/conduwuit-example.toml#L371-L469) |
| **Continuwuity docs** | [continuwuity.org](https://continuwuity.org/introduction) |

---

> *The bridge is just a doorway. The destination is a cozier, safer home — and it's already waiting for you.* 🏡🐾