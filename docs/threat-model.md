# Threat model

## What this project does and where untrusted input enters

Convolvger is a command-line tool that turns a public AI-chat share link
into a Markdown document and a JSON archive on the user's own machine.
It runs as the user, with the user's permissions. There is no server, no
account and no stored credential.

Everything a provider serves is untrusted. It enters in four ways:

- **Fetched shares** (ChatGPT, Gemini, Grok, DeepSeek, Kimi, Qwen): the
  CLI requests the share over HTTPS with no cookie or token, following
  redirects. Bodies are read through `providers/_http.py`, which refuses
  more than 32 MiB counted after decompression.
- **Captured shares** (Claude, Perplexity): a bookmarklet
  (`capture/bookmarklet.py`) runs on the share page, reads the page's own
  API with the user's session, and posts the result to a listener
  (`capture/server.py`) on `127.0.0.1:23477`. The listener accepts one
  snapshot within 180 seconds and refuses the following:
  - bodies over 32 MiB;
  - JSON that is not an object or nests too deeply;
  - an `Origin` that differs from the origin of the URL the payload
    claims;
  - a URL that no provider handles.
- **Saved snapshots** (`export --from-file`): read up to 32 MiB.
- **Archives being checked** (`verify`): read with no size limit, by
  design, because an archive can be larger than its source.

All provider JSON is parsed through `core/jsonload.py`, which refuses
nesting deeper than 64 levels in sources and 512 in archives. ChatGPT's
payload is also refused if it unfolds past 32 MiB.

Untrusted text then flows into:

- file names derived from the conversation title (`cli/_output.py`
  `file_stem`, written with exclusive create in the current directory);
- the Markdown renderer (`renderers/markdown.py`): code fences, citation
  links and the Sources list;
- the JSON archive (`core/archive.py`).

## Components that matter most / least

Most:

- the capture listener and the bookmarklet;
- the provider parsers (`providers/*/_parse.py`, plus ChatGPT's
  `_turbostream.py`);
- `core/jsonload.py` and `providers/_http.py`;
- output file naming and writing (`cli/_output.py`);
- Markdown escaping of fences, link text and link addresses.

Less: `validation/`, `core/findings.py`, and the CLI's messages.

Out of scope: `tests/`, `.github/`, and the providers' own servers.

## How to exercise it

- `tests/fixtures/<provider>/` holds synthetic snapshots of every
  provider's format. Feed one in with
  `convolvger export <share-url> --from-file <file> -o -`.
- `convolvger inspect <share-url> --from-file <file>` parses without
  writing.
- `convolvger verify <archive.json>` checks an archive.
  `tests/fixtures/archives/` has archives in schemas 2 and 3.
- The listener is started with `convolvger export --capture`, and a
  payload can be posted to it with any HTTP client.

No network access is needed for any of these.

## How you rate severity

- **Critical:** code execution, or writing a file outside the path the
  user chose or the current directory, triggered by share content or a
  web page.
- **High:** a web page other than the share page getting a snapshot
  accepted, or reading anything back from the listener; share content
  escaping its Markdown structure in a way that changes links or code
  outside it.
- **Medium:** a crash, hang, or memory use well beyond the documented
  limits from input within them; a refusal that can be bypassed.
- **Low:** anything that needs the user to pass unusual flags, or that
  only affects the CLI's own messages.

## Anything to leave alone

These are known and accepted:

- **Programs on the same machine.** Another local program can post a
  forged snapshot to the listener. It could equally write the archive
  file itself, so this grants it nothing.
- **Slow senders.** A client that sends one byte at a time can hold the
  single-threaded listener past its deadline. Each read gets the time
  that was left when the connection opened. The listener only runs while
  the user waits for a capture.
- **`verify` has no size limit.** This is deliberate (see above).
- **HTML in messages.** Message text is written into the Markdown as the
  provider served it. Raw HTML or a `javascript:` link inside a message
  survives, and rendering it safely is the viewer's job. Links that
  Convolvger builds itself (citations, Sources) are escaped.
- **Sharer identity.** Names, usernames and account ids are kept in the
  JSON archive's `provider_metadata` and are never rendered. Archiving
  what the share serves is the purpose of the tool.
- **Pages of a share beyond the first.** Only the first page of a long
  Perplexity thread is read, and a finding reports the rest.
