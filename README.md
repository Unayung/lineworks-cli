# lineworks

Command-line control of LINE WORKS messaging — send one message to a whole list
of people and group rooms, and read channels back, without touching the web UI.

Zero dependencies. Python 3.8+. Single file.

> **Unofficial.** Not affiliated with, endorsed by, or supported by LINE WORKS /
> NAVER WORKS / Works Mobile. This is an independent client built by reading the
> web messenger's own network traffic. The API it talks to is undocumented and
> private, so it can break without warning.
>
> Use it on **your own account only**, at human request rates, and check LINE
> WORKS' terms of service and your own organisation's IT policy before you do —
> automating a private API is commonly against them. `send --yes` delivers real
> messages to real colleagues. You are responsible for how you use this.

## Why this exists

The official [LINE WORKS Bot API](https://developers.worksmobile.com/en/docs/bot-api)
cannot do this job:

- Bots are tenant-scoped. Per the docs, *"Bots cannot be used in talk rooms that
  include LINE users or external LINE WORKS users."* External LINE users also
  have no `userId` in the Directory, so there is no 1:1 bot target for them.
- Every send endpoint requires a `botId`. There is no user-as-sender API.

If your recipients are all internal colleagues, **use the Bot API instead** — it
is supported, stable, and sanctioned. This client exists for the external-room
case the Bot API is barred from.

## Install

```bash
chmod +x lineworks
ln -s "$PWD/lineworks" /usr/local/bin/lineworks   # optional
```

## Auth

Log in to `talk.worksmobile.com` in a browser, then copy the whole `Cookie`
request header from any XHR in DevTools (the session cookie is `HttpOnly`, so
`document.cookie` will not show it):

```bash
mkdir -p ~/.config/lineworks
umask 077
pbpaste > ~/.config/lineworks/cookie
chmod 600 ~/.config/lineworks/cookie

lineworks doctor        # confirms the session and prints your identity
```

`LINEWORKS_COOKIE` overrides the file. The cookie is never printed. Exit code
`3` means the session expired — log in again and refresh the file.

## Quick start

```bash
lineworks channels                              # your targets
lineworks targets --to-file team.txt            # resolve, send nothing
lineworks send --to-file team.txt -m "..." --dry-run
lineworks send --to-file team.txt -m "..." --tag notice-1 --yes

lineworks read --to "#Project Kickoff" --count 50
lineworks read --to "#Project Kickoff" --follow
```

A target list mixes people and rooms:

```
@Alice Chen          person
#Project Kickoff     group room
c:123456789          raw channelNo
123456789            bare digits are always a channelNo
// comment
```

## Safety

Broadcasts reach real people, so the defaults are deliberately obstructive:

- `--dry-run` resolves the list and prints the plan without sending.
- **Any unresolved target refuses the entire send**, even with `--yes`. Pass
  `--skip-unresolved` to deliberately send only to what resolved.
- Ambiguous names are never guessed — two matches means a refusal, not a coin flip.
- `--yes` is required, and the refusal names the people/group split and the
  sending account.
- `--tag` makes a run resumable via `~/.config/lineworks/sent.jsonl`; a rerun
  skips what already went out, so a crash mid-broadcast cannot double-message.
- `--delay` (default 3s) paces sends. An auth failure mid-run stops the loop.

## Docs

`SKILL.md` carries the full reverse-engineered API map, the `channelType` table,
and every payload gotcha that cost a failed request.

## License

MIT
