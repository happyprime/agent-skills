# Stage 2 — Targeted Debate (sub-agent prompt template)

Stage 2 is **optional and conditional**. It runs only when synthesis (Stage 3) flagged a finding in `needs-debate.md` — typically because two personas disagreed on severity by more than one tier, or proposed contradictory fixes. Stage 2 is invoked as a follow-up; it does not run automatically.

The orchestrator inlines this text into a single sub-agent that plays multiple persona roles through one structured round.

---

You are running a structured debate on a contested finding. Your output is either a **resolution** (the finding is updated in place with a defended severity and fix) or an **escalation** (a new entry in `decisions-needed.md` because the disagreement is a genuine product/architecture choice).

## Inputs

- **Disputed finding.** Provided inline: `${FINDING_PATH}` and its full text.
- **Contributing personas.** The personas listed in the finding's `reviewers:` field. Their full persona files are provided inline.
- **Raw findings from each persona.** Their original raw finding files (pre-synthesis) under `${RUN_DIR}/raw/` are provided inline so you have each persona's original framing.
- **Run directory.** `${RUN_DIR}` — for writing updates.

## Procedure

You will play each persona in turn. Do not let one persona's voice contaminate another. When you switch personas, switch fully — reread the persona file before writing.

### Round 1 — Opening positions

For each persona, in the order they appear in the finding's `reviewers:` array:

```markdown
## Opening — <persona-name>

**Position:** <one sentence — severity I'd assign and the fix I'd propose>

**Reasoning:** <one paragraph in this persona's voice, citing their rubric and primary categories>

**Evidence cited:** <file:line references>
```

### Round 2 — Direct response

For each persona, respond to the other personas' openings.

```markdown
## Response — <persona-name>

<One paragraph. Address the strongest point made by another persona. Either concede, refine your position, or explain why their lens leads to the wrong conclusion here.>
```

### Round 3 — Synthesis attempt

Switch to the senior-engineer persona. Read the openings and responses. Try to land on a unified position.

```markdown
## Synthesis attempt

**Re-reading the evidence:** <one paragraph — what's actually in the code, stripped of persona framing>

**Where the personas agree:** <bullet list>

**Where they disagree:** <bullet list — be specific about whether it's severity, fix, or both>

**Resolution:** <One of: "RESOLVED — see updated finding" OR "ESCALATE — this is a decision the user must make">
```

### Round 4 — Outcome

#### If RESOLVED

Update the finding file in place at `${FINDING_PATH}`. Keep the original `id`, `title`, and `reviewers`. Update `severity`, `confidence`, and the body sections as the synthesis dictates. Add a `## Synthesis notes` section at the bottom of the body with:

```markdown
## Synthesis notes

This finding went through Stage 2 debate on <date>. Contributing personas: <list>.

**Original disagreement:** <one or two sentences>

**Resolved position:** <one paragraph — what was decided and why>

**Severity locked at:** <severity> (confidence: <high|medium|low>)
```

Also: remove this finding's entry from `needs-debate.md`. If `needs-debate.md` is now empty, delete the file.

#### If ESCALATE

Do not modify the finding. Instead, append an entry to `decisions-needed.md`:

```markdown
## <Finding ID> — <title>

**Affected finding(s):** <ID list>

**The question:** <one or two sentences — the actual choice the user needs to make>

**Options:**

- **A.** <one option with its cost / trade-off>
- **B.** <another option with its cost / trade-off>
- **C.** <if relevant>

**Recommendation:** <one option, named, with confidence>

**Reasoning:** <one paragraph — why this recommendation, and what would change it>

**Contributing personas:** <list>
```

Also remove the finding's entry from `needs-debate.md` (since it's been processed) and clean up the file if empty.

## Discipline reminders

- One round per stage. Don't drift into a multi-turn argument; that's noise.
- Each persona stays in lane during rounds 1 and 2. The senior persona synthesizes in round 3 — only there.
- The senior persona has the authority to overrule both specialists' severity ratings via the cross-domain rubric.
- If a debate exposes that the original finding was actually two findings (different issues conflated), split it: update one, file a new one. Note the split in the synthesis notes.
- Don't change `id` or `title` unless splitting; downstream artifacts may reference them.
