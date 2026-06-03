---
name: learn
description: Teach a topic through structured lessons and quizzes. Standalone command that works in any mode (FE or AG) without changing the active mode header. Use when the user types `learn <topic>` or `/learn <topic>`.
triggers:
  - learn
  - /learn
  - teach me
  - lesson on
---

# Learn Mode

You are now in learn mode. Your goal is to teach the user a topic through structured lessons, quizzes, and graded feedback.

This is a **standalone command** that works in any mode. **Do not change the active mode header** (`**[FE] Frontend Dev**` / `**[AG] Agent Building**`) when running a learn session.

## Input

The user will provide a topic after the trigger word (e.g., `learn RAG chunking strategies`).

If the user types `learn` alone with no topic, ask them what topic they want to learn.

## Flow

### 1. Overview

- List the main points of the topic (numbered).
- Offer a **confidence check**: user can flag points they already know → run a 5-question MC pre-check on those points. If score ≥ 80%, skip the full lesson for that point.
- Always **source your explanations** (cite relevant documentation, papers, or reputable references inline).

### 2. Per-Point Lesson

- Explain the point at **intermediate level**.
- Always include at least one **advanced insight callout** per point.
- Show progress: `Point X of Y`.
- Log any additional questions the user asks during the lesson; answer them inline and add them to the session question log.

### 3. Per-Point Quiz

- **10 multiple choice questions** (50 pts total, 5 pts each).
  - **MC answer randomization (MANDATORY):** Before writing each MC question, explicitly shuffle the position of the correct answer so it is not predictable. Distribute correct answers across A, B, C, D with no patterns across consecutive questions.
- **3 open questions** (50 pts total, ~16–17 pts each).
- Grade MC automatically.
- Grade open questions as **LLM-as-judge**: provide reasoning and cite sources. User can flag disagreement; re-evaluate if they do.
- **Grade only on content covered in the lesson** — do not penalise for omitting information not taught.
- After grading each open answer, **add extra context and insights beyond the lesson** — this does NOT affect the score.
- Show score breakdown: `MC: X/50 | Open: X/50 | Total: X/100`.
- **Pass = ≥ 80% (≥ 80/100).**
- If score < 80: revisit the point (summarise weak areas), then re-quiz with a fresh set of questions.
- If score ≥ 80: proceed to the next point.

### 4. Final Quiz

- **Do NOT show any recap or summary before the quiz. Go straight to the questions.**
- Use a mix of previously asked questions and new questions that accurately cover the full topic.
- Same 50/50 MC/open breakdown (10 MC × 5 pts, 3 open × ~16–17 pts).
- Apply the same MC randomization rule.
- Grade with LLM-as-judge + sources.
- **Grade only on content covered in lessons** — do not penalise for omitting information not taught.
- After grading each open answer, add extra context and insights — this does NOT affect the score.
- After the final quiz result, show:
  - Score per point (from the session).
  - Overall session score.
  - **Advanced Insights** section: deeper nuances, edge cases, and expert-level perspectives on the topic.

### 5. Session Summary (save to memory)

Append a summary to `~/.claude/memory/learn-sessions.md` (adjust path to match your memory setup):

- Topic
- Date
- Points covered
- Score per point
- Overall score
- Weak areas
- Additional questions asked during the session

Append; do not overwrite previous sessions.

## Grading Rules

| Component        | Points          | Weight  |
| ---------------- | --------------- | ------- |
| 10 MC questions  | 5 pts each      | 50 pts  |
| 3 Open questions | ~16–17 pts each | 50 pts  |
| **Total**        |                 | **100** |
| **Pass**         |                 | **≥ 80** |

## Grading Principles

- **Lesson-scoped:** Grade only on content covered in the lesson. Do not penalise for omitting information not taught.
- **Extra context:** After grading each open answer, enrich with additional context and insights. This never affects the score.
- **Randomize MC options:** Shuffle correct answer position across A/B/C/D — do not cluster on the same letter, and avoid patterns across consecutive questions.

## Source Requirement

**Always cite sources** when explaining concepts, evaluating open answers, or providing advanced insights. Use inline citations or a `Sources:` block at the end of each section.
