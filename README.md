# About this dataset

Real sessions between developers and AI coding agents, from six public sources, in one dataset.

**Data:** Duke CS cluster, `/usr/xtmp/zf93/errata-data/2026-10-11` (the newest version) ·
[Versions](#versions) · [Get the data](#get-the-data) · [Columns](#column-reference)

> **Status.** A new version is added each week, beside the ones before, on Duke Computer Science's cluster in
> temporary storage. The newest is `2026-10-11`. Use it for research only, and do not try to identify the people in
> these sessions.

## Versions

Use the newest folder. Each folder is a complete dataset on its own: a file a week did not change is the same file
as the week before's, so it takes no extra space. Older folders stay as they were published, for anyone
reproducing a result.

| Version | Published | Sessions kept | New sessions | Updated sessions | Typed messages | Pushback | Size |
|---|---|---|---|---|---|---|---|
| `2026-10-11` | 2026-10-12 | 60,214 | 3,460 | 1,201 | 460,302 | 137,766 | 79.8 GB |
| `2026-10-04` | 2026-10-06 | 57,786 | the first version | – | 435,878 | 131,590 | 77.0 GB |

<details>
<summary><b>2026-10-11</b>: what changed</summary>

- 3,460 sessions are new: Entire 2,557, SpecStory 177 and Aider 134 are kept; the other 592 repeat sessions already
  in the dataset.
- 1,201 sessions were updated: read again because their source changed (new checkpoints, repository details).
  None lost a message.
- 506 SWE-chat v2 sessions are now kept from their Entire copy, which holds the same messages; SWE-chat's labels
  moved with them.
- The three copied datasets had nothing new.

</details>

## Overview

Version `2026-10-11` holds 60,214 sessions from 32 coding agents, started between November 2023 and October 2026.
They make 19.4 million rows, plus 150 thousand from helper agents the sessions started. Secrets are masked.

Each row says who produced it and whether a person typed it. Most labels come from the agents' own records or
from fixed rules. One, `pushback`, comes from a model we ran; SWE-chat v2's own labels are copied as they are. We
checked the labels on a random sample of rows; the [results](#how-accurate-the-labels-are) are below.

## Get the data

The data is on Duke Computer Science's cluster:

```
/usr/xtmp/zf93/errata-data/
├── 2026-10-11/                       the newest version: 79.8 GB, 22,446 files
│   ├── changelog.json                counts, the copied datasets' sources and licences, how pushback was labelled
│   ├── data/<source>/*.parquet       one row per session, one folder per source
│   ├── conversations/<source>/*.parquet
│   ├── conversations_subagents/<source>/*.parquet
│   └── copied/                       the copied datasets' own files, as released
├── errata-data-2026-10-11.sha256     a checksum for every file of 2026-10-11
├── 2026-10-04/                       the version before, laid out the same way
└── errata-data-2026-10-04.sha256
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
    FROM '/usr/xtmp/zf93/errata-data/2026-10-11/conversations/*/*.parquet'
    WHERE typed_by_developer AND copy_of IS NULL
    GROUP BY agent ORDER BY typed_messages DESC
""").show()
```

**To copy all of it**, use Globus, Duke CS's recommended way to move large data. In the Globus file manager
(app.globus.org), open the collection "Duke Compsci XTMP" at the path `/zf93/errata-data/`, and transfer
`2026-10-11` and its checksum file to your own collection.

**To copy a part**, use rsync through the login server, with your NetID in place of `<netid>`. For example, the
conversations tables (6.2 GB):

```bash
rsync -a --info=progress2 <netid>@login.cs.duke.edu:/usr/xtmp/zf93/errata-data/2026-10-11/conversations ./2026-10-11/
```

**To check a full copy**, run this in the folder that holds `2026-10-11` and its checksum file:

```bash
sha256sum -c --quiet errata-data-2026-10-11.sha256
```

## What's in the dataset

| Table | One row is | Rows |
|---|---|---|
| `entire`, `specstory`, `aider`, `swe-chat-v2`, `dataclaw`, `openhands-feedback` | a session, with its raw files (the copied datasets' are in `copied/`) | 83,931 |
| `conversations` | one piece of a message: text, reasoning, a tool call, a tool's output, a notice | 19,379,785 |
| `conversations_subagents` | the same, for helper agents a session started | 150,027 |

The six session tables are the six sources (version `2026-10-11`):

| Source | Sessions kept | What it is |
|---|---|---|
| Entire | 26,699 | the agent's own log, saved by the Entire CLI in public GitHub repositories |
| SpecStory | 14,956 | chat histories the SpecStory editor extension saves as Markdown, committed to GitHub |
| DataClaw | 8,369 | session exports developers uploaded to Hugging Face, copied as released |
| Aider | 5,935 | Aider's chat-history files committed to GitHub; 90% are one repository's benchmark runs |
| SWE-chat v2 | 4,026 | SALT-NLP's public dataset; its other 13,942 sessions are copies of Entire's or of its own, kept once |
| OpenHands feedback | 229 | OpenHands' public dataset of user sessions with feedback |

A session found in several places is kept once, and so is a conversation saved twice (an earlier save of a longer
chat, an export copied again): the fullest copy is kept. The others stay in the session tables with `duplicate_of`
set, and they have no rows.

## Columns you will use

| Column | Meaning | Empty (null) on |
|---|---|---|
| `session_key`, `turn_number` | The session, and the row's position in it. | no row |
| `agent` | The coding agent. | rows whose source does not name it: 15% of SpecStory's, 11% of DataClaw's |
| `kind` | What the row is: `text`, `thinking`, `tool_call`, `tool_result`, `notice`, `command`, `queued`, ... | no row |
| `actor` | Who made it: `developer`, `agent`, `tool`, `harness` (the agent program's own text) or `peer`. | no row |
| `typed_by_developer` | `true` if a person may have typed the message: every message a person wrote is marked, and where the format records no author, some a program wrote are too. | no row |
| `copy_of` | Set on a copy of a message stored elsewhere: the delivered copy of one queued while the agent was busy, or -1 on the developer's messages a forked helper thread inherited. Skip these rows to count each message once. | every row but those copies |
| `content` | The text, with secrets replaced by `[REDACTED]`. | rows with no text: progress and snapshot rows, queue bookkeeping, most notices, reasoning stored hidden, empty tool outputs. Some hold an empty string instead of null, so test for both |
| `tool_name`, `command`, `file_path` | What the agent ran, and on what. | rows that are not tool calls or outputs; `command` is only on shell calls (and on commands a developer entered), `file_path` only on calls that name a file |
| `category`, `bash_category` | The kind of tool call (`Action`, `Research`, ...) and of shell command (`git`, `test/build`, ...). | rows that are not tool calls or outputs; `bash_category` also on other tools than the shell |
| `is_error` | The tool reported a failure. | every row but tool outputs (and 476 status notices), and the 19% of outputs whose format records no failure |
| `model`, `input_tokens`, `output_tokens` | The model, and its token counts for each call. | rows the agent did not write. `model` is missing from all of Aider and OpenHands feedback; the counts are on one row per call, and missing from all of Aider, SpecStory and OpenHands feedback |
| `pushback` | `true` if a typed message pushes back on the agent's work. Written by our model. | every row not typed, and 1,794 typed rows ([below](#labels-rules-and-a-model)) |
| `prompt_intent`, `prompt_pushback`, `language` | SWE-chat v2's own labels of its prompts, copied as they are. | every row but the prompts SWE-chat v2 labelled, which are only in the `swe-chat-v2` and `entire` folders |

A column that does not apply to a row is empty there. The [column reference](#column-reference) says, for every
column of every table, where it is empty.

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
| The raw logs | `files` in the session tables; a copied dataset's session names its files in `copied/` in `source_facts` (`published_files`) |

## Labels: rules and a model

Claude Code, Codex, Gemini CLI, Copilot CLI, Cursor, OpenCode and Pi record who typed a message, and we read
those records. SpecStory, Aider and DataClaw record no author, and neither do unattended runs (`codex exec`,
`claude -p`). There a user message counts as typed unless its text is plainly a program's, such as a notice, a
template or a button's text. The rule is built to miss no message a person wrote: when it cannot tell, it marks
the message typed. A turn SpecStory's export kept without its content (pasted output, an image) is marked typed,
with no text.

`pushback` is the one label we had a model write. gpt-6-luna read each typed message on its own, without the rest
of the session, and answered yes or no. Of version `2026-10-11`'s 460,302 typed rows, it marked 137,766 as pushback.
1,794 have no label: Azure's content filter refused 54, and 1,740 hold no text. The instructions the model got are in the version's
`changelog.json` (`pushback_labels`).

SWE-chat v2's authors labelled its prompts with their own model: `prompt_intent`, `prompt_pushback` and
`language`. We copy those values unchanged onto the same messages, for the sessions SWE-chat v2 holds (in the
`swe-chat-v2` and `entire` folders), and label nothing in these columns ourselves. In a Claude Code session they
can sit on a queued message's row (`queue_operation`) or an entry Claude Code marks as its own (`system_injected`),
which SWE-chat files as prompts. Our `pushback` is separate: it
covers the typed messages of every source, SWE-chat's included. SWE-chat's three columns are empty on every
other row, and so is `pushback` on every row not typed.

## How accurate the labels are

We drew a random sample of 1,098 rows of version `2026-10-04` and compared the dataset's labels with what two
readers saw in each row.
Both readers were Claude models working from one written guide, not people. Each typed message they disagreed
with the dataset on was checked against the session's raw file. Where the readers differ, both results are shown.

| What was checked | Sample | Result |
|---|---|---|
| A message a person wrote that is not marked typed | 278 user rows not marked typed | 1, in a Codex helper thread; that rule was fixed and the data corrected |
| A row marked typed that a program wrote | 300 rows marked typed | 1–3% overall; DataClaw 8–21%, every other source under 5% |
| `pushback` | 200 typed messages | agrees with the readers on 88%; the readers agree with each other on 94% |
| A near-copy: the same conversation saved twice | 100 pairs | 100 of 100 |
| A row's `kind` and `actor` | 120 rows | 117–119 of 120 |
| An unmasked string that looks like a key | 100 strings | none is a working secret |

The two readers agreed on every typed message both could decide (526 of 526).

## Limitations

- `typed_by_developer` errs toward marking: where a format records no author, some messages a program wrote are
  marked typed, most of them in DataClaw, where helper agents' task briefs read as typed.
- `pushback` is one model's reading of one message without its context; use it to find candidates, not as truth.
- Sources are lopsided: 90% of Aider's sessions are one benchmark script, and one repository is 11% of Entire.
- Masking catches most secrets, not every one: a password written out in a message is not caught. The
  `redactions` column counts what was masked in each session.
- The secret scanner also takes ordinary names for keys (a dtype, model and class names, settings' names, paths).
  306 such texts are shown again: none has the shape of a key, and each is mostly dictionary words or appears
  unmasked elsewhere in the dataset at least as often as it was masked. A name that has the shape of a key stays
  masked.
- 1,738 SpecStory turns hold no text: the export kept the turn but not what the person sent.
- SWE-chat v2's own labels sit on 65% of the prompts it labelled (measured on version `2026-10-04`). It labels
  each block of a prompt on its own, blocks a tool inserted included (environment details, instruction files,
  referenced files' contents), and this dataset keeps those inside the message or as a notice of their own.
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
