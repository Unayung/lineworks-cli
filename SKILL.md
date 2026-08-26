---
name: lineworks
description: Send a message as your own LINE WORKS user account to many targets at once - a mixed list of people and group talk rooms, including rooms containing external LINE users that bots cannot reach. Resolve a target list, preview the plan, then broadcast with rate limiting and resume-safe idempotency. Use whenever the user wants to send the same LINE WORKS message to multiple recipients.
---

# lineworks

Zero-dependency Python CLI driving the private web-messenger API behind
`talk.worksmobile.com`, authenticated with a browser session cookie.

**Always pass `--json`** when parsing output. Works before or after the
subcommand:

```json
{"ok": true, "data": ...}
{"ok": false, "error": {"message": "...", "code": 2, "detail": {...}}}
```

Exit codes: `0` ok - `1` error - `2` usage - `3` auth - `4` network.
On exit `3` the session expired: log in again in the browser and refresh the
cookie file. **Do not retry.**

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

## Everything is a channel

There is no separate person-vs-group endpoint. A 1:1 chat and a group room are
both a `channelNo`, sent through the same call. The kind only affects display.

`channelType` (undocumented; inferred from live data):

| type | meaning |
|---|---|
| 1 | 1:1 direct chat |
| 8 | memo-to-self (excluded from broadcasts) |
| 10 | **external / cross-tenant room** - the ones bots cannot reach |
| 2, 3, 4, 5 | internal group rooms of various flavours |

**Two hard limits on who you can reach:**

1. **A channel must already exist.** Someone never messaged has no `channelNo`
   and resolves `notfound`. Open the chat once in the web UI, then re-run.
2. **`channels` only sees the last ~30 days.** The underlying call is a delta
   sync, so a dormant room is invisible even though it exists. Post in it, or
   pin its `channelNo` via `--directory`.

```bash
lineworks channels                 # active channels = the target list
lineworks channels --days 14       # narrower window
lineworks channels --raw --json    # unparsed response
```

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
- `syncUserChannelList`'s response shape is **unverified** (the capture omitted
  response bodies). The parser walks the tree for any dict with `channelNo`
  rather than assuming a path, and falls back to raw. If `channels` looks thin,
  check `channels --raw --json` and widen it.
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
