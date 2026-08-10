# claudish-to-english

A Claude Code plugin that shows a **plain-English rewrite** of each assistant
message, produced by a **local LLM (ollama)**. It is **display-only**: Claude's
own reasoning and the saved transcript keep the original text — only what you
read on screen changes.

An optional second hook rewrites **Markdown files** into plain English when they
are written or edited (opt-in, off by default).

> Status: working prototype. Every hook fails **open** — if anything goes wrong
> (ollama down, timeout, missing dependency), you simply see Claude's original
> text. The plugin can never swallow or corrupt an answer.

---

## Requirements (read this first)

This plugin shells out to a **local** model. Nothing works until these are in place:

| Requirement | Why | Install |
|---|---|---|
| **ollama**, running | Does the rewriting, locally | `brew install ollama` then `ollama serve` |
| A pulled model | The actual rewriter | `ollama pull llama3.2:3b` (~2 GB) |
| `jq` | Parses hook JSON | ships with macOS; else `brew install jq` |
| `curl` | Talks to ollama | ships with macOS |

Warm the model once after `ollama serve` (the first call is a slow cold load):

```bash
ollama run llama3.2:3b "hi"
```

**Without ollama running, the plugin does nothing** — Claude's output shows
normally, unchanged. That is by design, not a bug. The first time it can't reach
ollama in a session it appends a one-line notice so you know why (once per
session; set `CLAUDISH_NOTICE=0` to silence it).

---

## Install

From the community marketplace:

```shell
/plugin marketplace add anthropics/claude-plugins-community
/plugin install claudish-to-english@claude-community
```

Or directly from this repository (also serves its own marketplace):

```shell
/plugin marketplace add gvzdv/claudish-to-english
/plugin install claudish-to-english@gvzdv-plugins
```

If the install summary says `Run /reload-plugins to activate.`, run that command.

**Try before installing** (loads it for one session, no install):

```bash
claude --plugin-dir /path/to/claudish-to-english
```

Run `/reload-plugins` after edits; if it doesn't load, check the `/plugin`
**Errors** tab.

---

## How the display hook works

Claude Code fires the `MessageDisplay` event **once per streamed chunk**, not
once per message. Each fire is a separate process carrying `message_id`,
`index`, a `final` flag, and this chunk's `delta` (a text fragment, not the
whole message). So the hook **buffers every delta** to a temp file (keyed by
`message_id`) and only calls the model on the **final** chunk, once the whole
message is known:

```
chunk 0 (final:false) ─┐
chunk 1 (final:false) ─┤ append each delta to $TMPDIR/claudish-to-english/<session>/<message>/<index>.part
chunk 2 (final:false) ─┘  → emit nothing (append) or "" (replace)
chunk 3 (final:true)  ──► reconstruct full message → call ollama once → show the rewrite
                          → delete the buffer
```

On that final chunk it also reads the **original user question** from the
transcript and passes it to the model as **context only** — to keep the rewrite
on-topic. The model is told never to answer or repeat the question; it only
rewrites the assistant's message.

### Display modes

| `CLAUDISH_MODE` | On screen | Notes |
|---|---|---|
| `append` (default) | Original streams normally, then a `💬 In plain English:` block is appended. | Safest. No streaming loss; if the LLM fails you just don't get the extra block. |
| `replace` | Only the simplified version (original chunks suppressed while streaming). | Experimental. Appears all at once after LLM latency; on failure it re-shows the full original. |

---

## Markdown file rewrite (optional second hook)

A `PostToolUse` hook (`rewrite-md.sh`) rewrites Markdown **files** into plain
English when they are written or edited. Unlike the display hook, this changes
bytes on disk.

**Opt-in by directory.** It does nothing unless `CLAUDISH_MD_DIR` is set, and it
only touches `*.md` files whose resolved path is inside that directory. Every
other `README`, `CLAUDE.md`, or doc you edit is left alone.

| `CLAUDISH_MD_MODE` | Result | Notes |
|---|---|---|
| `sibling` (default) | Writes `NAME.plain.md` next to `NAME.md`. | Non-destructive; the original is never touched. |
| `overwrite` | Replaces `NAME.md` in place. | Adds a `<!-- claudish-to-english:rewritten -->` marker so a re-write is skipped (idempotent). A weak model can degrade real docs — use with care. |

In both modes: YAML frontmatter is split off and re-attached **verbatim**, fenced
code is left to the model instruction, short files are skipped, and the write is
atomic. Fail-open here means the file is left **exactly as the agent wrote it**.

```jsonc
// enable for one directory, sibling mode (safe default), via env in the hook command
CLAUDISH_MD_DIR=/ABS/PATH/docs/plain
```

---

## Configuration (env vars)

| Var | Default | Meaning |
|---|---|---|
| `CLAUDISH_ENABLED` | `1` | Master switch. `0` = pass everything through. |
| `CLAUDISH_MODE` | `append` | `append` or `replace` (display hook). |
| `CLAUDISH_MODEL` | `llama3.2:3b` | ollama model name. |
| `CLAUDISH_OLLAMA` | `http://localhost:11434` | ollama base URL. |
| `CLAUDISH_MIN_CHARS` | `200` | Skip messages/files whose prose (code stripped) is shorter than this. |
| `CLAUDISH_STUB` | `0` | `1` = deterministic stub instead of the model (for testing display mechanics). |
| `CLAUDISH_TIMEOUT` | `45` | LLM client timeout (seconds). Keep it below the hook `timeout`. |
| `CLAUDISH_DEBUG` | `0` | `1` = write a debug log to `$TMPDIR/claudish-to-english/`. |
| `CLAUDISH_NOTICE` | `1` | `1` = append a one-time, once-per-session notice when ollama is unreachable. `0` = stay fully silent (pure fail-open). |
| `CLAUDISH_MD_DIR` | *(unset)* | **Markdown hook opt-in.** Only `*.md` under this directory is rewritten. Unset = the Markdown hook does nothing. |
| `CLAUDISH_MD_MODE` | `sibling` | `sibling` (`NAME.plain.md`) or `overwrite` (in place). |
| `CLAUDISH_MD_SUFFIX` | `plain` | Sibling infix: `NAME.<suffix>.md`. |

The hook `timeout` (default 10s for `MessageDisplay`) is raised to 60s in
`hooks/hooks.json` to survive a cold model load; `CLAUDISH_TIMEOUT` keeps the LLM
call itself bounded.

**Quick kill switch:** set `CLAUDISH_ENABLED=0`, or disable the plugin.

### Reasoning models

The request sends `"think": false`. Models with a hidden reasoning phase
otherwise spend most of their time generating reasoning tokens you never see —
much slower for identical output quality on this simple task. Keep it off.

---

## Privacy / egress

The rewriter runs **entirely locally** against ollama, so **no conversation
content leaves your machine**. If you ever point `CLAUDISH_OLLAMA` at a
remote/hosted endpoint, that context (which can include file contents from tool
results) would be sent off-box — don't do that unless you understand and accept
it.

---

## Layout

```
claudish-to-english/
├── .claude-plugin/
│   ├── plugin.json         # plugin manifest
│   └── marketplace.json    # so the repo can be added as a marketplace directly
├── hooks/
│   └── hooks.json          # MessageDisplay -> rewrite.sh ; PostToolUse -> rewrite-md.sh
├── rewrite.sh              # display-rewrite hook
├── rewrite-md.sh           # markdown-file rewrite hook (opt-in)
├── LICENSE
└── README.md
```

## License

MIT — see [LICENSE](./LICENSE).
