# bread-baileys

A lightweight, custom **WebSockets-based TypeScript library** for interacting with the WhatsApp Web API.

`bread-baileys` is a fork of [Baileys](https://github.com/WhiskeySockets/Baileys) maintained by **bread**. It talks to WhatsApp directly over a WebSocket, so it does not need Selenium, Puppeteer or any browser, and it supports the multi-device version of WhatsApp.

> [!CAUTION]
> This is version `0.0.1`, based on the Baileys 7.0 line (7.0.0-rc14). Version 7 introduced several breaking changes compared to 6.x (for example LID addressing, a deprecated `printQRInTerminal` option, and no built-in in-memory store).

---

## Table of Contents

- [What is different in bread-baileys](#what-is-different-in-bread-baileys)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Connecting an Account](#connecting-an-account)
  - [QR code](#connect-with-a-qr-code)
  - [Pairing code](#connect-with-a-pairing-code)
  - [Receive full history](#receive-full-history)
- [Saving and Restoring Sessions](#saving-and-restoring-sessions)
- [Important Socket Config Notes](#important-socket-config-notes)
- [Handling Events](#handling-events)
- [WhatsApp IDs](#whatsapp-ids)
- [Sending Messages](#sending-messages)
  - [Text, quote, mention, forward](#non-media-messages)
  - [Media messages](#media-messages)
- [Interactive Messages](#interactive-messages)
  - [Quick reply buttons (legacy)](#legacy-buttons)
  - [Native flow buttons](#native-flow-buttons)
  - [List message](#list-message)
  - [Template buttons](#template-buttons)
  - [Carousel](#carousel)
  - [AI rich response](#ai-rich-response)
  - [Raw protobuf passthrough](#raw-protobuf-passthrough)
- [Modifying Messages](#modifying-messages)
- [Working With Media](#working-with-media)
- [Chat States](#chat-states)
- [Modifying Chats](#modifying-chats)
- [User Queries](#user-queries)
- [Changing Your Profile](#changing-your-profile)
- [Groups](#groups)
- [Privacy](#privacy)
- [Broadcast Lists and Stories](#broadcast-lists-and-stories)
- [Writing Custom Functionality](#writing-custom-functionality)
- [Project Structure](#project-structure)
- [Building From Source](#building-from-source)
- [Disclaimer](#disclaimer)
- [Credits](#credits)
- [License](#license)

---

## What is different in bread-baileys

- **Interactive messages built in.** Send buttons, native flow messages, list messages, template buttons and carousels with a single `sendMessage` call. The required `<biz>` node is attached automatically.
- **AI rich responses.** Send rich blocks (text, code, tables, LaTeX, images, image grids, maps and HTML pages) through the `airich` content type.
- **`Browsers.bread()` preset** in addition to the standard browser presets.
- **Documentation uses `bread`** as the socket variable name (`const bread = makeWASocket(...)`).
- **Clean source.** Comments were removed from the codebase, and only short English doc comments remain on the custom features.

---

## Requirements

- Node.js **20 or newer**
- A WhatsApp account that you can link as a companion device

---

## Installation

Install from npm:

```bash
npm install bread-baileys
```

Install straight from your GitHub repository:

```bash
npm install github:DrBreadz/bread-baileys
```

Install from a local folder or tarball while developing:

```bash
npm install /path/to/bread-baileys
```

```bash
cd /path/to/bread-baileys
npm pack
npm install ./bread-baileys-0.0.1.tgz
```

Import it in your project:

```ts
import makeWASocket from 'bread-baileys'
```

The package is an ES module (`"type": "module"`), so use `import` syntax.

Optional peer dependencies:

| Package | Used for |
| --- | --- |
| `sharp` or `jimp` | Generating image and sticker thumbnails |
| `link-preview-js` | Generating link previews |
| `audio-decode` | Audio waveform generation |
| `ffmpeg` (system binary) | Generating video thumbnails and converting audio |

---

## Quick Start

```ts
import makeWASocket, { DisconnectReason, useMultiFileAuthState } from 'bread-baileys'
import { Boom } from '@hapi/boom'

async function start() {
	const { state, saveCreds } = await useMultiFileAuthState('auth_info')

	const bread = makeWASocket({ auth: state })

	bread.ev.on('creds.update', saveCreds)

	bread.ev.on('connection.update', update => {
		const { connection, lastDisconnect, qr } = update

		if (qr) {
			console.log('Scan this QR code with WhatsApp:', qr)
		}

		if (connection === 'close') {
			const statusCode = (lastDisconnect?.error as Boom)?.output?.statusCode
			if (statusCode !== DisconnectReason.loggedOut) {
				start()
			}
		} else if (connection === 'open') {
			console.log('Connected')
		}
	})

	bread.ev.on('messages.upsert', async ({ messages }) => {
		for (const message of messages) {
			if (message.key.fromMe || !message.message) continue
			await bread.sendMessage(message.key.remoteJid!, { text: 'Hello from bread-baileys!' })
		}
	})
}

start()
```

> [!TIP]
> The `qr` value is a raw string. Render it with a package such as `qrcode-terminal` or `qrcode` to get a scannable image.

---

## Connecting an Account

WhatsApp's multi-device API lets `bread-baileys` authenticate as a second client by scanning a **QR code** or entering a **pairing code** on your phone.

> [!TIP]
> All supported socket options are listed in the `SocketConfig` type. Your editor will autocomplete them.

### Connect with a QR code

The QR string is delivered through the `connection.update` event. The old `printQRInTerminal` option is deprecated, so handle the QR yourself:

```ts
import makeWASocket, { Browsers } from 'bread-baileys'
import qrcode from 'qrcode-terminal'

const bread = makeWASocket({
	browser: Browsers.bread('Chrome')
})

bread.ev.on('connection.update', ({ qr }) => {
	if (qr) qrcode.generate(qr, { small: true })
})
```

Available browser presets: `Browsers.ubuntu`, `Browsers.macOS`, `Browsers.windows`, `Browsers.android`, `Browsers.baileys`, `Browsers.bread` and `Browsers.appropriate`.

### Connect with a pairing code

A pairing code lets you link the device without scanning a QR code. The phone number must contain **digits only** (no `+`, `()` or `-`) and must include the country code.

```ts
import makeWASocket, { useMultiFileAuthState } from 'bread-baileys'

const { state, saveCreds } = await useMultiFileAuthState('auth_info')
const bread = makeWASocket({ auth: state })

bread.ev.on('creds.update', saveCreds)

if (!bread.authState.creds.registered) {
	const code = await bread.requestPairingCode('628123456789')
	console.log('Pairing code:', code)
}
```

You can also pass a custom pairing code, which must be exactly 8 characters:

```ts
const code = await bread.requestPairingCode('628123456789', 'BREAD123')
```

### Receive full history

1. Set `syncFullHistory` to `true`.
2. Emulate a desktop client to receive more history:

```ts
const bread = makeWASocket({
	browser: Browsers.macOS('Desktop'),
	syncFullHistory: true
})
```

History arrives through the `messaging-history.set` event.

---

## Saving and Restoring Sessions

You do not want to scan the QR code on every start. Persist the credentials and reuse them:

```ts
import makeWASocket, { useMultiFileAuthState } from 'bread-baileys'

const { state, saveCreds } = await useMultiFileAuthState('auth_info')

const bread = makeWASocket({ auth: state })

bread.ev.on('creds.update', saveCreds)
```

> [!IMPORTANT]
> `useMultiFileAuthState` stores the auth state in a folder. It is a good reference for writing your own SQL or NoSQL implementation, which is recommended for production systems.

> [!NOTE]
> Whenever a message is sent or received, Signal sessions change and the auth keys (`authState.keys`) are updated. Always persist those updates. Failing to do so prevents messages from reaching recipients. `useMultiFileAuthState` handles this for you. For custom storage, wrap your key store with `makeCacheableSignalKeyStore` to make it faster.

When serializing credentials to JSON yourself, use the `BufferJSON` helper (`BufferJSON.replacer` and `BufferJSON.reviver`).

---

## Important Socket Config Notes

### Cache group metadata (recommended)

If your bot works in groups, cache the metadata so it is not fetched on every send:

```ts
import NodeCache from '@cacheable/node-cache'

const groupCache = new NodeCache({ stdTTL: 5 * 60, useClones: false })

const bread = makeWASocket({
	cachedGroupMetadata: async jid => groupCache.get(jid)
})

bread.ev.on('groups.update', async ([event]) => {
	const metadata = await bread.groupMetadata(event.id!)
	groupCache.set(event.id!, metadata)
})

bread.ev.on('group-participants.update', async event => {
	const metadata = await bread.groupMetadata(event.id)
	groupCache.set(event.id, metadata)
})
```

### Improve retries and decrypt poll votes

Provide a `getMessage` function backed by your own store. It is used to resend messages that failed to deliver and to decrypt poll votes:

```ts
const bread = makeWASocket({
	getMessage: async key => await getMessageFromYourStore(key)
})
```

### Receive notifications in the WhatsApp app

By default the socket marks itself online, which suppresses phone notifications. Disable that:

```ts
const bread = makeWASocket({
	markOnlineOnConnect: false
})
```

### Use the latest WhatsApp Web version

```ts
import makeWASocket, { fetchLatestBaileysVersion } from 'bread-baileys'

const { version } = await fetchLatestBaileysVersion()
const bread = makeWASocket({ version })
```

---

## Handling Events

The socket uses an EventEmitter-style API. All events are fully typed (see the `BaileysEventMap` type).

```ts
bread.ev.on('messages.upsert', ({ messages }) => {
	console.log('received messages', messages)
})
```

Common events:

| Event | Description |
| --- | --- |
| `connection.update` | Connection state changes, QR codes and disconnect reasons |
| `creds.update` | Credentials changed, save them |
| `messaging-history.set` | History sync received |
| `messages.upsert` | New messages |
| `messages.update` | Message status updates and poll votes |
| `messages.delete` | Messages deleted |
| `messages.reaction` | Reactions added or removed |
| `message-receipt.update` | Delivery and read receipts |
| `presence.update` | Typing and online presence |
| `chats.upsert`, `chats.update`, `chats.delete` | Chat changes |
| `contacts.upsert`, `contacts.update` | Contact changes |
| `groups.upsert`, `groups.update` | Group changes |
| `group-participants.update` | Members added, removed, promoted or demoted |
| `group.join-request` | Join requests |
| `blocklist.set`, `blocklist.update` | Blocklist changes |
| `call` | Incoming calls |

Always loop over the array when handling `messages.upsert`:

```ts
bread.ev.on('messages.upsert', async ({ messages }) => {
	for (const message of messages) {
		console.log(JSON.stringify(message, undefined, 2))
	}
})
```

### Decrypt poll votes

Poll votes are encrypted and arrive in `messages.update`:

```ts
import { getAggregateVotesInPollMessage } from 'bread-baileys'

bread.ev.on('messages.update', async updates => {
	for (const { key, update } of updates) {
		if (update.pollUpdates) {
			const pollCreation = await getMessage(key)
			if (pollCreation) {
				console.log(
					'poll aggregation:',
					getAggregateVotesInPollMessage({
						message: pollCreation,
						pollUpdates: update.pollUpdates
					})
				)
			}
		}
	}
})
```

`getMessage` is a function you implement on top of your own store.

### Data store

`bread-baileys` does not ship a store for chats, contacts or messages. Build one that fits your needs by listening to the events above (for example `messages.upsert`, `chats.upsert` and `contacts.upsert`) and saving the data to your database. Storing a full chat history in memory wastes RAM.

---

## WhatsApp IDs

An ID (also called a `jid`) identifies the person, group or channel you interact with.

| Type | Format | Example |
| --- | --- | --- |
| Person (phone number) | `[country code][number]@s.whatsapp.net` | `628123456789@s.whatsapp.net` |
| Person (LID) | `[id]@lid` | `1234567890@lid` |
| Group | `[id]@g.us` | `123456789-123345@g.us` |
| Broadcast list | `[timestamp]@broadcast` | `1700000000@broadcast` |
| Status (stories) | `status@broadcast` | `status@broadcast` |
| Channel (newsletter) | `[id]@newsletter` | `120363000000000000@newsletter` |

Useful helpers: `jidNormalizedUser`, `jidEncode`, `jidDecode`, `isJidGroup`, `isJidNewsletter`, `isPnUser`, `isLidUser`.

Other handy utilities:

- `getContentType` returns the content type of a message.
- `getDevice` returns the device a message was sent from.
- `makeCacheableSignalKeyStore` makes the auth key store faster.
- `downloadContentFromMessage` and `downloadMediaMessage` download media.
- `generateMessageIDV2` generates a message ID.

---

## Sending Messages

Every message is sent with `bread.sendMessage(jid, content, options)`. It returns the full generated message, including its `key`.

### Non-media messages

#### Text

```ts
await bread.sendMessage(jid, { text: 'Hello world' })
```

#### Quote a message (works with every type)

```ts
await bread.sendMessage(jid, { text: 'Replying to you' }, { quoted: message })
```

#### Mention users (works with most types)

```ts
await bread.sendMessage(jid, {
	text: '@628123456789 hello',
	mentions: ['628123456789@s.whatsapp.net']
})
```

Mention everyone in a group:

```ts
await bread.sendMessage(groupJid, { text: 'Hello everyone', mentionAll: true })
```

#### Forward a message

```ts
const message = await getMessageFromYourStore(key)
await bread.sendMessage(jid, { forward: message })
```

#### Location

```ts
await bread.sendMessage(jid, {
	location: {
		degreesLatitude: 24.121231,
		degreesLongitude: 55.1121221
	}
})
```

#### Contact

```ts
const vcard =
	'BEGIN:VCARD\n' +
	'VERSION:3.0\n' +
	'FN:Jeff Singh\n' +
	'ORG:Ashoka Uni;\n' +
	'TEL;type=CELL;type=VOICE;waid=911234567890:+91 12345 67890\n' +
	'END:VCARD'

await bread.sendMessage(jid, {
	contacts: {
		displayName: 'Jeff',
		contacts: [{ vcard }]
	}
})
```

#### Reaction

```ts
await bread.sendMessage(jid, {
	react: {
		text: '💖',
		key: message.key
	}
})
```

Use an empty string as `text` to remove the reaction.

#### Pin a message

| Duration | Seconds |
| --- | --- |
| 24 hours | 86400 |
| 7 days | 604800 |
| 30 days | 2592000 |

```ts
await bread.sendMessage(jid, {
	pin: {
		type: 1,
		time: 86400,
		key: message.key
	}
})
```

Use `type: 0` to unpin.

#### Poll

```ts
await bread.sendMessage(jid, {
	poll: {
		name: 'My Poll',
		values: ['Option 1', 'Option 2', 'Option 3'],
		selectableCount: 1,
		toAnnouncementGroup: false
	}
})
```

#### Link preview

Install `link-preview-js` and include a link in the text. The preview is generated automatically:

```ts
await bread.sendMessage(jid, { text: 'Check out https://github.com' })
```

Pass `linkPreview: null` to disable the preview for a message.

### Media messages

For every media type you can pass a `Buffer`, a `{ url }` or a `{ stream }`.

> [!TIP]
> Prefer `{ url }` or `{ stream }`. They are encrypted as a stream and never load the whole file into memory.

#### Image

```ts
await bread.sendMessage(jid, {
	image: { url: './media/photo.png' },
	caption: 'Hello world'
})
```

#### Video

```ts
await bread.sendMessage(jid, {
	video: { url: './media/video.mp4' },
	caption: 'Hello world',
	ptv: false
})
```

Set `ptv: true` to send the video as a round video note.

#### GIF

WhatsApp does not support `.gif` files, so GIFs are sent as MP4 videos with `gifPlayback`:

```ts
import { readFileSync } from 'fs'

await bread.sendMessage(jid, {
	video: readFileSync('./media/animation.mp4'),
	caption: 'Hello world',
	gifPlayback: true
})
```

#### Audio

For audio to play on every device, convert it to Opus in an OGG container with `ffmpeg`:

```bash
ffmpeg -i input.mp4 -avoid_negative_ts make_zero -ac 1 -c:a libopus output.ogg
```

```ts
await bread.sendMessage(jid, {
	audio: { url: './media/audio.ogg' },
	mimetype: 'audio/ogg; codecs=opus',
	ptt: true
})
```

Set `ptt: true` to send it as a voice note.

#### Sticker

```ts
await bread.sendMessage(jid, {
	sticker: { url: './media/sticker.webp' }
})
```

#### View once

Add `viewOnce: true` to an image, video or audio message:

```ts
await bread.sendMessage(jid, {
	image: { url: './media/photo.png' },
	viewOnce: true,
	caption: 'This disappears after one view'
})
```

#### Disappearing messages

```ts
import { WA_DEFAULT_EPHEMERAL } from 'bread-baileys'

await bread.sendMessage(jid, { disappearingMessagesInChat: WA_DEFAULT_EPHEMERAL })

await bread.sendMessage(jid, { text: 'hello' }, { ephemeralExpiration: WA_DEFAULT_EPHEMERAL })

await bread.sendMessage(jid, { disappearingMessagesInChat: false })
```

| Duration | Seconds |
| --- | --- |
| Off | 0 |
| 24 hours | 86400 |
| 7 days | 604800 |
| 90 days | 7776000 |

---

## Interactive Messages

`bread-baileys` generates interactive messages directly from `sendMessage`. The content shape you pass decides which message is built, and the extra `<biz>` node that WhatsApp needs to render it is added for you.

All interactive types accept these optional fields:

| Field | Description |
| --- | --- |
| `mentions` | Array of JIDs to mention |
| `mentionAll` | Mention everyone in the group |
| `contextInfo` | Extra `contextInfo` merged into the message |

> [!NOTE]
> Interactive messages depend on how WhatsApp renders them on each client, and that behavior is controlled by WhatsApp. Rendering can differ between clients, account types and app versions, and may change over time. Test on the devices you care about.

### Legacy buttons

Simple reply buttons. Each button is `{ id, text }`:

```ts
await bread.sendMessage(jid, {
	text: 'Choose an option',
	footer: 'bread-baileys',
	buttons: [
		{ id: 'yes', text: 'Yes' },
		{ id: 'no', text: 'No' }
	]
})
```

Add an `image` or `video` to show media above the buttons, or a `title` for a text header:

```ts
await bread.sendMessage(jid, {
	image: { url: './media/photo.png' },
	caption: 'Pick one',
	footer: 'bread-baileys',
	buttons: [{ id: 'a', text: 'Option A' }]
})
```

### Native flow buttons

Native flow buttons support quick replies, links, copy buttons, call buttons and select menus. `buttonParamsJson` can be an object and is stringified automatically.

```ts
await bread.sendMessage(jid, {
	title: 'Welcome',
	text: 'Pick an action below',
	footer: 'bread-baileys',
	interactiveButtons: [
		{
			name: 'quick_reply',
			buttonParamsJson: { display_text: 'Say hi', id: 'hi' }
		},
		{
			name: 'cta_url',
			buttonParamsJson: {
				display_text: 'Open website',
				url: 'https://example.com',
				merchant_url: 'https://example.com'
			}
		},
		{
			name: 'cta_copy',
			buttonParamsJson: { display_text: 'Copy code', copy_code: 'BREAD2026' }
		},
		{
			name: 'cta_call',
			buttonParamsJson: { display_text: 'Call us', phone_number: '+628123456789' }
		}
	]
})
```

Select menu:

```ts
await bread.sendMessage(jid, {
	text: 'Choose a flavor',
	interactiveButtons: [
		{
			name: 'single_select',
			buttonParamsJson: {
				title: 'Flavors',
				sections: [
					{
						title: 'Popular',
						rows: [
							{ header: '', title: 'Vanilla', description: 'Classic', id: 'vanilla' },
							{ header: '', title: 'Chocolate', description: 'Rich', id: 'chocolate' }
						]
					}
				]
			}
		}
	]
})
```

Add `image` or `video` to attach media in the header, and `subtitle` for a header subtitle. Use `messageParamsJson` to pass extra message-level params.

### List message

```ts
await bread.sendMessage(jid, {
	title: 'Menu',
	text: 'Select an item from the list',
	footer: 'bread-baileys',
	buttonText: 'Open menu',
	sections: [
		{
			title: 'Drinks',
			rows: [
				{ title: 'Coffee', rowId: 'coffee', description: 'Hot or iced' },
				{ title: 'Tea', rowId: 'tea', description: 'Green or black' }
			]
		},
		{
			title: 'Food',
			rows: [{ title: 'Bread', rowId: 'bread', description: 'Freshly baked' }]
		}
	]
})
```

### Template buttons

```ts
await bread.sendMessage(jid, {
	title: 'Hello',
	text: 'Template message with buttons',
	footer: 'bread-baileys',
	templateButtons: [
		{ index: 1, quickReplyButton: { displayText: 'Reply', id: 'reply' } },
		{ index: 2, urlButton: { displayText: 'Website', url: 'https://example.com' } },
		{ index: 3, callButton: { displayText: 'Call', phoneNumber: '+628123456789' } }
	]
})
```

### Carousel

Each card can have a title, subtitle, body, footer, image and native flow buttons:

```ts
await bread.sendMessage(jid, {
	carousel: {
		text: 'Our products',
		footer: 'bread-baileys',
		cards: [
			{
				title: 'Sourdough',
				subtitle: 'Fresh daily',
				body: 'Slow fermented for 24 hours',
				footer: 'From $5',
				image: { url: './media/sourdough.png' },
				buttons: [
					{
						name: 'cta_url',
						buttonParamsJson: { display_text: 'Order', url: 'https://example.com/sourdough' }
					}
				]
			},
			{
				title: 'Baguette',
				body: 'Crispy crust, soft inside',
				image: { url: './media/baguette.png' },
				buttons: [
					{
						name: 'quick_reply',
						buttonParamsJson: { display_text: 'I want this', id: 'baguette' }
					}
				]
			}
		]
	}
})
```

You can set `carousel.cardType` to change the card layout. The default is horizontal scrolling cards.

### AI rich response

The `airich` content type sends a rich response made of ordered blocks. It shows up with the forwarded label and AI bot information by default.

```ts
await bread.sendMessage(jid, {
	title: 'Result',
	footer: 'Generated by bread-baileys',
	airich: [
		{ text: '## Hello\nThis is **markdown** text.' },
		{ code: 'console.log("hello")', language: 'javascript' },
		{
			table: {
				title: 'Prices',
				header: true,
				rows: [
					['Item', 'Price'],
					['Bread', 5],
					['Butter', 3]
				]
			}
		},
		{ latex: { text: 'E = mc^2' } },
		{ image: { url: 'https://example.com/picture.png' } },
		{ grid: ['https://example.com/1.png', 'https://example.com/2.png'] }
	]
})
```

Supported blocks:

| Block | Shape | Notes |
| --- | --- | --- |
| Text | `{ text }` | Markdown text |
| Code | `{ code, language? }` | Syntax highlighted code block |
| Table | `{ table: { title?, rows, header? } }` | The first row is a header unless `header: false` |
| LaTeX | `{ latex: { text?, expressions? } }` | Math expressions |
| Image | `{ image }` | A URL string, a `Buffer` or `{ url }` |
| Grid | `{ grid: [...] }` | Several images shown as a grid |
| Map | `{ map }` | Raw map metadata |
| HTML | `{ html, fallback?, trustedSources? }` | HTML page rendered by WhatsApp. `fallback` is the text for older clients |
| Raw | `{ raw }` | A raw `AIRichResponseSubMessage` |

Extra options:

| Field | Description |
| --- | --- |
| `title` | Info line shown above the message |
| `footer` | Footer text appended to the response |
| `forwarded` | Set to `false` to hide the forwarded label and bot info. Default is `true` |
| `botJid` | Bot JID used in the forwarded bot info. Default is `0@bot` |

String image URLs are used as they are. Buffers are uploaded without encryption so that the resulting URL can be rendered.

### Raw protobuf passthrough

If you already have a fully built protobuf message, pass it directly:

```ts
await bread.sendMessage(jid, { interactiveMessage: myInteractiveMessage })
await bread.sendMessage(jid, { buttonsMessage: myButtonsMessage })
await bread.sendMessage(jid, { listMessage: myListMessage })
await bread.sendMessage(jid, { templateMessage: myTemplateMessage })
await bread.sendMessage(jid, { richResponseMessage: myRichResponseMessage })
```

### Reading button responses

Replies to interactive messages arrive in `messages.upsert`. Check the message type to find the selected option:

```ts
bread.ev.on('messages.upsert', ({ messages }) => {
	for (const message of messages) {
		const content = message.message
		if (!content) continue

		const buttonId = content.buttonsResponseMessage?.selectedButtonId
		const listRowId = content.listResponseMessage?.singleSelectReply?.selectedRowId
		const nativeFlow = content.interactiveResponseMessage?.nativeFlowResponseMessage?.paramsJson

		console.log({ buttonId, listRowId, nativeFlow })
	}
})
```

---

## Modifying Messages

### Delete a message for everyone

```ts
const sent = await bread.sendMessage(jid, { text: 'Oops' })
await bread.sendMessage(jid, { delete: sent.key })
```

Deleting a message only for yourself is done with `chatModify`, see [Modifying Chats](#modifying-chats).

### Edit a message

```ts
await bread.sendMessage(jid, {
	text: 'Updated text',
	edit: sent.key
})
```

---

## Working With Media

### Thumbnails

Thumbnails for images and stickers are generated automatically when `sharp` or `jimp` is installed. Video thumbnails need `ffmpeg` on your system.

### Download received media

```ts
import { createWriteStream } from 'fs'
import { downloadMediaMessage, getContentType } from 'bread-baileys'

bread.ev.on('messages.upsert', async ({ messages }) => {
	for (const m of messages) {
		if (!m.message) continue

		const type = getContentType(m.message)

		if (type === 'imageMessage') {
			const stream = await downloadMediaMessage(
				m,
				'stream',
				{},
				{
					logger,
					reuploadRequest: bread.updateMediaMessage
				}
			)

			stream.pipe(createWriteStream('./download.jpeg'))
		}
	}
})
```

Use `'buffer'` instead of `'stream'` to get the file as a `Buffer`.

### Re-upload expired media

WhatsApp removes old media from its servers. Another device that still has the file can re-upload it:

```ts
await bread.updateMediaMessage(message)
```

---

## Chat States

### Reject a call

Get `callId` and `callFrom` from the `call` event:

```ts
bread.ev.on('call', async ([call]) => {
	await bread.rejectCall(call.id, call.from)
})
```

### Read messages

Message keys must be marked as read explicitly:

```ts
await bread.readMessages([message.key])
```

### Update presence

```ts
await bread.sendPresenceUpdate('composing', jid)
await bread.sendPresenceUpdate('available', jid)
```

Presence can be `available`, `unavailable`, `composing`, `recording` or `paused`. If a desktop client is active, WhatsApp does not push notifications to your phone. Mark the bot offline with `bread.sendPresenceUpdate('unavailable')` to get them back.

---

## Modifying Chats

> [!IMPORTANT]
> If one of these app-state updates is malformed, WhatsApp can log you out of all your devices.

### Archive a chat

```ts
const lastMessage = await getLastMessageInChat(jid)
await bread.chatModify({ archive: true, lastMessages: [lastMessage] }, jid)
```

### Mute or unmute a chat

```ts
await bread.chatModify({ mute: 8 * 60 * 60 * 1000 }, jid)
await bread.chatModify({ mute: null }, jid)
```

### Mark a chat read or unread

```ts
await bread.chatModify({ markRead: false, lastMessages: [lastMessage] }, jid)
```

### Delete a message for yourself

```ts
await bread.chatModify(
	{
		clear: {
			messages: [{ id: 'MESSAGE_ID', fromMe: true, timestamp: '1654823909' }]
		}
	},
	jid
)
```

### Delete a chat

```ts
await bread.chatModify(
	{
		delete: true,
		lastMessages: [{ key: lastMessage.key, messageTimestamp: lastMessage.messageTimestamp }]
	},
	jid
)
```

### Pin or unpin a chat

```ts
await bread.chatModify({ pin: true }, jid)
```

### Star or unstar a message

```ts
await bread.chatModify(
	{
		star: {
			messages: [{ id: 'MESSAGE_ID', fromMe: true }],
			star: true
		}
	},
	jid
)
```

`getLastMessageInChat` is a function you implement on top of your own store.

---

## User Queries

### Check if a number is on WhatsApp

```ts
const [result] = (await bread.onWhatsApp('628123456789')) ?? []
if (result?.exists) {
	console.log('Registered as', result.jid)
}
```

### Fetch chat history

You need the oldest message you currently have in the chat. Results arrive in the `messaging-history.set` event:

```ts
const oldest = await getOldestMessageInChat(jid)
await bread.fetchMessageHistory(50, oldest.key, oldest.messageTimestamp)
```

The maximum is 50 messages per request.

### Fetch a status

```ts
const status = await bread.fetchStatus(jid)
console.log(status)
```

### Fetch a profile picture (groups too)

```ts
const lowRes = await bread.profilePictureUrl(jid)
const highRes = await bread.profilePictureUrl(jid, 'image')
```

### Fetch a business profile

```ts
const profile = await bread.getBusinessProfile(jid)
console.log(profile?.description, profile?.category)
```

### Fetch someone's presence

```ts
bread.ev.on('presence.update', console.log)
await bread.presenceSubscribe(jid)
```

---

## Changing Your Profile

```ts
await bread.updateProfileStatus('Hello World!')
await bread.updateProfileName('bread')
await bread.updateProfilePicture(jid, { url: './new-picture.jpeg' })
await bread.removeProfilePicture(jid)
```

`updateProfilePicture` and `removeProfilePicture` also work for groups when you are an admin.

---

## Groups

You must be an admin to change most group properties.

```ts
const group = await bread.groupCreate('My Group', ['1234@s.whatsapp.net', '5678@s.whatsapp.net'])
await bread.sendMessage(group.id, { text: 'Hello everyone' })
```

```ts
await bread.groupParticipantsUpdate(jid, ['1234@s.whatsapp.net'], 'add')
```

The last argument can be `add`, `remove`, `promote` or `demote`.

```ts
await bread.groupUpdateSubject(jid, 'New subject')
await bread.groupUpdateDescription(jid, 'New description')
```

```ts
await bread.groupSettingUpdate(jid, 'announcement')
await bread.groupSettingUpdate(jid, 'not_announcement')
await bread.groupSettingUpdate(jid, 'locked')
await bread.groupSettingUpdate(jid, 'unlocked')
```

```ts
await bread.groupLeave(jid)
```

Invite links:

```ts
const code = await bread.groupInviteCode(jid)
console.log('https://chat.whatsapp.com/' + code)

const newCode = await bread.groupRevokeInvite(jid)

const joinedJid = await bread.groupAcceptInvite(code)

const info = await bread.groupGetInviteInfo(code)

const joined = await bread.groupAcceptInviteV4(jid, groupInviteMessage)
```

Pass only the code (not the full URL) to `groupAcceptInvite` and `groupGetInviteInfo`.

Metadata:

```ts
const metadata = await bread.groupMetadata(jid)
console.log(metadata.id, metadata.subject, metadata.desc)

const allGroups = await bread.groupFetchAllParticipating()
```

Join requests:

```ts
const requests = await bread.groupRequestParticipantsList(jid)

await bread.groupRequestParticipantsUpdate(jid, ['1234@s.whatsapp.net'], 'approve')
```

The last argument can be `approve` or `reject`.

Disappearing messages and add mode:

```ts
await bread.groupToggleEphemeral(jid, 86400)

await bread.groupMemberAddMode(jid, 'all_member_add')
```

`groupMemberAddMode` accepts `all_member_add` or `admin_add`.

---

## Privacy

```ts
await bread.updateBlockStatus(jid, 'block')
await bread.updateBlockStatus(jid, 'unblock')

const settings = await bread.fetchPrivacySettings(true)
const blocklist = await bread.fetchBlocklist()

await bread.updateLastSeenPrivacy('contacts')
await bread.updateOnlinePrivacy('match_last_seen')
await bread.updateProfilePicturePrivacy('contacts')
await bread.updateStatusPrivacy('contacts')
await bread.updateReadReceiptsPrivacy('none')
await bread.updateGroupsAddPrivacy('contacts')
await bread.updateDefaultDisappearingMode(86400)
```

Accepted values:

| Method | Values |
| --- | --- |
| `updateLastSeenPrivacy` | `all`, `contacts`, `contact_blacklist`, `none` |
| `updateOnlinePrivacy` | `all`, `match_last_seen` |
| `updateProfilePicturePrivacy` | `all`, `contacts`, `contact_blacklist`, `none` |
| `updateStatusPrivacy` | `all`, `contacts`, `contact_blacklist`, `none` |
| `updateReadReceiptsPrivacy` | `all`, `none` |
| `updateGroupsAddPrivacy` | `all`, `contacts`, `contact_blacklist` |

---

## Broadcast Lists and Stories

Messages can be sent to broadcast lists and to stories by adding a few message options:

```ts
await bread.sendMessage(
	'status@broadcast',
	{
		image: { url: './media/photo.png' },
		caption: 'My story'
	},
	{
		backgroundColor: '#1a1a1a',
		font: 1,
		statusJidList: ['1234@s.whatsapp.net', '5678@s.whatsapp.net'],
		broadcast: true
	}
)
```

- The content can be text, an image, a video or a voice note.
- `statusJidList` is the list of people who will receive the status.
- `broadcast: true` enables broadcast mode.
- Broadcast list IDs look like `12345678@broadcast`. You send to them the same way you send to groups and individual chats.

---

## Writing Custom Functionality

`bread-baileys` is built to be extended. Instead of rewriting internals, listen to the raw WhatsApp traffic and add your own handlers.

### Enable debug logs

```ts
import P from 'pino'

const bread = makeWASocket({
	logger: P({ level: 'debug' })
})
```

At debug level you can see every frame WhatsApp sends. A frame has three parts:

- `tag`: what the frame is about (for example `message`)
- `attrs`: string key-value metadata (usually including the ID)
- `content`: the actual data

### Listen for raw WebSocket events

```ts
bread.ws.on('CB:edge_routing', (node: BinaryNode) => {})

bread.ws.on('CB:edge_routing,id:abcd', (node: BinaryNode) => {})

bread.ws.on('CB:edge_routing,id:abcd,routing_info', (node: BinaryNode) => {})
```

### Send custom nodes

`relayMessage` and `query` are exposed on the socket for advanced use:

```ts
await bread.relayMessage(jid, message, { messageId, additionalNodes: [] })
```

---

## Project Structure

```
src/
  Defaults/    default socket config and constants
  Signal/      Signal protocol and LID mapping
  Socket/      socket layers (socket, chats, messages, groups, business, communities, newsletter)
  Types/       TypeScript types
  Utils/       helpers (messages, media, auth state, crypto, event buffer, ...)
  WABinary/    WhatsApp binary protocol encoder and decoder
  WAM/         WhatsApp analytics encoder
  WAUSync/     USync query protocols
WAProto/       compiled protobuf definitions
```

The socket is built in layers. Each layer wraps the one below it:

`socket` → `chats` → `groups` → `newsletter` → `messages-send` → `messages-recv` → `business` → `communities`

`makeWASocket` returns the top layer, so everything is available on a single object.

The interactive message logic lives in:

- `src/Utils/messages.ts` (`generateInteractiveContent`)
- `src/Socket/messages-send.ts` (`getInteractiveNodes`)
- `src/Types/Message.ts` (`InteractiveMessageContent`, `AIRichBlock`)

---

## Building From Source

```bash
git clone https://github.com/DrBreadz/bread-baileys.git
cd bread-baileys
npm install
npm run build
```

The compiled output is written to `lib/`.

| Script | Description |
| --- | --- |
| `npm run build` | Compile TypeScript and fix ESM import extensions |
| `npm pack` | Build and create a `.tgz` package |

---

## Disclaimer

This project is not affiliated, associated, authorized, endorsed by, or in any way officially connected with WhatsApp or any of its subsidiaries or affiliates. The official WhatsApp website is whatsapp.com. "WhatsApp" and related names, marks, emblems and images are registered trademarks of their respective owners.

The maintainers do not condone using this library in ways that violate the WhatsApp Terms of Service. You are responsible for using it fairly and as intended. Use it at your own discretion and do not use it for spam, stalkerware, or bulk and automated messaging.

---

## Credits

- **bread**: author and maintainer of bread-baileys.
- [Baileys](https://github.com/WhiskeySockets/Baileys) by Rajeh Taher and the WhiskeySockets contributors: the original library this project is based on.
- [@pokearaujo](https://github.com/pokearaujo/multidevice), [@Sigalor](https://github.com/sigalor/whatsapp-web-reveng) and [@Rhymen](https://github.com/Rhymen/go-whatsapp/) for their research on the WhatsApp protocol.

---

## License

Released under the [MIT License](LICENSE). The original copyright notice is kept as required by the license.
