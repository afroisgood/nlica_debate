# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

辯論計時器 — a two-page web app for a live debate event "生活之間：從住民到公民". No build step, no framework, no package manager. Pure HTML/CSS/JS deployed to GitHub Pages.

**Live URL**: https://afroisgood.github.io/debate-timer/

## Deploy

```bash
git add index.html vote.html
git commit -m "..."
git push origin main   # GitHub Pages auto-deploys from main branch
```

Changes are live within ~1 minute of push. No build or CI step.

## Architecture

### Two pages

| File | Role |
|---|---|
| `index.html` | Admin / main display — shown on the presenter's screen. Controls timer, manages stages, displays live votes and comments, draws/reveals audience questions. |
| `vote.html` | Participant page — opened on phones via QR code. Submits votes, comments, likes, topic ballot votes, and anonymous questions for 正方/反方. |
| `questions.html` | Standalone admin page (separate URL, own password gate) for moderating submitted questions — not linked from index.html. |

### Firebase Realtime Database (compat SDK 10.12.0)

All state is shared via Firebase. No backend code exists — everything is client-side JS talking directly to the DB.

**Data paths:**
```
debate/
  config/
    currentStage      String  — stage name synced from index.html on every loadStage()
    votingOpen        Boolean — controls whether 觀眾投票 is accepting votes
    topicVotingOpen   Boolean — controls whether 下期題目票選 is accepting votes
    voteRevealed      Boolean — controls fullscreen vote-result overlay on index.html
    questionRevealed  Boolean — controls fullscreen drawn-question overlay on index.html
    selectedQuestion  Object  — { id, text, side } of the currently drawn question (no nickname — questions are anonymous)
  votes/
    {deviceId}/      { nickname, side, updatedAt }
  comments/
    {pushId}/        { nickname, text, side, createdAt, deleted }
  likes/
    {commentId}/
      {deviceId}     true
  topicVotes/
    {deviceId}/
      {topicIndex}   true
  questions/
    {pushId}/        { text, side, createdAt, drawn, hidden }  — no nickname; submitted anonymously from vote.html at any time
```

**Firebase rules** must explicitly grant `.read`/`.write` per sub-node. Reading a parent node (`debate/`) fails silently if the parent has no `.read` rule — always read individual sub-nodes.

### Stage system (index.html)

`STAGES` array drives everything: each entry is `[name, durationSeconds]`. Duration `0` = no timer (end states). Special stage names trigger different UI modes:

- `"自由辯論"` → dual-circle timer (正方/反方 independent countdowns)
- `"下期題目票選"` → full-width topic ballot display, 8-minute timer, auto-closes voting at 0
- `"活動結束"` → terminal state

`loadStage()` is the central function: stops timer, resets state, syncs `currentStage` to Firebase, shows/hides UI panels.

### Admin gate (index.html)

All timer/nav controls are `disabled` on load. `setControlsEnabled(bool)` enables/disables: prev/next stage, toggle, reset, ±time buttons, stage-select dropdown, dot bar navigation, switch-side button, and the reset-topic-votes button. Password is hardcoded as `ADMIN_PASSWORD`.

Keyboard shortcuts (space/arrows/R/F11) are also blocked when not admin.

### vote.html stage switching

`vote.html` listens to `debate/config/currentStage` and switches between three section states:
- Normal: shows `#vote-section`, `#comment-section`, `#feed-section`
- Topic ballot (`"下期題目票選"`): shows `#topic-vote-section` only
- Voting closed: `votingAllowed` flag blocks `castVote()`; `topicVotingAllowed` flag blocks `handleTopicVote()`

### Audience questions (提問正方/反方)

Unlike voting/comments, question submission is **not** tied to a specific stage. In `vote.html`, a persistent `#question-fab` ("🙋 提問" button, always visible) opens `#question-modal`: the participant picks 正方 or 反方 first, then types the question. Submissions are anonymous — no `nickname` field is written — and push to `debate/questions`.

`questions.html` is a separate, unlinked admin page (own password gate, same `ADMIN_PASSWORD`) for moderating the pool: lists questions per side with live counts, "刪除" soft-deletes a question (`hidden: true`), "清空全部提問" soft-deletes everything currently visible. Hidden questions are filtered out everywhere (admin list and draw candidates) but never physically removed, matching the soft-delete convention used for `comments`.

On `index.html`, the draw flow itself is unchanged from before and still stage-gated: the `#draw-question-btn` admin control only appears during `"觀眾提問正方"` / `"觀眾提問反方"` (`.admin-active.qna-active` gate, `qna-active` toggled in `loadStage()`). Clicking it runs a client-side "slot machine" animation over undrawn, non-hidden candidates for the active side, then writes the winner to `debate/config/selectedQuestion`, marks it `drawn: true`, and sets `debate/config/questionRevealed = true` to trigger the `#question-overlay` fullscreen reveal (same show/hide pattern as `#vote-result-overlay`).

### Performance patterns

- **Like updates**: patch-only via `data-id` attribute — no full re-render of comment list
- **Like debounce**: `likingInFlight` Set prevents double-tap race conditions
- **Firebase cleanup**: `beforeunload` calls `.off()` on all refs in both files

### Topics constant

Both files contain an identical `TOPICS` array (6 strings). If topics change, update both files.

## Firebase Rules Reference

Current rules structure (update in Firebase Console if adding new paths):

```json
{
  "rules": {
    "debate": {
      "config":     { ".read": true, ".write": true },
      "votes":      { ".read": true, "$deviceId": { ".write": true } },
      "comments":   { ".read": true, "$commentId": { ".write": "..." } },
      "likes":      { ".read": true, "$commentId": { "$deviceId": { ".write": true } } },
      "topicVotes": { ".read": true, "$deviceId": { ".write": true } },
      "questions":  { ".read": true, "$questionId": { ".write": "!data.exists() || (data.child('drawn').val() !== true && newData.child('drawn').val() === true) || (data.child('hidden').val() !== true && newData.child('hidden').val() === true)" } }
    }
  }
}
```
