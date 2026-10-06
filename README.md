# About this dataset

Real sessions between developers and AI coding agents, from six public sources, in one dataset.

**Data:** Duke CS cluster, `/usr/xtmp/zf93/errata-data/2026-10-04` · [Get the data](#get-the-data) ·
[Columns](#column-reference)

> **Status.** Version `2026-10-04` is on Duke Computer Science's cluster, in temporary storage. Use it for
> research only, and do not try to identify the people in these sessions.

## Overview

Version `2026-10-04` holds 59,030 sessions from 28 coding agents, started between November 2023 and October 2026.
They make 19.4 million rows, plus 92 thousand from helper agents the sessions started. Secrets are masked.

Each row says who produced it and whether a person typed it. Most labels come from the agents' own records or
from fixed rules. One, `pushback`, comes from a model.

## Get the data

The data is on Duke Computer Science's cluster:

```
/usr/xtmp/zf93/errata-data/
├── 2026-10-04/                       77.5 GB, 22,319 files
│   ├── changelog.json                counts, the copied datasets' sources and licences, how pushback was labelled
│   ├── data/<source>/*.parquet       one row per session, one folder per source
│   ├── conversations/<source>/*.parquet
│   ├── conversations_subagents/<source>/*.parquet
│   └── copied/                       the copied datasets' own files, as released
└── errata-data-2026-10-04.sha256     a checksum for every file
```

Anyone with a Duke CS account can read it. Questions and removal requests go to its maintainer (NetID
`zf93`).

The folder is in temporary storage (xtmp). Files not read or changed for 180 days are deleted, and nothing
there is backed up.

**On a Duke CS machine**, read the files where they are. They are Parquet, so DuckDB, pandas or polars read
them directly:

```python
import duckdb

duckdb.sql("""
    SELECT agent, count(*) AS typed_messages, round(avg(pushback::int), 3) AS pushback_share
    FROM '/usr/xtmp/zf93/errata-data/2026-10-04/conversations/*/*.parquet'
    WHERE typed_by_developer AND copy_of IS NULL
    GROUP BY agent ORDER BY typed_messages DESC
""").show()
```

**To copy all of it**, use Globus, Duke CS's recommended way to move large data. In the Globus file manager
(app.globus.org), open the collection "Duke Compsci XTMP" at the path `/zf93/errata-data/`, and transfer
`2026-10-04` and the checksum file to your own collection.

**To copy a part**, use rsync through the login server, with your NetID in place of `<netid>`. For example, the
conversations tables (6.0 GB):

```bash
rsync -a --info=progress2 <netid>@login.cs.duke.edu:/usr/xtmp/zf93/errata-data/2026-10-04/conversations ./2026-10-04/
```

**To check a full copy**, run this in the folder that holds `2026-10-04` and the checksum file:

```bash
sha256sum -c --quiet errata-data-2026-10-04.sha256
```

## What's in the dataset

| Table | One row is | Rows |
|---|---|---|
| `entire`, `specstory`, `aider`, `swe-chat-v2`, `dataclaw`, `openhands-feedback` | a session, with its raw files | 80,471 |
| `conversations` | one piece of a message: text, reasoning, a tool call, a tool's output, a notice | 19,439,229 |
| `conversations_subagents` | the same, for helper agents a session started | 91,967 |

The six session tables are the six sources:

| Source | Sessions kept | What it is |
|---|---|---|
| Entire | 24,375 | the agent's own log, saved by the Entire CLI in public GitHub repositories |
| SpecStory | 15,037 | chat histories the SpecStory editor extension saves as Markdown, committed to GitHub |
| DataClaw | 9,002 | session exports developers uploaded to Hugging Face, copied as released |
| Aider | 5,801 | Aider's chat-history files committed to GitHub; 92% are one repository's benchmark runs |
| SWE-chat v2 | 4,553 | SALT-NLP's public dataset; its other 13,415 sessions are the same as Entire's, kept once |
| OpenHands feedback | 262 | OpenHands' public dataset of user sessions with feedback |

A session found in several places is kept once. Its other copies stay in the session tables with `duplicate_of`
set, and they have no rows.

## Columns you will use

| Column | Meaning |
|---|---|
| `session_key`, `turn_number` | The session, and the row's position in it. |
| `agent` | The coding agent. Empty where the source does not name it (many SpecStory and DataClaw sessions). |
| `kind` | What the row is: `text`, `thinking`, `tool_call`, `tool_result`, `notice`, `command`, `queued`, ... |
| `actor` | Who made it: `developer`, `agent`, `tool`, `harness` (the agent program's own text) or `peer`. |
| `typed_by_developer` | `true` if a person typed the message. Empty where the format cannot tell. |
| `copy_of` | Set on the delivered copy of a message queued while the agent was busy. Skip it to count each message once. |
| `content` | The text, with secrets replaced by `[REDACTED]`. |
| `tool_name`, `command`, `file_path` | What the agent ran, and on what. |
| `category`, `bash_category` | The kind of tool call (`Action`, `Research`, ...) and of shell command (`git`, `test/build`, ...). |
| `is_error` | The tool reported a failure. |
| `model`, `input_tokens`, `output_tokens` | The model, and its token counts for each call. |
| `pushback` | `true` if a typed message pushes back on the agent's work. Written by a model. |

The session tables add `agent`, `models`, `repo`, `repo_license`, `started_at`, `ended_at`, counts such as
`n_user_messages` and `n_tool_calls`, and `files`, the raw files themselves. The
[column reference](#column-reference) at the end defines every column of every table.

## Common queries

| To get | Use |
|---|---|
| What developers asked | `typed_by_developer` and `copy_of` is null |
| Moments of pushback | `pushback = true`; the rows just before show what the agent had done |
| What the agent did | `kind = 'tool_call'`, then `tool_name`, `category`, `command` |
| Failed tool calls | `is_error = true` |
| One agent's sessions | `agent = 'Claude Code'` (or `'Codex'`, `'Cursor'`, ...) |
| One row per distinct session | a session table where `duplicate_of` is null |
| The raw logs | `files` in the session tables |

## Labels: rules and a model

Claude Code, Codex, Gemini CLI, Copilot CLI, Cursor, OpenCode and Pi record who typed a message, and we read
those records. SpecStory, Aider and DataClaw do not. There a user message counts as typed unless its text is
plainly a tool's, such as a notice or a template.

`pushback` is the one label a model wrote. gpt-6-luna read each typed message on its own, without the rest of
the session, and answered yes or no. Of 453,843 typed rows, it marked 135,000 as pushback. 47 have no label,
because Azure's content filter refused them. No human checked these labels. The instructions the model got are in
the version's `changelog.json` (`pushback_labels`).

SWE-chat's own prompt labels, which its authors made with a model, are carried over to its sessions' rows.

## Limitations

- "Typed" is a rule's guess where a format records nothing, so a script driving an agent can read as a person.
- `pushback` is one model's reading of one message without its context; use it to find candidates, not as truth.
- Sources are lopsided: 92% of Aider's sessions are one benchmark script, and one repository is 12% of Entire.
- Masking catches most secrets, not every one; the `redactions` column counts what was masked in each session.
- Only public logs on default branches are found; private and branch-only sessions are missing.

## Column reference

Every column of every table, with its type, a description and notes:

- [Session tables](docs/columns/sessions.md) (`data/<source>/`): 36 columns, one row per session.
- [`conversations`](docs/columns/conversations.md): 50 columns, one row per piece of a message.
- [`conversations_subagents`](docs/columns/conversations_subagents.md): 50 columns, the rows of helper agents a session started.

## Licence

The data keeps each source's licence. `repo_license` gives each session's repository licence; the copied datasets
keep theirs: SWE-chat v2 is ODC-By, and the DataClaw datasets and OpenHands feedback are MIT. `changelog.json`
credits each one (`copied_datasets`): its source, revision, author and licence.
