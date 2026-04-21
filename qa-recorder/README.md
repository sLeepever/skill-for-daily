# qa-recorder

A Claude Code skill that retrospectively extracts Q&A pairs from your conversation and saves them to a Markdown file of your choice.

## What it does

Instead of running as a persistent "recording mode", this skill activates on demand, scans your conversation history, extracts questions you asked, and walks you through confirming and saving them — along with their answers — to a local file.

## Trigger

Invoke at any point in a conversation:

```
/qa-recorder
```

Or include the keyword in your message:

```
启动记录
```

### Optional inline parameters

| Parameter | Example | Description |
|-----------|---------|-------------|
| `path:` | `path:D:/notes` | Output folder |
| `file:` | `file:python.md` | Output filename |
| `last:` | `last:10` | Only extract from the last N user messages |

Full example:

```
启动记录 path:D:/notes file:python问题.md last:10
```

## Workflow

1. **Extract** — scans conversation history and numbers all genuine questions
2. **Label follow-ups** — detects context-dependent questions and either rewrites them to be self-contained (`[已展开]`) or flags them for a background block (`[含背景]`)
3. **Select** — you choose which questions to save ("全部" / "1 3 5" / "取消")
4. **Review answers** — shows the default answer for each question (full code/tables preserved, long prose summarized); you mark which ones to replace
5. **Set destination** — if not specified at trigger time, asks for path + filename in one prompt; defaults to `claude-qa.md` if you reply "默认"
6. **Preview** — shows the full content to be written before touching the file
7. **Write** — appends to the file (creates it if missing); checks for near-duplicate entries before writing
8. **Exit** — confirms the saved path and exits cleanly

## Answer quality constraints

When active, answers follow these rules:

- **Stable facts** (syntax, definitions, math): answered from training knowledge
- **Time-sensitive or version-specific info**: web-searched with inline citations — `[claim] — [Source](https://url)`
- **Sources**: official docs, standards bodies, established publications only — no anonymous blogs or AI aggregators
- **Contested claims**: verified against two independent primary sources; conflicts presented explicitly
- **Uncertainty**: disclosed directly rather than guessed

## Output format

Entries are appended to the target file:

```markdown
---
记录时间：2026-04-21 14:30

**问题：** How does Python's GIL affect multithreading?

**答案：** ...

```

Follow-up questions that couldn't be rewritten get a context block:

```markdown
---
记录时间：2026-04-21 14:31

**背景：** Discussion was about Python threading vs multiprocessing.

**问题：** 那换成 asyncio 呢？

**答案：** ...

```

## Notes

- **Append only** — never overwrites existing content
- **Deduplication** — warns before writing a question that's substantially identical to one already in the file
- If the output folder doesn't exist, you'll be asked whether to create it
- If the conversation was partially compressed, the skill notes which questions may be missing
