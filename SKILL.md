---
name: lineworks
description: Drive LINE WORKS messaging as your own user account - send text, images, files or stickers to many people and group rooms at once (including rooms with external LINE users, which bots cannot reach), read channel history, list channels with their real member rosters, and browse the domain directory with starred contacts. Use whenever the user wants to send, broadcast, or read LINE WORKS messages, or inspect their LINE WORKS channels, members or contacts.
---

# lineworks

Zero-dependency Python CLI driving the private web-messenger API behind
`talk.worksmobile.com`, authenticated with a browser session cookie.

**Binary:** `lineworks` if on PATH, else `~/lineworks-cli/lineworks`
(executable, stdlib only, no venv needed).

**Always pass `--json`** when parsing output. Works before or after the
subcommand:

```json
{"ok": true, "data": ...}
{"ok": false, "error": {"message": "...", "code": 2, "detail": {...}}}
```

Exit codes: `0` ok - `1` error - `2` usage - `3` auth - `4` network.
On exit `3` the session expired: log in again in the browser and refresh the
cookie file. **Do not retry.**

| command | |
|---|---|
| `doctor` / `whoami` | session + identity check |
| `channels` | every channel: type, size, whether it holds LINE users |
| `channels --members` | real member roster per channel, classified |
| `targets` | resolve a target list **without sending** - run this first |
| `send` | text, `--file`, or `--sticker`, to many targets at once |
| `read` | channel history; `--follow` polls |
| `contacts` | domain directory; `--starred` for starred contacts |
| `stickers` | browse sticker packages and their ids |
| `raw` | any endpoint with auth attached; `--enc` picks the body encoding |

## Why not the official API

The Bot API cannot do this job, for two documented reasons:

- Bots are tenant-scoped: *"LINE ユーザーまたは外部 LINE WORKS ユーザーを含む
  トークルームでは Bot は利用できません"* - unusable in any room containing a
  consumer LINE user or an external LINE WORKS user. External LINE users have
  no `userId` in the Directory either, so there is no 1:1 bot target.
- Every send endpoint requires a `botId`. There is no user-as-sender API.

For internal-only recipients, prefer the official Bot API - it is supported and
stable. This CLI exists for the external case it cannot serve.

## Auth

Cookie from a normal browser login in `~/.config/lineworks/cookie` (mode 0600)
or `LINEWORKS_COOKIE`. The CLI refuses a world-readable cookie file.

```bash
lineworks doctor      # cookie + identity check
lineworks whoami      # userNo / domainId / email
```

**Identity needs no configuration.** The server returns `x-user-no`,
`x-user-domain` and `x-user-email` response headers on every call; `whoami`
reads them. Never print the cookie, never ask the user to paste it into chat.

## Profiles (multiple identities)

One machine, several real people: each identity is a directory holding its own
`cookie`, `device` and `sent.jsonl`:

```
~/.config/lineworks/profiles/<name>/cookie   # mode 0600, one per person
```

Select with `--profile <name>` or `LINEWORKS_PROFILE=<name>` (flag wins).
`doctor` prints `ok [<name>] - email...`; send plans and `--json` envelopes
carry a `profile` field, so logs show who acted.

**Once `profiles/` exists there is NO default identity.** Any command without
a profile exits `3` with "pick an identity" - deliberate, so an agent can
never fall back to the wrong person. The pre-profiles single-user layout
(`~/.config/lineworks/cookie`) keeps working as long as `profiles/` does not
exist. `LINEWORKS_COOKIE` still overrides everything - an explicit credential
is an explicit identity choice.

Rules when driving this as an agent:

- Set `LINEWORKS_PROFILE` once for the whole session/process; do not pass
  different `--profile` values within one task.
- Before the first `send` under a profile, run `doctor` and confirm the
  reported email is the person you are supposed to be acting as.
- Profile names are one path segment (`/` and leading `.` rejected, exit `2`).
- Migration is just `mkdir -p` + move the three files; nothing else changes.

## Everything is a channel

There is no separate person-vs-group endpoint. A 1:1 chat and a group room are
both a `channelNo`, sent through the same call. The kind only affects display.

`channelType` (undocumented; inferred from live data):

| type | meaning |
|---|---|
| 1 | 1:1 direct chat |
| 8 | memo-to-self (excluded from broadcasts) |
| 2, 3, 4, 5, 6, 10 | group rooms of various flavours |

**Do not use `channelType` to detect external rooms.** Most LINE-connected rooms
are type 10, but not all - a type 6 room was observed carrying a LINE user. Use
`channelExtras` instead:

| signal | meaning |
|---|---|
| `channelExtras.serviceType == "line"` | LINE-connected room (`LINE` column) |
| `channelExtras.localTenants` non-empty | other LINE WORKS tenants (INFERRED - always `[]` in observed data) |
| neither, plus `domainId`/`orgId` | internal room |

At member level, `writerMemberInfos[].domainId` classifies each person:

| domainId | who |
|---|---|
| `0` | **consumer LINE user** (no `domainName`) |
| == your `domainId` | your own colleague |
| other nonzero | external LINE WORKS user (another tenant) |

```bash
lineworks channels --members          # real roster per channel, classified
```

**`--members` is a real roster.** `getChannelMembers` returns actual membership
including silent members, so external LINE users are classified properly:

```
POST /p/oneapp/client/chat/getChannelMembers
{"channelNo": N, "pagingCount": 500, "memberUpdateTime": <cursor>}
-> {"members": [...], "nextMemberUpdateTime": ...}
```

- Pages 500 at a time; a full page means re-request with
  `nextMemberUpdateTime` as `memberUpdateTime`.
- **The roster includes people who have LEFT** (`join: false`). Counting them
  inflates the room - filtering to `join: true` matches the channel list's
  `userCount` exactly. The CLI reports current members and notes departures
  separately (`20 members: internal 3, line 17; 3 left`).
- One request per channel, so `--members` is opt-in.

Note that in a LINE-connected room **your own userNo is a per-service alias**,
not the `userNo` that `whoami` reports.

**One hard limit:** a channel must already exist. Someone never messaged has no
`channelNo` and resolves `notfound`. Open the chat once in the web UI, then
re-run.

```bash
lineworks channels                 # COMPLETE list, dormant rooms included
lineworks channels --members       # real roster per channel (1 request each)
lineworks channels --sync          # delta-sync fallback (~30d window)
lineworks channels --raw --json
```

`channels` uses `getUserChannelListByType`, which returns **every** channel,
grouped `normal` / `official` / `teamroom`. It needs `pagingCount` in the body -
an empty body is rejected with `code 400 Key:pagingCount`.

`--sync` is the older `syncUserChannelList` path, kept as a fallback. It is a
**delta sync**: `updateTime` means "changed since", `0` is rejected (`code 400`)
and much past ~30-45 days is rejected (`code 2012`), so the CLI walks the window
back until one is accepted. **It silently omits dormant rooms** - on this tenant
it returned 14 channels where the full list returns 24, hiding three
LINE-connected external rooms. Do not use it to build a broadcast list.

## Target lists

```
@Alice Chen          person
#Project Kickoff     group room
person:Bob Lee       person, long form
group:Sales Team     group room, long form
Carol Wang           unprefixed -> auto, must resolve unambiguously
c:123456789          raw channelNo, no lookup
123456789            bare digits are ALWAYS a channelNo, whatever the prefix
# note               comment ('#' + space)
// note              comment, unambiguous
```

`#` is overloaded - it marks a group AND starts a comment. The rule is
whitespace: `# note` is a comment, `#Sales` is the room named Sales. Prefer
`//` and `group:` when generating lists programmatically.

Names are matched against **open channel titles**: exact, then case-insensitive,
then substring. A query matching two channels resolves `ambiguous` and **will
not pick one**. Always run `targets` before `send`:

```bash
lineworks targets --to-file team.txt --json
```

A raw channelNo is looked up in the channel list so the plan shows **which room
it is** - never send to a bare number you have not seen named.

Statuses: `resolved` - `ambiguous` - `notfound` (no open channel) - `unmapped`.
A refused send prints the full breakdown, not just a count.
`--directory file.json` (`{"people":{name:channelNo},"groups":{...}}`) overrides
channel lookup entirely - useful to pin exact ids and skip a fuzzy match.

## Sending

```bash
lineworks send --to-file team.txt -m "Standup moved to 10am" --dry-run
lineworks send --to-file team.txt -m "Standup moved to 10am" \
                --tag standup-0826 --yes
```

- `--dry-run` prints the resolved plan and message, sends nothing.
- **Any unresolved target refuses the whole send** (exit 2), even with `--yes`.
  `--skip-unresolved` deliberately sends only to what resolved.
- `--yes` is required, and the refusal names the people/group split and the
  sending account.
- `--delay` seconds between sends, default 3. Keep it human.
- `--tag` makes a run resumable: sends append to `~/.config/lineworks/sent.jsonl`
  and a rerun with the same tag skips what already went. **Always pass `--tag`
  for lists over ~20**, or a crash mid-run double-messages real colleagues.
- An auth failure mid-run **stops the loop** rather than hammering a dead
  session; network failures skip that target and continue.

Never pass `--yes` unless the user asked to send that specific message to that
specific list in that message.

### Images and files

```bash
lineworks send --to "@Jimmy Hsiao" --file ./photo.png --yes
```

**Two steps, and the upload itself commits the message** - there is no third
`sendMessage` call:

1. `POST /p/oneapp/client/chat/issueResourcePath` (talk host, raw JSON)
   `{serviceId:"works", channelNo, filename, filesize, msgType}`
   -> `{code:200, resourcePath:"/k/oneapp/r/...", fileUuid}`
2. `POST https://storage.worksmobile.com<resourcePath>?Servicekey=oneapp&writeMode=overwrite&isMakethumbnail=true`
   multipart body with field `file`; the message metadata rides in **headers**:
   `X-channelNo`, `X-type`, `X-extras` (JSON), `X-ocn: 1`, `Web-Device-ID`,
   and `X-serviceId: works`.

Gotchas, each one a failed upload:

- **`X-serviceId: works` is mandatory.** Without it every upload fails with
  `400 Bad request Key:serviceId EEXIST` - a message that names the wrong
  problem entirely. It is a header; passing serviceId in the query does nothing.
- **The storage service uses `code: 0` for success** - the exact opposite of the
  chat API, where 0 is not success and 200 is. Two services, two conventions.
- `X-extras` needs `{filesize, filename, resourcepath}`, plus `{width, height}`
  for an image. The CLI reads dimensions from PNG/JPEG/GIF headers directly.
- The upload host comes from `window.envData.storageAddress` on the logged-in
  talk page (`https://storage.worksmobile.com` here). It is per-tenant config,
  not a constant - re-read it if uploads start 404ing.

`messageTypeCode` (from the bundle's own enum): 1 text, 4 location, 5 bot,
8/10 rich, **11 image**, 12 audio, 14 video, 15 sticker, **16 file**, 17 team
note, 18 sticker v3, 22 merge-forward, 26 profile, 27 bot rich, 29 template.
The CLI picks 11 when it can read image dimensions, else 16.

Files use the same path as images - `send --file` picks type 11 when it can
read image dimensions (PNG/JPEG/GIF headers) and type 16 otherwise. Both are
verified working.

### Stickers

**A sticker is NOT an upload.** It is an ordinary `sendMessage` with
`type: 18` and the sticker identifiers in `extras`:

```json
{"pkgId":12034,"pkgVer":1,"stkId":"66122838","stkType":"","stkOpt":""}
```

```bash
lineworks stickers                       # 28 packages
lineworks stickers --pkg 12034           # that package's sticker ids
lineworks send --to "@Someone" --sticker 12034:66122838 --yes
```

- **`stkType` MUST be empty for a sticker.** The client picks the image URL with
  `stkType ? "emojis" : "stickers"`, so any truthy value renders a broken image.
  `"works"` is what EMOJI use - putting it on a sticker looks right in the
  payload and fails on screen. **The server accepts every value with
  `code: 200`**, so this is invisible server-side; verified instead by fetching
  both URLs (`/stickers/...` 200, `/emojis/...` 404).
- `stkOpt` carries flags; `"A"` means animated (use the animation URL).
- **Package contents are a static file, not an API**:
  `/p/static/static/wm/stickers/<v6>/<v3>/<v1>/<pkgId>/PC/productInfo.meta`,
  where the three path segments derive from the **version**, not the id:
  `ver//10**6 / ver//10**3 / ver%10**3`. It lists every `stickers[].id`.
- The package catalogue is
  `GET /p/alice/admin/authapi/v1.0/sticker-categories/v7?suggestScheme=2`
  (`stickerPackages` + `emojiPackages`, each with `id` and `version`).

## Reading

`getChannelInfo` doubles as the history read - same endpoint as channel
metadata, form-flat encoding.

```bash
lineworks read --to "#CS team" --count 60      # recent text, newest last
lineworks read --to "#CS team" --since 60      # only after messageNo 60
lineworks read --to "#CS team" --all           # include system events
lineworks read --to "#CS team" --follow        # poll (>=5s floor)
lineworks read --to "#CS team" --json
```

- `messageNo` is an **AFTER cursor**; `0` means the most recent tail.
  `direction` appears inert - 0, 1 and 2 all returned the same window.
- Text is filtered to `messageTypeCode` 1 (typed) and 5 (bot/rich) by default.
  Types 101/102 are join/leave events with empty `content` and their payload in
  `extras`; 4, 6, 18, 27, 96 are cards and rich content. `--all` shows them.
- Writer names come from `recentMessage.writerMemberInfos`. **`botList` comes
  back empty here**, so a bot's messages show a bare `userNo` rather than a
  name - a known gap, not a parse failure.
- `readInfos[]` (userNo, join, readMsgs) and `largestReadMsgNo` are in the same
  response - read receipts are available but not yet surfaced as a command.

## Contacts (a second service)

The contacts directory lives on **`contact.worksmobile.com`**, not the talk
host - guessing contact paths under `talk.worksmobile.com` returns 404 forever.
Same session cookie, but it wants an `x-xsrf-token` header; the web client
generates a fresh UUID per call and the server accepts any well-formed one, so
there is nothing to harvest from the cookie.

```bash
lineworks contacts            # whole domain; * marks a starred contact
lineworks contacts --starred  # starred only
lineworks contacts --json
```

| endpoint | notes |
|---|---|
| `POST /v2/api/search` | `{"type":"USER","page":1,"maxResults":N,"filter":{},"domainId":"..."}` |
| `POST /v2/api/domain/users/{userNo}/important` | **sets** starred (empty body, 204) |
| `GET /v2/api/domains/{d}/orgunits/{o}/forHeader` | org unit header info |

- **"Starred" is `important`** in the API. The UI function is
  `updateImportantDomainUser`. Each search result carries `important: bool`.
- **Omit `orgUnitId` to get the whole domain**; pass one (with
  `includedSubOrgUnit`) to scope to an org unit.
- **`filter: {"important": true}` is IGNORED** - the server returns every user
  regardless, so `--starred` filters client-side. Do not trust that filter.
- A contact's `id` matches the channel's `userList[].mappingContactNos`, which
  is the join between a chat and a directory entry.
- Only the *set* call was observed. Un-starring is presumably `DELETE` on the
  same path, but that is **unverified** - this CLI does not write here at all.

## API map (reverse-engineered)

Base `https://talk.worksmobile.com`. Required headers are added for you:
`web-device-id` (`<userNo>-<uuid>`, persisted in `~/.config/lineworks/device`),
`x-ocn: 11`, `x-request-id`, `device-language`, `x-translate-lang`, Origin,
Referer.

| endpoint | encoding |
|---|---|
| `POST /p/oneapp/client/chat/sendMessage` | form `payload=<json>` |
| `POST /p/oneapp/client/chat/getReadInfos` | form `payload=<json>` |
| `POST /p/oneapp/client/chat/getChannelInfo` | form, flat params |
| `POST /p/oneapp/client/chat/syncUserChannelList` | raw JSON |
| `POST /p/oneapp/client/chat/getMessageUnreadCountByType` | raw JSON |
| `GET  /p/contact/v3/domain/contacts/my?<epoch_ms>` | - |

Responses carry their own `code` **in the body**, independent of HTTP status:
`200` success, `400` bad argument, `2012` updateTime too old. An HTTP 200 with
body `code: 400` is a failure.

Send payload:

```json
{"serviceId":"works","channelNo":123456789,"tempMessageId":"987654321",
 "caller":{"domainId":100000001,"userNo":110000000000001},
 "extras":"","content":"hi","msgTid":"987654321","type":1}
```

## Payload gotchas

- **Three body encodings on one API.** form-`payload=<json>`, form-flat, and
  raw JSON - see the table. Using the wrong one is how you get a 200 that does
  nothing. `raw --enc json|payload|form` exposes the choice.
- `tempMessageId` and `msgTid` are **the same value**, a fresh 9-digit number
  per message. The protocol appears to dedupe on it, so reusing one risks a
  silently dropped message. `send_one` regenerates per send.
- `caller` must carry both `domainId` and `userNo` - `whoami` supplies them.
- Content is `ensure_ascii=False` then urlencoded; CJK round-trips intact.
- Responses may carry a UTF-8 BOM; decode with `utf-8-sig`.
- **Body `code` is 200 on success, not 0.** Treating 0 as the success value
  marks every delivered message as failed.
- **`syncUserChannelList.updateTime` is a delta cursor, not a list flag.**
  `0` is rejected outright (`code 400 Invalid updateTime`), and anything past
  roughly 30-45 days is rejected too (`code 2012 too old`). `fetch_channels`
  walks the window back (30/21/14/7/1d) until one is accepted. The practical
  effect: **channels with no recent activity never appear.**
- Channels arrive in `result[]`; each has `channelNo`, `channelType`, `title`,
  `userCount`, `botCount`, `joined`. The parser walks for `channelNo` rather
  than trusting that path.
- `getChannelInfo` with `recentMessageCount` reads messages back - use it to
  confirm a send actually landed rather than trusting the 200.
- DevTools strips `Cookie` from HAR exports - a capture will not contain the
  credential, and cannot be used to authenticate on its own.

## Reading the bundle (how the above was found)

`https://talk.worksmobile.com/dist/main-<hash>.js` is served **unauthenticated**
(~3 MB). It contains the URL builders, the message-type enum and the upload
flow. The hash changes on deploy; get the current one from the page source. This
is far cheaper than probing endpoints blind - the contacts service cost 14 wasted
404s before a capture found it.

Endpoints visible in the bundle but **not yet implemented** here:

| endpoint | why it matters |
|---|---|
| `/client/chat/getVisibleUserChannelList` | another channel list; returned 0 rows for every body tried |
| `/client/chat/getMessageListByNoList` | fetch specific messages by number |
| `message/sendMulti` | referenced as a 1-by-1 multi-send link |

## Escape hatch

```bash
lineworks raw POST /p/oneapp/client/chat/getChannelInfo \
          --enc form --data '{"channelNo":123456789,"direction":0}'
lineworks --debug ...     # req/resp to stderr, cookie redacted
```

## Rules

- Only ever send from the user's own account.
- Automated messaging to external LINE users can trip spam heuristics on the
  LINE side, and this is an undocumented private API on a corporate tenant.
  Both are the user's call to make knowingly.
- Response shapes can change without notice. If a field is missing the CLI
  returns what it got rather than crashing; re-check with `raw` or `--debug`.
