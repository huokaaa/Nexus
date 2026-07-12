# Plugin Development Guide — Nexus

This document is the single source of truth for developing plugins in Nexus. It describes the plugin structure, the APIs exposed through `run(m, ctx)`, and the helper libraries available under `lib/`.

Every new plugin should follow this guide to ensure consistency across the project.

> Status: Living document. Whenever new APIs are added to `lib/`, this document should be updated accordingly.

---

## 1. Plugin Structure

Plugins are located inside:

```
plugins/<category>/<plugin>.js
```

or directly under:

```
plugins/<plugin>.js
```

(for example, `menu.js`).

The plugin loader supports hot reload, so changes are detected automatically during development without restarting the bot.

Files prefixed with `_` are considered internal plugins. These are usually:

- Event listeners that don't expose commands.
- Registry files exporting multiple plugins.
- Internal helpers.

This is only a naming convention. The loader still imports every `.js` file under `plugins/`.

Example:

```js
import { createFrame } from '../../lib/Format.js'

export default {
   command: 'ping',
   hidden: ['p'],
   category: 'other',

   group: false,
   private: false,
   owner: false,
   admin: false,
   botAdmin: false,
   partner: false,
   premium: false,

   limit: false,
   energy: false,

   async run(m, ctx) {
      const start = Date.now()

      const text = createFrame()
         .header('Pong')
         .blank()
         .item(`Latency: ${Date.now() - start}ms`)
         .footer()
         .toString()

      await m.reply(text)
   }
}
```

Only `command` and `run` are required. All other fields are optional.

---

## 2. Plugin Properties

| Property | Type | Description |
|---|---|---|
| `command` | `string \| string[]` | Primary command name(s). |
| `hidden` | `string \| string[]` | Additional aliases that won't appear in the menu. |
| `category` | `string` | Menu category. |
| `group` | `boolean` | Only executable inside groups. |
| `private` | `boolean` | Only executable in private chats. |
| `owner` | `boolean` | Restrict to the bot owner. |
| `partner` | `boolean` | Restrict to partner/reseller accounts. |
| `premium` | `boolean` | Restrict to premium users. |
| `admin` | `boolean` | Require group admin privileges. |
| `botAdmin` | `boolean` | Require the bot to be a group admin. |
| `limit` | `boolean \| number` | Deduct user limit before execution. (`true` = 1) |
| `energy` | `boolean \| number` | Deduct user energy before execution. (`true` = 1) |
| `setCommand` | `boolean` | For plain-TEXT display plugins only (see §10) — auto-registers an owner-only `/set<command>` letting the owner customize the output via a template string. Don't use this on functional plugins (downloads, AI, etc). |
| `run(m, ctx)` | `Function` | Main plugin handler. |

If a plugin doesn't define `command`, it is automatically registered as an event listener.

---

## 3. Message Object (`m`)

The message object is created by `lib/Serialize.js`.

### Properties

| Property | Description |
|---|---|
| `m.chat` | Chat JID |
| `m.sender` | Sender JID |
| `m.pushName` | Display name |
| `m.isGroup` | Group chat |
| `m.isPrivate` | Private chat |
| `m.isStatus` | WhatsApp Status |
| `m.fromMe` | Sent by the bot's own session |
| `m.isMe` | **Not** a plain alias of `fromMe` — true only when `fromMe` is set AND the message ID itself matches the bot's own ID pattern. Used to distinguish messages the bot generated from messages merely sent from the same account (e.g. via a linked device). |
| `m.body` | Original message text |
| `m.prefix` | Command prefix |
| `m.command` | Parsed command |
| `m.text` | Message without prefix/command |
| `m.args` | Command arguments |
| `m.mentionedJid` | Mentioned users |
| `m.quoted` | Quoted message (if any) |
| `m.msg` | Raw Baileys message |

### Methods

| Method | Description |
|---|---|
| `m.reply(text, options?, extra?)` | Reply to the current message. |
| `m.react(reaction, extra?)` | Send a reaction. |
| `m.download()` | Download attached media as `Buffer`. |

Quoted messages expose the same API, including:

```js
m.quoted.download()
```

---

## 4. Context Object (`ctx`)

The second parameter passed into every plugin.

| Property | Description |
|---|---|
| `sock` | Baileys socket instance |
| `db` | Database instance |
| `store` | Cached chats, contacts and metadata |
| `user` | Mutable user database object |
| `group` | Mutable group database object |
| `setting` | Global bot settings |
| `groupMetadata` | WhatsApp group metadata |
| `isOwner` | Owner flag |
| `isPartner` | Partner flag |
| `isPremium` | Premium flag |
| `isBanned` | Banned flag |
| `isAdmin` | Group admin flag |
| `isBotAdmin` | Bot admin flag |
| `isPrefix` | Active prefix |
| `command` | Parsed command |
| `text` | Parsed text |
| `args` | Parsed arguments |
| `body` | Original message |

Relative import paths depend on the plugin location.

```
plugins/plugin.js        → ../lib/Format.js
plugins/tools/plugin.js  → ../../lib/Format.js
```

---

## 5. Socket Helpers

Common helper methods exposed through `ctx.sock`.

| Method | Signature | Description |
|---|---|---|
| `sendText` | `(jid, text, quoted?, content?, options?)` | Send plain text. |
| `sendMedia` | `(jid, source, caption?, quoted?, content?, options?)` | Send image, video, sticker, audio or document. `source` may be a Buffer, local path or URL. |
| `sendMessage` | `(jid, content, options?)` | Low-level Baileys wrapper. |

See `lib/Utilities.js` for the complete implementation.

---

## 6. Output Formatting (`lib/Format.js`)

Plugins should use `createFrame()` instead of manually constructing message boxes.

Example:

```js
import { createFrame } from '../../lib/Format.js'

const text = createFrame()
   .header('Title')
   .blank()
   .item('First item')
   .item('Second item')
   .divider('Another Section')
   .item('More content')
   .footer()
   .toString()

await m.reply(text)
```

Available methods:

| Method | Description |
|---|---|
| `header(title)` | Frame header |
| `blank()` | Empty line |
| `item(text)` | Bullet item with automatic wrapping |
| `raw(text)` | Plain wrapped line |
| `divider(label?)` | Section divider |
| `footer()` | Frame footer |
| `toString()` | Render final output |

Additional helpers:

```js
import {
   wrapLine,
   wrapWithPrefix,
   progressBar
} from '../../lib/Format.js'
```

**Design rule: no emoji, anywhere** (chat output or console logs). Only the box-drawing/geometric symbols already used by the frame system (`│ ╭ ╰ ├ ✦ ▸ ─ █ ░`) are allowed. The one established exception is `m.react('🕒')` as a "command received, processing" confirmation — that's a deliberate, singular convention, not an invitation to add more reaction emoji.

---

## 7. Logging (`lib/Logger.js`)

Use the built-in logger instead of direct `console.log()` calls.

```js
import { Logger } from '../../lib/Logger.js'

Logger.info('Processing request')
Logger.ok('Media sent successfully')
Logger.warn('Slow API response')
Logger.error('Failed to fetch data', error)
Logger.debug('Cache hit', key)
```

Available levels:

- `info`
- `ok`
- `warn`
- `error`
- `debug`

The logger automatically includes timestamps and ANSI colors (configurable through `consoleColors`).

---

## 8. Common Patterns

### Require quoted media

```js
const q = m.quoted || m

if (!q.msg?.mimetype) {
   return m.reply(
      createFrame()
         .header('Error')
         .blank()
         .item('Reply to an image, video or other media first.')
         .footer()
         .toString()
   )
}

const buffer = await q.download()

await ctx.sock.sendMedia(
   m.chat,
   buffer,
   '',
   m,
   { sticker: true }
)
```

### Validate arguments

```js
if (!m.text) {
   return m.reply(
      createFrame()
         .header('Usage')
         .blank()
         .item(`Example: ${ctx.isPrefix}${ctx.command} text`)
         .footer()
         .toString()
   )
}
```

### Error handling

Plugins interacting with external services should always use `try...catch`.

```js
async run(m, ctx) {
   try {
      // ...
   }
   catch (error) {
      Logger.error('Plugin failed', ctx.command, error)

      await m.reply(
         createFrame()
            .header('Error')
            .blank()
            .item(error.message)
            .footer()
            .toString()
      )
   }
}
```

### Avoid concurrent commands from the same sender

The bot already enforces a per-sender lock at the `Listener.js` level — a sender can't have two commands running at once (they'll get a "still processing" reply automatically). Plugins don't need to implement this themselves.

---

## 9. Customizable Text Plugins (`setCommand`)

For plugins that only display text/info with no real function — e.g. `credits` — set `setCommand: true` instead of building a separate `/setX` plugin by hand:

```js
import { buildNativeFlowItem, renderTemplate } from '../../lib/Template.js'

export default {
   command: 'credits',
   category: 'other',
   setCommand: true,
   async run(m, ctx) {
      const raw = ctx.setting.customTemplates?.credits

      const rendered = raw
         ? renderTemplate(raw, m, ctx)
         : { text: 'Default text here', hasThumbnail: false, buttons: [], mentions: [] }

      // ... send using rendered.text / rendered.hasThumbnail / rendered.buttons / rendered.mentions
   }
}
```

This automatically registers `/set<command>` (e.g. `/setcredits`) — owner-only, no extra file needed. The owner runs it with a template string using these tags:

| Tag | Meaning |
|---|---|
| `<thumbnail>` | Flag: attach `botThumbnail` as an image. Position doesn't matter, stripped from the text. |
| `<cta,type,text,isi>` | One interactive button. `type`: `url`, `call`, or `copy`. |
| `<user>` | @mention of whoever sent the command |
| `<pushname>` | Sender's display name |
| `<botname>` / `<ownername>` / `<ownernumber>` | Bot/owner identity |
| `<runtime>` | Process uptime |
| `<date>` / `<time>` / `<greeting>` | Current date/time/time-of-day greeting |
| `<prefix>` | Prefix used for this invocation |
| `<groupname>` | Current group's name (`-` in private chat) |
| `<totalusers>` / `<totalgroups>` | Registered user/group counts |
| `<version>` | Bot version from `package.json` |
| `<donate>` / `<github>` | Configured donate URL / GitHub repo |
| `<commandcount>` / `<plugincount>` | Currently loaded command/plugin counts |

`/set<command> reset` reverts to the plugin's own default output.

Do NOT use `setCommand` on plugins that perform an actual action (downloads, AI generation, stalking, etc.) — it's only for plain informational/text output.

## 10. Existing Categories

Refer to:

```
lib/Constants.js
```

Specifically:

- `CATEGORY_DESCRIPTIONS`
- `POPULAR_CATEGORIES`

Use existing categories whenever possible instead of introducing new ones unnecessarily.
