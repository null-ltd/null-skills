---
name: handoff
description: Write a resume prompt for a cold successor session.
disable-model-invocation: true
---

Compact this session into a prompt the next agent can run on cold. It is not a status report and not a summary for humans. The reader is an agent with none of your context, and the file is its opening prompt.

## 1. Fix the forward purpose

Write the handoff for what the next session will *do*: continue the implementation, debug the failure, review the approach, pick up on another machine. If the user stated the purpose, use it. If not, ask before writing anything; a handoff without a purpose keeps everything and helps nothing.

Purpose set, apply it as the filter: keep what the next session needs to act, cut what it doesn't. A debugging successor needs the failure and the ruled-out causes; it does not need the feature's design history.

Done when you can state in one sentence what the successor will do first.

## 2. Compose the handoff

Sections, in order:

- **Mission.** What is being attempted and why, 2 to 3 sentences. Name the issue/PR if one exists.
- **Settled.** Decisions already made, each as one line *linking to where the decision lives* (a design document, an issue, a commit). Never restate a decision the link carries; a restatement goes stale, the link doesn't. A decision that lives only in this conversation is written out here; this file is its only record.
- **Tried and rejected.** Approaches attempted this session that failed or were abandoned, and why. This is the one thing no other artifact records; losing it means the successor re-walks dead ends. Be specific: what was tried, what happened, why it's out.
- **Current state.** Where the work stands. If resuming depends on a location not already identified in this handoff, name the repository and checkout, document, issue, or other resource. Include branch, commit, and uncommitted files when relevant; conversation-only work needs no external location. Then give the blocker or failure *verbatim*: paste the exact error text, failing test name, or command output, not a paraphrase.
- **Next action.** The exact first move: a command to run, a file and the change to make, a question to answer. The successor should be able to act without re-deriving anything.
- **Open questions.** Each with who or what can resolve it (the user, a file to read, an experiment to run).

Completion test: an agent that has never seen this conversation could resume from the file alone, and every rejected approach it might otherwise retry is on the list.

## 3. Redact

The successor reads this file verbatim. Before writing it, strip secrets, API keys, tokens, connection strings, credentials in URLs, and PII. Replace with a placeholder naming where the real value lives (`$STRIPE_KEY`, lives in `.env`). Verbatim error output is the most common leak; scan it.

## 4. Deliver the path

Write the handoff into the directory the project's context files name for handoffs; when none is named, ask where before writing. File name `YYYY-MM-DD-<slug>.md`, slug from the work (`2026-09-01-auth-token-refresh.md`). The path is the deliverable; the user hands it to the next session. End your reply with the full absolute path, prominent and on its own line.
