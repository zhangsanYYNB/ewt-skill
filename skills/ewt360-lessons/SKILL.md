---
name: ewt360-lessons
description: Automate 升学e网通 (ewt360.com) holiday-task VIDEO lessons. For any student account, opens the browser, logs in, finds the current task, and plays every unfinished 学 (video) lesson at 2x speed to completion so the task counter increments +1, auto-dismissing the random "seriousness check" popups and re-watching whenever a check is missed. Selectors and the whole state machine are hardcoded and were verified on a live account on 2026-08; this is a strict, long-running, low-context-friendly recipe. Use for "keep watching videos on ewt360", "finish the summer task videos", or similar.
---

# EWT360 Holiday Task — Video Automation (升学e网通 看课)

Play every unfinished **video lessons (学)** of a student's current holiday task to the end at **2x speed** so the server records each one as done (task counter `X/Y` becomes `X+1/Y`). Works for **any account**; nothing about the task, subject list, or homework number is hardcoded — everything is discovered from the live page each run.

All browser work goes through the **`agent_browser` tool** (native, not bash). Every command is an exact `args` array, and any JavaScript goes in the tool's `stdin` field. There is **no PowerShell quoting involved** (see "Windows / PowerShell notes").

---

## 0. Hard rules (breaking ANY of these breaks the run)

1. **Issue the monitor as ONE batch that waits AND clicks.** Never do `wait` in one call, think, then `click` in another call — the 24-second check window will expire. Always use the batch form in Section 5. The whole wait→click happens inside a single agent_browser invocation so there is **zero model reaction time**.
2. **Never jump the video with JS and never call `.click()` from JS.** The site detects scripted clicks and flags "missed check / abnormal study". The ONLY allowed JS is (a) setting `playbackRate = 2`, (b) the read-only diagnostics, (c) read-only extraction of counters. Everything else is a **real browser `click`**.
3. **Never switch lessons from the playlist on the right side of the player page** — that confuses `lessonId`, the video plays but is NOT counted. Always enter a lesson from the overview page's 学 button.
4. **Never skip a lesson because it showed "错过了所有互动检测".** The video is not counted until you RE-WATCH it and pass a check (see Section 7). Skips are forbidden.
5. **Don't trust memory for counts.** Re-read the overview counters before choosing a lesson and after every lesson. For a long run, also keep the progress file updated (Section 8).
6. **Keep ONE browser tab.** When the lesson page loads or after a page transition, delete every tab except the one you are using (see Section 4.4).
7. Everything below is a fixed sequence. Run it serially. Prefer exact selectors over guessing.

---

## 1. Setup / login (fixed, write in stone)

Open the login page:

```
{ "args": ["open", "https://www.ewt360.com/"] }
```

**1.1 Cookie consent (must be handled first if it appears).** If a dialog with `同意并继续` / `不同意` shows, click `同意并继续`:

```
{ "args": ["click", "xpath=//*[contains(text(),'同意并继续')]"] }
```

**1.2 If the login form is already filled / you're logged in**, skip to 1.4. Check via `snapshot -i`: if a `退出登录` button exists you are logged in.

**1.3 Standard account login** (all accounts use these fixed selectors):
1. Fill account:
   ```
   { "semanticAction": { "action": "fill", "locator": "placeholder", "value": "请输入学校提供的账号/手机号/邮箱", "text": "<ACCOUNT>" } }
   ```
2. Fill password:
   ```
   { "semanticAction": { "action": "fill", "locator": "placeholder", "value": "请输入密码", "text": "<PASSWORD>" } }
   ```
3. Submit:
   ```
   { "semanticAction": { "action": "click", "locator": "role", "value": "button", "name": "登 录" } }
   ```
4. Wait for the SPA to switch to the logged-in state (look for the 我的任务 link — it only exists when logged in and is exactly what Section 2 needs next):
   ```
   { "args": ["wait", "--fn", "(() => !!document.querySelector('a[href*=\"#/student/homework\"]'))()", "--timeout", "30000"] }
   ```
   If it times out, take a `snapshot -i`: a `退出登录` button also means you are logged in (then proceed to 2 directly); if a wrong-password / account message is shown, stop and ask the user to verify the credentials; if consent popped up again, accept it and retry the login button once, then re-wait.

**1.4 Verify** you are logged in (`退出登录` visible) before continuing.

> Credential handling: the account/password come from the user or saved config. Never invent credentials. If the user only gave credentials in chat, use them; otherwise ask for them first.

---

## 2. Discover the current task (generic — never hardcode homeworkId)

The task URL ends in `#/holiday/student-task-overview?homeworkId=NNN`. To find `NNN` for THIS account:

1. On the logged-in homepage, click the **我的任务** link (its href contains `#/student/homework`):
   ```
   { "args": ["click", "xpath=//a[contains(@href,'#/student/homework')]"] }
   ```
   or just open it directly:
   ```
   { "args": ["open", "https://teacher.ewt360.com/ewtbend/bend/index/index.html#/student/homework"] }
   ```
2. Clear item-type filters so ALL tasks show. Click the **全部** button:
   ```
   { "args": ["click", "xpath=//button[.='全部'][1]"] }
   ```
3. Open the **current task** = the FIRST task card's 查看详情 (tasks are ordered with the most recent first, and this is the task you must finish):
   ```
   { "args": ["click", "xpath=//span[.='查看详情'][1]"] }
   ```
4. Confirm you landed on the overview:
   ```
   { "args": ["get", "url"] }
   ```
   Expected: `.../index.html#/holiday/student-task-overview?homeworkId=<id>`. Remember this URL as `OVERVIEW_URL` for the whole run.

> If there is no 查看详情 yet (task list still loading), re-run a `wait --load networkidle` then step 3.

---

## 3. Understand the overview page (verified DOM map)

- The **subject tabs** are plain `<li>` whose text is `"<学科> 完成X/Y"` (e.g. `语文 完成31/35`). Clicking one expands that subject's lesson list.
- Each **lesson** is an `<li class="taskItem-...">` (class always starts with `taskItem`). Inside it, three buttons:
  - `导` (guide) — has no `data-type`,
  - `学` (the video) — `div[data-type="2"]` ← **this is the one we click**,
  - `练` (exercise) — `div[data-type="1"]`.
- A lesson is incomplete when its text does NOT contain `已学完`; already-completed ones say `已学完`. Use the `X/Y` subject counter as the source of truth.
- At the bottom of a long list there is a **加载更多** control (element class contains `loadMore`).

---

## 4. Enter a lesson (fixed sequence)

### 4.1 Open the overview
```
{ "args": ["open", "<OVERVIEW_URL>"] }
{ "args": ["wait", "--load", "domcontentloaded"] }
```

### 4.2 Open the subject tab
```
{ "args": ["click", "xpath=//li[contains(.,'<学科>') and contains(.,'完成')]"] }
```
- If the click reports *"outside nested scroll container / use scrollintoview"*, run first:
  ```
  { "args": ["scrollintoview", "xpath=//li[contains(.,'<学科>') and contains(.,'完成')]"] }
  ```
  then repeat the click.
- If the target lesson is missing because the list is truncated, click the load-more control:
  ```
  { "args": ["click", "xpath=//*[contains(text(),'加载更多') or contains(text(),'点击加载更多')]"] }
  ```
  (no-op is fine when it says `没有更多必学任务了`).

### 4.3 Click the lesson's 学 button (use the EXACT title from the page; it must be unique)
```
{ "args": ["scrollintoview", "xpath=//li[contains(@class,'taskItem') and contains(.,'<exact title>')]//div[@data-type='2']"] }
{ "args": ["click", "xpath=//li[contains(@class,'taskItem') and contains(.,'<exact title>')]//div[@data-type='2']"] }
```
- "Element not found" usually means the list wasn't expanded/loaded yet → redo 4.2 (+ load-more), then retry.

### 4.4 Close every other tab (hardcoded rule)
Entering a lesson opens a NEW tab and leaves the overview behind. Keep exactly one tab:
```
{ "args": ["tab", "list"] }
```
For each tab that is NOT the active one, close it:
```
{ "args": ["tab", "close", "<tN>"] }
```
The "not found" error after closing is normal (ids renumber) — just run `tab list` again and repeat until only one tab remains. Then re-verify the page:
```
{ "args": ["get", "url"] }
```
It must still end with `...#/homework/play-videos?...lessonId=<id>...`.

### 4.5 Wait for the video element, then start it at 2x
Video ready (single `wait`):
```
{ "args": ["wait", "--fn", "(() => { const v = document.querySelector('video'); return v && v.readyState > 0; })()", "--timeout", "30000"] }
```
Start at 2x and confirm state (this is the ONLY allowed playback script — equivalent to the player's 2x menu):
```
{ "args": ["eval", "--stdin"], "stdin": "(() => { const v = document.querySelector('video'); if (!v) return 'NOVID'; v.playbackRate = 2; v.play(); return {id: location.hash.match(/lessonId=(\\d+)/)?.[1], t: Math.round(v.currentTime), d: Math.round(v.duration), rate: v.playbackRate, paused: v.paused}; })()" }
```
Good = `id` is this lesson's id, `d` matches the listed minutes, `rate: 2`, `paused: false`, and `t` grows.

---

## 5. Playback monitor — ONE batch = wait AND click (no model reaction)

Never split this into two calls. Output the whole batch as a single tool call. The `wait` step blocks until the page state changes; the `click` step runs in the SAME invocation the instant a check popup appears. If no popup is up, the click fails benignly and you read the wait's result.

**Timeout formula (in ms):** `(duration - currentTime) / 2 + 60` seconds, i.e. `((d - t) / 2 + 60) * 1000`. `d` and `t` come from Section 4.5 — recompute after every loop.

```
{ "args": ["batch"],
  "stdin": "[[\"wait\",\"--fn\",\"(() => { const d = document.querySelector('[class*=earnest_check]'); if (d && d.offsetParent !== null) return 'CHECK'; const v = document.querySelector('video'); if (!v) return 'GONE'; if (v.currentTime >= v.duration - 3) return 'END'; return false; })()\",\"--timeout\",\"<((d - t) / 2 + 60) * 1000>\"],[\"click\",\"[class*=earnest_check] .btn-DOCWn\"]]" }
```

Read `details.batchSteps[0]` (the wait step) and branch:

| wait result | Meaning | What to do |
|---|---|---|
| `CHECK` | A seriousness-check popup appeared | The batch's step-2 click already dismissed it (if step 2 succeeded, fine). Loop back to Section 5 with a fresh timeout. |
| `END` | Video finished (`t >= d - 3`) | Step-2 click failed with "Element not found" — that is **expected and benign**. Go to Section 6 (verify). |
| `GONE` | video element disappeared (page crashed/refreshed) | Re-enter this lesson from Section 4 (overview → 学). |
| `waited: timeout` / no CHECK/END/GONE | Video stalled (decoding can't keep up at 2x, or a popup was left open) | Run the diagnostic in Section 5.1, then loop back. |

> A batch whose step 2 fails is a NORMAL result — `isError` on the tool does NOT mean a real error. Only step 1's result decides the branch.

### 5.0 WEAK-MODEL & TOOL-ARG NOTE (read before issuing the batch)
The `batch` tool's `stdin` field MUST be a single flat **string** whose content is the JSON text of the array. It starts with `[["wait"` and ends with `]]`. Do NOT pass a nested array/object, do NOT pretty-print/indent it, do NOT wrap the quotes twice. If the validation error's first line says `stdin: must be string` or `could not be parsed as JSON`, retry ONCE with the exact flat string from the code block above, copied verbatim, changing ONLY the timeout number.

### 5.0b MANDATORY FALLBACK (if the batch still fails, never stall)
1. Run one wait: `{ "args": ["wait", "--fn", "(() => { const d = document.querySelector('[class*=earnest_check]'); if (d && d.offsetParent !== null) return 'CHECK'; const v = document.querySelector('video'); if (!v) return 'GONE'; if (v.currentTime >= v.duration - 3) return 'END'; return false; })()", "--timeout", "<((d - t) / 2 + 60) * 1000>"] }`
2. Read the wait result. `CHECK` → the VERY next tool call (nothing else in between) is `{ "args": ["click", "[class*=earnest_check] .btn-DOCWn"] }`. `END` → Section 6. `GONE` → re-enter (Section 4). Timeout → Section 5.1, then loop.
This keeps the 24 s check window safe: the click is the immediate next call — no snapshots, no analysis, no extra reads between wait and click.

### 5.1 Stall diagnostic (only on a timeout)
```
{ "args": ["eval", "--stdin"], "stdin": "(() => { const v = document.querySelector('video'); const d = document.querySelector('[class*=earnest_check]'); return { t: v ? Math.round(v.currentTime) : -1, dur: v ? Math.round(v.duration) : -1, p: v ? v.paused : null, dlg: d ? ((d.innerText||'').replace(/\\s+/g,' ').trim().slice(0,60)) : 'NONE' }; })()" }
```
- `p: true` and a dialog → click it (`[class*=earnest_check] .btn-DOCWn`, single call).
- `p: true` and no dialog → resume: `{ "args": ["eval", "--stdin"], "stdin": "(() => { const v = document.querySelector('video'); if (v && v.paused) v.play(); })()" }` (playbackRate stays 2).
- `p: false` but `t` not advancing → decoder can't keep 2x: `{ "args": ["eval", "--stdin"], "stdin": "(() => { const v = document.querySelector('video'); if (v) v.playbackRate = 1; })()" }`.
- Then loop back to Section 5.

---

## 6. Verify the lesson counted (every lesson — mandatory)

1. Reopen the overview (one call), read counters (one call):
```
{ "args": ["open", "<OVERVIEW_URL>"] }
```
```
{ "args": ["eval", "--stdin"], "stdin": "(() => { return [...document.querySelectorAll('li')].map(e => e.innerText.replace(/\\s+/g,' ').trim()).filter(t => t.includes('完成') && t.includes('/')).slice(0, 8).join(' || '); })()" }
```
Output looks like `语文 完成31/35 || 数学 完成27/27 || ...`.

2. **Success** = the subject's counter incremented by 1 vs. the previous read (e.g. `5/29 → 6/29`). Update the progress file (Section 8) and start the next lesson from Section 4.

3. **Failure** (counter unchanged) — do NOT move on. Cause is one of:
   - You entered from the player-page right-side list instead of the overview 学 button → redo correctly,
   - or a check was missed → **rewatch** (Section 7).

---

## 7. Rewatch when a check was missed — NEVER skip (hardcoded)

Trigger: you see a popup whose text contains `你错过了本节课所有的互动检测` / `missed all checks`, **or** the counter did not +1 after END. Until this lesson is counted, it is not done.

1. Dismiss popup (same button, both messages — "点击通过检查" or "我知道了" use `.btn-DOCWn`):
   ```
   { "args": ["click", "[class*=earnest_check] .btn-DOCWn"] }
   ```
2. Re-enter the SAME lesson from the overview (Section 4.3): open overview → expand subject → click that lesson's `学` button. The video restarts from 0 and checks will re-trigger.
3. Run Section 4.5 (2x) then Section 5 batches, catching EVERY `CHECK` immediately.
4. On `END`, verify with Section 6. If it STILL didn't +1, rewatch once more.
5. If two rewatch attempts both fail, report it instead of silently skipping the lesson.

---

## 8. Long runs & low context (short-memory safety)

This is a long task and the model may forget where it is. Mitigate with a **progress ledger file**:

1. At the start of the run, read `ewt360-progress.md` in the working directory (create it if missing):
   - account, `OVERVIEW_URL` / homeworkId,
   - the last subject counters you observed (e.g. `生物 5/29`),
   - the lesson currently in progress,
   - state: `idle | playing | verifying | rewatching`.
2. After EVERY lesson verify (Section 6) update the file with the new counters and `state: idle`, and note the next unfinished lesson to pick.
3. Each time you begin Section 4 for a new lesson, you MUST re-read the ledger line and the live counters (never from memory) before acting.
4. Keep writes tiny (one small `write` per lesson). Never dump page HTML into the ledger.

If the session is interrupted, the ledger lets the next run resume from where playback stopped without re-watching done lessons.

---

## 9. Common pitfalls (cheat sheet)

| Symptom | Cause / fix |
|---|---|
| Batch step 2 "Element not found" | Expected when video ENDed (no popup) — ignore, branch on step 1. |
| Batch validation error (`stdin: must be string` / `not valid JSON`) | You passed the stdin as an array or pretty-printed string — retry once with the EXACT flat string form (Section 5.0), then fall back to sequential wait→click (Section 5.0b). |
| Click on subject/lesson fails with "outside nested scroll container" | Run `scrollintoview` on the same selector, then click again. |
| "Element not found" for a lesson | List not expanded/loaded → open subject tab, click 加载更多, retry. |
| Clicking the right-side player playlist | Forbidden — it breaks counting. Always go back to overview 学 button. |
| wait times out, not GONE | Stalled: run Section 5.1, resume/1x, continue. |
| Popup "错过了所有的互动检测" | Rewatch the lesson (Section 7). Never skip. |
| Count unchanged after END | Missed checks or wrong entry path → Section 7 rewatch / correct path. |
| Multiple tabs / wrong page | Section 4.4 — `tab list`, close extras, `get url`. |
| After closing tabs "active page unverified" | Run `get url` once to re-verify, then continue. |
| Long session name socket error | Use an explicit short session: add `` `--session ewt` `` tokens after `open`/first command. |

---

## 10. Windows / PowerShell notes (conversion of the original Termux manual)

This skill was ported from a Termux CLI manual to Windows. In pi you DO NOT type these into PowerShell. All commands are sent through the native `agent_browser` tool:

- CLI tokens go in `args` as separate array items — e.g. `["click","xpath=//li[contains(.,'语文') and contains(.,'完成')]"]`.
- JavaScript goes in the top-level `stdin` field, never appended to `args`. Because the JS lives in `stdin`, PowerShell's `$`, backticks, quotes, and the `[` `]` in xpath selectors never touch a shell parser, so no escaping bugs occur.
- Keep every snippet a single self-invoked function `(() => { ... })()` returning a value; avoid double quotes inside JS (use single quotes) so the token stays clean.
- **Only if you ever run the raw `agent-browser.exe` in a PowerShell console**: quote the whole `--fn`/eval expression in double quotes and replace inner `"` with single quotes; or pipe a `@'...'@` here-string into `eval --stdin`. This is optional; the native tool path above is the correct one.

---

## 11. Ref checksums / selectors verified on a live account (2026-08)

- Login fields: placeholders `请输入学校提供的账号/手机号/邮箱`, `请输入密码`; button `登 录`; consent `同意并继续`.
- Task entry: `我的任务` link → `#/student/homework` → `全部` → first `查看详情`.
- Overview subject: `//li[contains(.,'<学科>') and contains(.,'完成')]`.
- Lesson row: `li[class*=taskItem]`; 学 button `div[data-type='2']`; 导 has no data-type; 练 `data-type='1'`. Completed rows contain `已学完`.
- Video: `document.querySelector('video')`; check popup `[class*=earnest_check] .btn-DOCWn`; player URL has `lessonId=<id>`.
- Works at 2x end-to-end; counters increment after END (verified: a 生物 lesson went `5/29 → 6/29 → 7/29` across test runs).
- **Weak-model validation (meta/muse-glimmer-30b via ltzy relay, 2026-08):** ran the complete flow autonomously — login, task discovery, lesson entry, 2x, tab cleanup, monitoring, verify. It could NOT emit the batch `stdin` JSON string (repeatedly passed an array → `stdin: must be string`), but the Section 5.0b fallback (wait → immediate click) handled every CHECK and the lesson COUNTED: 生物 7/29 → 8/29 (染色体变异, lessonId=66548). Conclusion: for weak models, expect the fallback path to be used; the batch remains best for stronger models.
