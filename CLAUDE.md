# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

辯論計時器 — a two-page web app for a live debate event "生活之間：從住民到公民". No build step, no framework, no package manager. Pure HTML/CSS/JS deployed to GitHub Pages.

**Live URL**: https://afroisgood.github.io/debate-timer/

## Deploy

```bash
git add index.html vote.html assets/
git commit -m "..."
git push origin main   # GitHub Pages auto-deploys from main branch
```

Changes are live within ~1 minute of push. No build or CI step.

## Architecture

### Two pages

| File | Role |
|---|---|
| `index.html` | Admin / main display — shown on the presenter's screen. Controls timer, manages stages, shows live A/B/C results of the current question, speaker-vote status, comments, and the fullscreen result overlay. |
| `vote.html` | Participant page — opened on phones via QR code. Votes A/B/C on the current question, votes for a speaker during the break, posts comments and likes. |

### Visual theme

Both pages share the 「漫畫爆炸」(comic pop) theme: cream halftone paper background, 3–5px black borders, hard offset shadows, skewed labels. Colors are CSS custom properties in each file's `:root` (`--red`, `--blue`, `--yellow` accent, `--ink`). The header logo is `assets/title.webp` (event key visual, transparent background). Fonts load from Google Fonts: Noto Sans TC (text) and Bungee (digits).

`index.html` layout (matches the design mockup): header = empty left spacer / title logo / stage chip (`#progress-text`) + QR button; left panel = stage banner → timer → control buttons → stage dots → next-stage line, plus an admin-only `#admin-tools` row (stage dropdown, ±60, fullscreen) shown via `body.admin-mode` (toggled in `setControlsEnabled()`); right panel = live vote card (`#vote-card`, rendered by `renderVoteCard()`: current question's A/B/C bars, or speaker-vote status outside question stages) + comment panel. Option colors: A `--opt-a` red, B `--opt-b` blue, C `--opt-c` yellow.

In `index.html`, JS reads `--cream` / `--coral` / `--orange` / `--sky` / `--warn` / `--alert` / `--gray-blue` via `getComputedStyle` for timer ring and side-label colors, so those names are kept as aliases of the theme colors.

### Firebase Realtime Database (compat SDK 10.12.0)

All state is shared via Firebase. No backend code exists — everything is client-side JS talking directly to the DB.

**Data paths:**
```
debate/
  config/
    currentStage      String  — stage name synced from index.html (loadStage() and on admin login)
    activeQuestion    Number|null — 0–4 while a question's stages run, else null; vote.html votes only on this question
    votingOpen        Boolean — admin kill-switch for all voting
    voteRevealed      Boolean — controls fullscreen result overlay on index.html
    display           Object  — { stageIndex, stageTotalTime, isRunning, timeLeft, isOvertime, overtime, at } timer snapshot published by the admin index.html (`publishTimerState()`); non-admin index.html screens mirror it (`applyDisplayState()`). `at` is server ms (local + `.info/serverTimeOffset`); while running, timeLeft/overtime are the run-segment baseline at `at`
  qVotes/
    {questionIndex}/
      {deviceId}/    { nickname, choice: "A"|"B"|"C", updatedAt }
  speakerVotes/
    {deviceId}/      { nickname, speaker: 0–3, updatedAt }   — only during "觀眾投票／休息"
  comments/
    {pushId}/        { nickname, text, createdAt, deleted }  (old entries may still carry `side`; ignored)
  likes/
    {commentId}/
      {deviceId}     true
```
Legacy paths `votes`, `topicVotes`, `questions`, `config/topicVotingOpen|questionRevealed|selectedQuestion` are no longer read or written.

**Firebase rules** must explicitly grant `.read`/`.write` per sub-node. Reading a parent node (`debate/`) fails silently if the parent has no `.read` rule — always read individual sub-nodes.

### Stage system (index.html)

`STAGES` is built from `QUESTIONS`: each entry is `[name, durationSeconds, questionIndex?]`. Per question (15 min): `第N題｜選邊站・投票` 180s → `第N題｜與談人立場說明` 480s → `第N題｜觀眾發言 1` 120s → `第N題｜觀眾發言 2` 120s. Then `觀眾投票／休息` 600s (= `SPEAKER_VOTE_STAGE`), `頒獎時間` 300s, `與談人分享` 900s, `活動結束` 0.

`stageQuestion()` returns the entry's question index (or null). `loadStage()` stops the timer, resets state, calls `syncStageToFirebase()` (writes `currentStage` + `activeQuestion`), then re-renders the vote card / overlay. The "公布結果" overlay (`renderVoteOverlay()`) shows the current question's A/B/C results during question stages, otherwise the speaker ranking (sorted, winner highlighted).

### Admin gate (index.html)

All timer/nav controls are `disabled` on load. `setControlsEnabled(bool)` enables/disables: prev/next stage, toggle, reset, ±time buttons, stage-select dropdown, dot bar navigation, and toggles `body.admin-mode`. Password is hardcoded as `ADMIN_PASSWORD` (index.html only). Logging in also syncs the current stage and timer to Firebase so phones and other (non-admin) main screens follow this screen. Admin actions (stage change, start/pause, reset, ±time) call `publishTimerState()`.

Keyboard shortcuts (space/arrows/R/F11) are also blocked when not admin.

### vote.html voting

`vote.html` listens to `config/activeQuestion` and `config/currentStage` and picks a mode in `refreshVoteMode()`:
- `question`: shows the question + A/B/C buttons with full option text; listens to `qVotes/{q}` for own choice and counts; after voting shows the live ratio. Can change choice while the question is active.
- `speaker`: during `觀眾投票／休息`, shows the 4 speakers; listens only to own `speakerVotes/{deviceId}` (results stay hidden until announced on the big screen).
- `idle`: no active vote.
`votingOpen === false` disables all vote buttons.

### Performance patterns

- **Like updates**: patch-only via `data-id` attribute — no full re-render of comment list
- **Like debounce**: `likingInFlight` Set prevents double-tap race conditions
- **Firebase cleanup**: `beforeunload` calls `.off()` on all refs in both files

### Questions / speakers constants

Both files contain identical `QUESTIONS` (title + options with `short` label for the big screen and full `text` for phones), `OPTION_LETTERS`, `SPEAKER_VOTE_STAGE`, `SPEAKER_VOTE_PROMPT` and `SPEAKERS`. If any change, update both files.

## Firebase Rules Reference

Current rules structure (update in Firebase Console if adding new paths):

```json
{
  "rules": {
    "debate": {
      "config":       { ".read": true, ".write": true },
      "qVotes":       { ".read": true, "$q": { "$deviceId": { ".write": true } } },
      "speakerVotes": { ".read": true, "$deviceId": { ".write": true } },
      "comments":     { ".read": true, "$commentId": { ".write": "..." } },
      "likes":        { ".read": true, "$commentId": { "$deviceId": { ".write": true } } }
    }
  }
}
```
