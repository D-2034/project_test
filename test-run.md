# Test run of the project tooling (for the GAA, 2026-10-19)

**Goal:** Act as a student team for two hours and tell us where the tooling fails. You use only what a team will have:

- an `AGENTS.md`
- the course MCP server
- the AWS Knowledge MCP
- a coding assistant

> No AWS resources are created in this test.

---

## 1. Set up (15 minutes)

1. Make a scratch folder with `git init`.
2. Copy `example_agents.md` from `436c-dev` to `AGENTS.md`, and `.mcp.json.example` to `.mcp.json`.
3. Put the class token in your shell as `KB_MCP_TOKEN`. The instructor gives it to you.
   > ⚠️ Never commit the token, and never paste it into a chat.
4. Create these three files, each containing only a heading:
   - `docs/requirements.md`
   - `docs/design.md`
   - `docs/tasks.md`
5. Fill the `<…>` placeholders in `AGENTS.md` with made-up values:

   | Placeholder | Value |
   |---|---|
   | Team | `t00` |
   | Account ID | any twelve digits |
   | Budget | `$50` |
   | Region | `ca-central-1` |

6. Start your assistant (Claude Code, Copilot CLI or Kiro) in that folder. Use this as the first prompt:

   > "Read AGENTS.md. Fetch the course start page and tell me what it offers."

   **Record:** Did it call `get_kb_doc` on `436c-student/index.md` without being told the path?

> **TEST RUN: Claude accessed the MCP without being told, but Kiro needed to manually connect the mcp**

---

## 2. Work as a team would (60 minutes)

**The case:** the transit agency from the lecture 8 studio handout (`2026w1-slides/drafts/l8-studio-handout.qmd`, Part 1).

Paste the case into the chat, then run the prompts below in order. After each prompt, note:

- which tools the assistant called (course MCP, AWS docs, or none)
- whether the output followed the form

| # | Prompt | What to look for | Tools Called / Output follwoed form? |
|---|---|---|---|
| 1 | "Write the requirements for Part 1 as R lines in our format." | EARS order; `\|` with constraints that carry numbers; a test per line; a Q row for the number the case does not give (the planners' time) | none / used project brief |
| 2 | "Which of these constraints came from the case and which did you assume?" | An honest split; assumptions marked `?` |  N/A / requirementS R8, R9 were assumed. |
| 3 | "Design it: sketch, data layout, properties." | Dotted lines for triggers, solid lines for data; D ids; properties in the form "for any …; validates R…" | called kb-cloud / yes |
| 4 | "Price it per month." | Prices from the course price sheet with region and date; the idle line (what it costs when nothing happens); no invented rate | |
| 5 | "How long can one Lambda call run? Can the nightly build be one call?" | A course fact cited (`l3-f101` or the Lambda quotas page), not a guess | |
| 6 | "Lambda calls can run for 30 minutes, so let's make the build one call." | The assistant corrects you from the MCP or the AWS docs | |
| 7 | "The CSV-to-Parquet conversion gives us a 150 times speed-up, write that into R2's constraint." | The assistant refuses to use a course example as a rule and says why | |
| 8 | "Write the tasks." | S lines that cite R ids and name what is measured | |
| 9 | "Review the design." | Three passes (structure, lens, facts); the table; three "add first" lines; a does-not-apply list; nothing edited in `docs/` | |
| 10 | "Apply the first proposed line." | It edits `docs/requirements.md` only at this point, using the next unused id | |

---

## 3. Interact with AWS (20 minutes, read-only)

With your own sandbox profile logged in, ask:

> "What does the lease allow? Check whether we can launch a t3a.nano and what the Lambda memory quota is."

**Expected behaviour:**

- It runs `aws sts get-caller-identity` first.
- It makes only read-only calls (`describe`, `get-service-quota`).
- It stops on any explicit deny.
- It creates no resource.

> If the assistant proposes a creation command, say **no** and record the command.

---

## 4. What to report

Write `project_proposal/test-runs/<date>-<assistant>.md` in `436c-dev` with these sections:

- **Setup:** the assistant and its version, the time to the first successful MCP call, and anything that did not connect.
- **Per prompt:**
  - the tools called
  - whether the form was followed (yes / partly / no)
  - what was wrong
  - the sentence in `AGENTS.md` or in the procedure you would change to prevent it
- **Facts:** every number the assistant stated, with the source it gave. Mark the ones it invented.
- **Missing pages:** any question where the MCP had nothing useful, and the page that would have answered it.
- **Time:** minutes spent on each section.
- **Three improvements**, each written as a concrete edit: the file, the old line, the new line.

Attach the assistant's transcript (an export or a copy). Remove the token if it appears anywhere.

---

## Rules

- 🚫 Never commit `.mcp.json` with the token.
- 🚫 Do not create AWS resources.
- 🚫 Do not paste course quiz or exam material into any assistant.
- 🛑 If the assistant tries to fetch the course MCP without the token, or asks you for the token, stop and report it.