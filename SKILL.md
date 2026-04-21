---
name: qa-recorder
description: Retrospectively extracts Q&A pairs from the current conversation and saves selected ones to a user-specified file. Triggered by /qa-recorder or the keyword "启动记录" in the prompt.
---

# QA Recorder Skill

## Design Philosophy

This skill is a **one-shot task**, not a persistent mode. It activates, does its job, and exits. There is no "recording mode", no session state to maintain, and no risk of state confusion. The user can invoke it at any point in the conversation — even retroactively after a long exchange.

## Trigger Conditions

Activate when:
- The user invokes `/qa-recorder`, OR
- The user's prompt contains the keyword **"启动记录"**

Optional inline parameters at trigger time:
- `path:D:/notes` — set the output folder
- `file:myfile.md` — set the output filename
- `last:N` — only extract questions from the last N user messages

Example: `启动记录 path:D:/notes file:python问题.md last:10`

## Execution Flow

### Step 1 — Extract questions from conversation history

Determine extraction scope:
- If `last:N` was specified at trigger time, only scan the last N user messages.
- Otherwise scan all visible conversation history.
- If `last:N` was not specified but more than 20 questions are found, after presenting the list add:
  > "共识别到 N 个问题，如需只保存某个范围，可告诉我（如「只要最后 5 个」），我可以重新提取。"

Apply these extraction rules:
- Include: genuine questions seeking information, explanation, or advice
- Exclude: meta-instructions ("修改一下"、"再试一次"), skill invocations, control commands, and casual confirmations
- If the conversation was partially compressed and early questions are unavailable, note explicitly: "（注：对话较长，早期部分已压缩，以下仅包含可见范围内的问题）"

Number each extracted question sequentially (Q1, Q2, ...).

**Follow-up question detection**: For each extracted question, determine if it is a follow-up that depends on prior context. A question is a follow-up if it:
- Contains pronouns or demonstratives referring to earlier content: 那、这、它、这个、那个、这种、这样、上面、前面
- Lacks a self-contained subject (e.g., "怎么实现？" with no stated subject)
- Uses comparative or additive framing implying a prior topic: "那如果…"、"那换成…"、"还有呢"

For each detected follow-up, attempt **Option A — rewrite to be self-contained**: replace pronouns and implied subjects with the actual referents from context. For example:
- `那用 Python 怎么做？` → `用 Python 实现 [the specific task from prior context] 怎么做？`
- `它有什么缺点？` → `[the specific tool/concept discussed] 有什么缺点？`

If the rewrite would be ambiguous or too complex to do accurately, fall back to **Option B — prepend context block**: keep the original question text and add a background block in the saved entry.

Mark each question to indicate its type:
- `[独立]` — standalone question, no dependency
- `[已展开]` — follow-up successfully rewritten; show both original and rewritten text
- `[含背景]` — follow-up that will be saved with a context block

### Step 2 — Present the list

Show the extracted list with type labels. For `[已展开]` questions, display both versions so the user can verify the rewrite:

```
我从对话中识别到以下 N 个问题：

Q1 [独立]：[question text]

Q2 [已展开]：[rewritten question text]
    原始问题：「[original question text]」

Q3 [含背景]：[original question text]
    （保存时将附加背景说明）
...

请问要保存哪些？可以回复：
- 「全部」— 保存所有问题
- 编号列表，如「1 3 5」— 只保存指定问题
- 「取消」— 退出，不保存任何内容
```

### Step 3 — Collect answer decisions

After the user specifies which questions to save, present the default answers:

```
以下是每题的默认答案（来自本次对话）：

Q1：[question]
→ [answer — for plain text: summarize to key conclusions in ≤3 sentences;
   for code blocks, lists, or tables: reproduce in full without truncation;
   for very long plain text (>600 chars): summarize and append「（已摘要，原回答更详细）」]

Q3：[question]
→ [...]

需要修改哪些答案？（直接说「不用修改」或列出题号）
```

For each question the user wants to modify, ask for the replacement answer **one at a time**.

### Step 4 — Re-verification on request (optional)

If the user asks to re-verify an answer before saving:

- **Stable, well-known facts** (syntax, definitions, math): use training knowledge, no search needed
- **Time-sensitive or version-specific information**: perform a web search, cite inline as `[claim] — [Source Name](https://url)`
- **Trusted sources only**: official docs, standards bodies, established tech publications; not anonymous blogs or AI aggregators
- **Contested claims**: verify against two independent primary sources (two sites citing the same upstream source do not count as independent); present both sides if they conflict
- **Uncertainty**: if no reliable source found, say so explicitly

### Step 5 — Resolve output path and filename

**Path and filename resolution** (in priority order):

1. Both specified at trigger time via `path:` and `file:` — use directly, skip asking.
2. Either or both not specified — ask in a **single question**:
   > "请指定保存路径和文件名，例如：`D:/notes/python.md`
   > 也可以回复「桌面」使用桌面默认路径，或回复「默认」使用完整默认值（桌面 + `claude-qa.md`）。"

   Parse the user's reply:
   - "默认": desktop path + filename `claude-qa.md`
   - "桌面": desktop path + ask for filename separately (or use `claude-qa.md` if user replies "默认")
   - A full path like `D:/notes/python.md`: split into folder + filename directly
   - A folder path without filename (e.g., `D:/notes`): use that folder + filename `claude-qa.md`
   - A filename only without folder (e.g., `myfile.md`): ask for folder, or confirm using desktop

   If the user provides a filename without `.md` extension, append it automatically.

   Desktop path detection: run `powershell -command "[Environment]::GetFolderPath('Desktop')"`, normalize to forward slashes. Fall back to `$HOME/Desktop` if PowerShell is unavailable.

   If the resolved folder does not exist: warn and ask whether to create it or choose another path.

**Never hardcode a username or path.**

If the resolved file does not exist, create it with this header:
```markdown
# Claude 问答记录
```

### Step 6 — Preview before writing

Show a complete preview of what will be written:

```
以下内容将追加写入文件，请确认：

────────────────────
记录时间：YYYY-MM-DD HH:mm

问题：[Q1 text]
答案：[final answer]

问题：[Q3 text]
背景：[context summary]
答案：[final answer]
────────────────────

保存路径：[full resolved file path]

回复「确认 / 好 / 可以 / 行 / 写吧 / 没问题 / ok」写入，或告诉我需要修改什么。
```

Treat any of the following as confirmation: `确认`、`好`、`可以`、`行`、`写吧`、`没问题`、`ok`、`好的`、`是`、`对`

Treat any of the following as cancellation: `取消`、`不`、`算了`、`不用了`

Any other reply is treated as a modification request — continue the conversation.

Only proceed to write after confirmation is received.

### Step 7 — Write file

**Append** each confirmed entry (escape any `---` inside answer text as `\-\-\-`):

For `[独立]` and `[已展开]` questions:
```markdown

---
记录时间：YYYY-MM-DD HH:mm

**问题：** [question text]

**答案：** [final answer]

```

For `[含背景]` questions:
```markdown

---
记录时间：YYYY-MM-DD HH:mm

**背景：** [1–2 sentence summary of the context this question depends on]

**问题：** [original question text]

**答案：** [final answer]

```

**Deduplication**: before writing each entry, check if a substantially identical question already exists in the file. If yes, warn:
> "「[question]」与文件中已有记录相似，是否仍要写入？(是/否)"

**On failure**: report the exact error message and suggest an alternative path.

### Step 8 — Confirm and exit

```
✅ 已成功记录 X 条问答到：
[full resolved file path]

如需修改已保存内容，请直接打开文件编辑。
```

The skill exits after this message. No persistent state remains.

## Rules

- **Append only**: never overwrite existing file content
- **One-shot**: exits cleanly after saving or cancellation; does not linger
- **No silent failures**: report errors explicitly, always offer a fallback
- **UI language**: all prompts and messages to the user are in Chinese; this skill definition is in English
