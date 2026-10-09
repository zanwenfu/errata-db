# `conversations` columns

`conversations/<source>/*.parquet`: one row per piece of a message, in order: a prompt, a reply's text, a thinking step, a tool call, a tool's output, a notice. Rows of one session share `session_key`, and `turn_number` gives their order.

The first 38 columns follow SWE-chat v2's `conversations` table, in its order, so code written for SWE-chat runs on this table. The last 12 are added for this dataset.

"Empty" means null. Types are Parquet's: `string`, `text` (a string that can be long), `int`, `bool` and `timestamp` (UTC).

| Column | Type | Description | Notes |
|---|---|---|---|
| `turn_id` | string | `<session_id>#<turn_number>`, or `<session_key>#<turn_number>` where the agent records no session id. |  |
| `session_id` | string | The agent's own session id. | Empty for Aider and OpenHands feedback, and for about half of SpecStory's sessions. |
| `checkpoint_pk` | string | Entire's `<repository>#<checkpoint id>` the files were read from. | Entire and SWE-chat v2 only. |
| `repo_id` | string | The GitHub repository, as `owner/name`. | Empty for DataClaw and OpenHands, which record none. |
| `turn_number` | int | The row's position in its session, from 0. | With `session_key`, it identifies the row. |
| `conversation_turn_number` | int | The position among the dialogue rows only (prompts and replies), from 0. | Empty on other rows. |
| `source_entry_uuid` | string | The id of the log entry the row came from. | Where the format records one; never on Aider's or SpecStory's rows. |
| `response_id` | string | The id of the model's API response. | Agent rows, where the format records one; never on Aider's, SpecStory's or OpenHands feedback's rows. |
| `content_block_index` | int | Which piece of its message the row is, from 0. |  |
| `role` | string | SWE-chat's role: `user`, `assistant`, `tool_use`, `tool_result`, `metadata`. | Who wrote the row is in `actor`. |
| `turn_type` | string | SWE-chat's row type: `user_prompt`, `assistant_response`, `assistant_thinking`, `tool_use`, `tool_result`, `system_injected`, `system_event`, `queue_operation`, `progress`, `file_snapshot`, `summary`. |  |
| `is_conversational` | bool | `true` for prompts and replies (`user_prompt`, `assistant_response`). |  |
| `content` | text | The row's text, with secrets as `[REDACTED]`. Images and documents appear as `[Image: image/png]`. | A tool's output over 100,000 characters is cut; see `content_truncated`. Empty, as null or as an empty string, on rows with no text: progress, snapshot and queue-bookkeeping rows, most status notices, reasoning the agent stored hidden (Codex's reasoning, much of Claude Code's thinking), empty tool outputs, and 1,734 SpecStory turns kept without what the person sent. Test for both. |
| `model` | string | The model that wrote the row. | Agent rows (replies, thinking, tool calls) where the format records it; empty on every other row, and on every Aider and OpenHands feedback row. |
| `timestamp` | timestamp | When the log entry was written. | Empty where the format records no time: every Aider row, and 43% of SpecStory's. |
| `input_tokens` | int | The model call's input tokens. | Once per call, on its first text row (else its first tool call), and only where the format records usage: Entire's and SWE-chat v2's sessions and 1% of DataClaw's rows, never Aider, SpecStory or OpenHands feedback. Anthropic models count without cache reads; OpenAI's (Codex) include cached input. |
| `output_tokens` | int | The model call's output tokens. | Placed as `input_tokens`. |
| `cache_creation_input_tokens` | int | Tokens the call wrote to the prompt cache. | Placed as `input_tokens`. |
| `cache_read_input_tokens` | int | Tokens the call read from the prompt cache. | Placed as `input_tokens`. |
| `is_continuation` | bool | `true` on a user row opening with Claude Code's "This session is being continued..." summary. |  |
| `is_first_turn` | bool | `true` on the session's first dialogue row. |  |
| `word_count` | int | Words in `content`, split on whitespace. |  |
| `char_count` | int | Characters in `content`. |  |
| `tool_name` | string | The tool called, as the agent names it. | Tool calls and their outputs only; a tool's output carries its call's. Aider records no tool calls. |
| `tool_call_id` | string | The call's id. | A tool's output carries its call's, which pairs the two. Empty where the format records no ids: every SpecStory row, and some calls of other formats. |
| `file_path` | string | The file a tool call works on. | Read from the call's input; only on calls that name a file. |
| `command` | string | The shell command a call ran, or a command the developer entered. | Shell calls. Also on a developer's command row (`kind` `command`, a slash command such as `/model`) and shell row (`kind` `shell`, a command they ran themselves), where it holds what they entered. |
| `pattern` | string | A search tool's pattern. | Search tools only (`grep`, `glob`, ...). |
| `tool_input_json` | text | The call's whole input, as JSON. | Tool calls only; empty where the format records no input (13% of calls). |
| `orchestration_kind` | string | `subagent_spawn` on a call that starts a helper agent. | Empty otherwise. |
| `category` | string | The tool call's kind, by SWE-chat's rules: `Action`, `Research`, `Orchestration`, `Other`. | Tool calls and their outputs only; a tool's output carries its call's. |
| `bash_category` | string | A shell command's kind: `file ops`, `git`, `package manager`, `test/build`, `other`. | Empty where no command was recorded. |
| `queue_op_subtype` | string | For a message queued while the agent worked: `user_prompt_enqueued`, `user_prompt_delivered`, `user_prompt_discarded`, `task_notification`, `other`. | Claude Code's queue rows only. |
| `agent` | string | The coding agent. | Empty where the source does not name it. |
| `strategy` | string | How Entire saved the session: `manual-commit`, `auto-commit`, `import`, ... | Entire and SWE-chat v2 only. |
| `language` | string | The prompt's language, by SWE-chat: `ENGLISH`, `CHINESE`, ... | Only on the prompts SWE-chat v2 labelled, in the sessions it holds (the `swe-chat-v2` and `entire` folders); empty on every other row, and on every row of the other four sources. |
| `prompt_intent` | string | SWE-chat's label of what a prompt asks: `understand`, `debug`, `create new code`, `refactor`, `test`, `git`, `connect`, `other`. | Written by a model, for SWE-chat. Only on the prompts SWE-chat v2 labelled, in the sessions it holds (the `swe-chat-v2` and `entire` folders); empty on every other row, and on every row of the other four sources. |
| `prompt_pushback` | string | SWE-chat's pushback label: `correction`, `rejection`, `failure_report`, `non_pushback`. | Written by a model that read the whole session, for SWE-chat. Only on the prompts SWE-chat v2 labelled, in the sessions it holds (the `swe-chat-v2` and `entire` folders); empty on every other row, and on every row of the other four sources. |
| `source` | string | `entire`, `specstory`, `aider`, `swe-chat-v2`, `dataclaw` or `openhands-feedback`. |  |
| `session_key` | string | `<source>:<source_id>`. It links the row to the session tables. |  |
| `typed_by_developer` | bool | `true` if a person may have typed this message. | Every message a developer wrote is marked; where the format records no author (SpecStory, Aider, DataClaw, unattended runs such as `codex exec` or `claude -p`), a message no rule names is taken as theirs, so some a program wrote are marked too. A turn SpecStory's export keeps without its content (pasted output, an image) is marked with no text (1,734 rows). |
| `actor` | string | Who made the row: `developer`, `agent`, `tool`, `harness` (the agent program's own text), `peer` (another agent or a teammate). |  |
| `kind` | string | What the row is: `text`, `thinking`, `tool_call`, `tool_result`, `notice`, `command`, `shell`, `queued`, `compaction`, `interrupt`, `rejection`, `plan`, `progress`, `snapshot`. | Finer than `turn_type`. |
| `is_error` | bool | A tool's output, or a status notice, marked as a failure. | Set on tool outputs where the format records it, and on 401 run-status notices of a few agents' runners (`system_event`); empty elsewhere. |
| `copy_of` | int | On the delivered copy of a queued message, the `turn_number` where it was queued; -1 on a developer's message a forked helper thread (Codex) inherited from its parent's conversation. | Skip these rows to count each message once. |
| `parent_uuid` | string | The log entry this one follows from, as the format records it. | Where the format records one; never on Aider's, SpecStory's or OpenHands feedback's rows. |
| `content_truncated` | bool | `true` if a tool's output was cut at 100,000 characters. | Set on tool outputs only; empty elsewhere. The whole output is in the session's `files`. |
| `raw_file` | string | The file in the session's `files` the row came from. |  |
| `raw_line` | int | The row's line in that file. | Empty where the file is one JSON document rather than a line per entry: OpenCode's sessions and Gemini CLI's JSON files. |
| `pushback` | bool | `true` if a typed message pushes back on the agent's work: corrects or redirects it, rejects or reverts it, or reports it broken. | Written by gpt-6-luna from the message alone. Typed rows only, empty on every other row; 1,785 typed rows have no label: 49 the model's content filter held back, 1,736 with no text to read. |

Other tables: [session tables](sessions.md), [`conversations_subagents`](conversations_subagents.md). The [README](../../README.md) says where the data is and how to get it.
