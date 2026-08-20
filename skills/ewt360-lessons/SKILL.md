---
name: ewt360-lessons
description: Automate 升学e网通 (ewt360.com) holiday-task VIDEO lessons with a fixed low-model-friendly state machine. Handles mandatory privacy consent before login, exhaustively clicks 点击加载更多 before selecting lessons, robustly removes stale tabs despite tab-close false-negative errors, plays at 2x while passing seriousness checks, verifies live counters, and never uses a shared progress file. Use for watching or testing EWT360 holiday-task 学 videos, including sequential multi-account runs.
---

# EWT360 Holiday Task — Video Automation

Use only the native **`agent_browser`** tool for the website. The commands and selectors below are intentionally fixed for weak models.

## 0. Hard rules

1. **Never read or write `ewt360-progress.md` or any other progress ledger.** Do not use `read`, `write`, or shell commands for run state. The live page is the only source of truth.
2. At every account boundary, discard the previous account's `OVERVIEW_URL`, homeworkId, counters, lesson title, and lessonId. Log in again and rediscover all values. Never reuse one account's values for another.
3. **Before clicking 登录, check the privacy-agreement checkbox.** The current control is `.privacy__agreement input[type='checkbox']`. Do not confuse it with `下次自动登录`.
4. **Do not select a lesson until the subject list says `没有更多必学任务了`.** `点击加载更多` is mandatory, not optional. Repeat it until the terminal text is observed.
5. **Keep one tab.** `tab close <id>` can close a tab successfully but still return `Tab <id> not found`. Therefore the close command's success/error is not authoritative; the following `tab list` is authoritative. Never put tab cleanup in a fail-fast batch.
6. Enter every lesson from the overview row's `学` button, `div[data-type='2']`. Never switch through the player page's right-side playlist; that can break counting.
7. Never seek/jump `currentTime`, never invoke DOM `.click()` from JavaScript, and never script a seriousness-check click. Real browser `click`/`check` commands are mandatory. JavaScript is allowed only for read-only state extraction and for setting the normal player rate/play state.
8. A lesson is complete only when the live subject counter increments by exactly 1. If it does not, rewatch the same lesson; never silently skip it.
9. Run state-changing steps serially. Do not use `multi_tool_use.parallel` for browser actions.
10. A raw `batch` is a fixed sequential command list. **It cannot inspect an earlier result and conditionally choose a different later command.** `--bail` can only stop after a failure. The model must read the returned `details.batchSteps` and branch in its next tool call. Section 6 uses a safe unconditional click after its wait.

---

## 1. Account boundary and login

### 1.1 Start the account

For the first browser call of each new account, use a fresh managed browser session:

```json
{ "args": ["open", "https://www.ewt360.com/"], "sessionMode": "fresh" }
```

Then inspect once:

```json
{ "args": ["snapshot", "-i"] }
```

If a separate popup named `同意并继续` is visible, click that visible ref first. This popup is different from the mandatory form checkbox below.

If the login form is absent but `退出登录` is visible, do not assume the existing session is the requested account. Click the visible `退出登录` control, reopen `https://www.ewt360.com/`, and wait for the login form.

### 1.2 Fill credentials

```json
{ "semanticAction": { "action": "fill", "locator": "placeholder", "value": "请输入学校提供的账号/手机号/邮箱", "text": "<ACCOUNT>" } }
```

```json
{ "semanticAction": { "action": "fill", "locator": "placeholder", "value": "请输入密码", "text": "<PASSWORD>" } }
```

### 1.3 Mandatory agreement — always before 登录

Use native `check`, which is idempotent and dispatches a real browser input action:

```json
{ "args": ["check", ".privacy__agreement input[type='checkbox']"] }
```

Read-only verification:

```json
{ "args": ["eval", "--stdin"], "stdin": "(() => ({ agreed: document.querySelector('.privacy__agreement input[type=checkbox]')?.checked === true, autoLogin: document.querySelector('#login__password_autoLogin')?.checked === true }))()" }
```

Proceed only when `agreed: true`. `autoLogin` is unrelated and may be either value.

Now click login:

```json
{ "args": ["click", "#login__password button[type='submit']"] }
```

Wait up to 60 seconds for the logged-in state:

```json
{ "args": ["wait", "--fn", "(() => document.body.innerText.includes('退出登录') || !!document.querySelector('a[href*=\"#/student/homework\"]'))()", "--timeout", "60000"] }
```

If this wait times out, run one read-only diagnostic. If it reports `logged: true`, continue; otherwise report the visible error rather than repeatedly submitting credentials:

```json
{ "args": ["eval", "--stdin"], "stdin": "(() => ({ logged: document.body.innerText.includes('退出登录'), loginVisible: !!document.querySelector('#login__password_userName')?.offsetParent, messages: [...document.querySelectorAll('[role=alert],.ant-message-notice,.ant-form-item-explain-error,.ant-modal')].filter(e => e.offsetParent !== null).map(e => (e.innerText||'').replace(/\\s+/g,' ').trim()).filter(Boolean) }))()" }
```

Never put credentials into a repository file, progress file, screenshot name, session name, or final report.

---

## 2. Discover this account's current task

Open the fixed task-list route directly:

```json
{ "args": ["open", "https://teacher.ewt360.com/ewtbend/bend/index/index.html#/student/homework"] }
```

Wait for task cards:

```json
{ "args": ["wait", "--fn", "(() => document.body.innerText.includes('查看详情'))()", "--timeout", "30000"] }
```

The page currently has two `全部` buttons: subject filter first, type filter second. Click both so an old filter cannot hide the current task:

```json
{ "args": ["click", "xpath=(//button[normalize-space(.)='全部'])[1]"] }
```

```json
{ "args": ["click", "xpath=(//button[normalize-space(.)='全部'])[2]"] }
```

Open the first/current task card:

```json
{ "args": ["click", "xpath=(//span[normalize-space(.)='查看详情'])[1]"] }
```

Wait for the overview URL:

```json
{ "args": ["wait", "--url", "**/holiday/student-task-overview?homeworkId=*", "--timeout", "30000"] }
```

```json
{ "args": ["get", "url"] }
```

The returned URL is this account's runtime-only `OVERVIEW_URL`. Never save it to disk and never carry it into the next account.

---

## 3. Read live counters

Wait for subject counters:

```json
{ "args": ["wait", "--fn", "(() => [...document.querySelectorAll('li')].some(e => /完成\\s*\\d+\\/\\d+/.test(e.innerText||'')))()", "--timeout", "30000"] }
```

Read them:

```json
{ "args": ["eval", "--stdin"], "stdin": "(() => [...document.querySelectorAll('li')].map(e => (e.innerText||'').replace(/\\s+/g,' ').trim()).filter(t => /完成\\s*\\d+\\/\\d+/.test(t)).slice(0,10))()" }
```

Immediately before entering a lesson, remember only in the current conversation:

- requested account (masked if reported),
- `OVERVIEW_URL`,
- requested subject,
- subject counter `BEFORE = X/Y`,
- chosen exact title.

Do not persist these values to a file.

---

## 4. Exhaustively load the requested subject — mandatory

### 4.1 Open overview and subject

```json
{ "args": ["open", "<OVERVIEW_URL>"] }
```

```json
{ "args": ["wait", "--load", "domcontentloaded"] }
```

```json
{ "args": ["click", "xpath=//li[contains(.,'<SUBJECT>') and contains(.,'完成')]"] }
```

If that click reports a nested-scroll error, run `scrollintoview` on exactly the same XPath and click again.

Wait until rows or a terminal load marker exists:

```json
{ "args": ["wait", "--fn", "(() => document.querySelectorAll('li[class*=taskItem]').length > 0 || document.body.innerText.includes('没有更多必学任务了'))()", "--timeout", "30000"] }
```

### 4.2 Mandatory load-more loop

Read the list state:

```json
{ "args": ["eval", "--stdin"], "stdin": "(() => { const visible = e => e.offsetParent !== null; const leaves = [...document.querySelectorAll('*')].filter(e => visible(e) && e.children.length === 0).map(e => (e.innerText||'').trim()); const rows = [...document.querySelectorAll('li[class*=taskItem]')].filter(visible); return { rowCount: rows.length, loadState: leaves.includes('点击加载更多') || leaves.includes('加载更多') ? 'MORE' : leaves.includes('没有更多必学任务了') ? 'DONE' : 'MISSING' }; })()" }
```

Branch exactly:

- `MORE`: note `OLD_ROW_COUNT`, click the exact leaf control below, wait for either a larger row count or terminal text, then run the state read again.
- `DONE`: only now continue to Section 4.3.
- `MISSING`: wait once for the known control state, read again, and if still missing take one snapshot and stop selection. **Never interpret MISSING as DONE.**

Exact load-more click:

```json
{ "args": ["click", "xpath=//*[not(*) and (normalize-space(.)='点击加载更多' or normalize-space(.)='加载更多')]"] }
```

After each click:

```json
{ "args": ["wait", "--fn", "(() => document.querySelectorAll('li[class*=taskItem]').length > <OLD_ROW_COUNT> || document.body.innerText.includes('没有更多必学任务了'))()", "--timeout", "15000"] }
```

Repeat the state-read → click → wait cycle until the returned value is explicitly `loadState: 'DONE'`. On the verified page, a long 语文 list changed from 30 rows to 35 rows after this required click.

### 4.3 Extract only incomplete video lessons

Run this only after `DONE`:

```json
{ "args": ["eval", "--stdin"], "stdin": "(() => [...document.querySelectorAll('li[class*=taskItem]')].filter(e => e.offsetParent !== null && e.querySelector('div[data-type=\"2\"]') && !(e.innerText||'').includes('已学完')).map(e => { const text=(e.innerText||'').replace(/\\s+/g,' ').trim(); const m=text.match(/^\\S+\\s+(.+?)\\s+视频\\s+(\\d+)分钟/); return { title:m?.[1]||text, minutes:m?Number(m[2]):9999, text }; }).sort((a,b)=>a.minutes-b.minutes || a.title.localeCompare(b.title)))()" }
```

Choose the first/shortest item unless the user named a lesson. Rows without `div[data-type='2']` are tests, not videos, and must not be selected.

---

## 5. Enter one lesson and enforce one-tab state

### 5.1 Pre-click cleanup: preserve the active overview

Run:

```json
{ "args": ["tab", "list"] }
```

The active line starts with `*`. Confirm that active URL is `OVERVIEW_URL`. If extra tabs exist, close **one non-active id at a time**:

```json
{ "args": ["tab", "close", "<NON_ACTIVE_TAB_ID>"] }
```

Regardless of whether that call says success or `Tab ... not found`, immediately run:

```json
{ "args": ["tab", "list"] }
```

Judge only the new list:

- closed id absent → close succeeded; do not retry that id,
- closed id still present → confirm overview is active, then retry once,
- repeat until exactly one active overview tab remains.

**Do not use `batch --bail` for tab cleanup.** A verified run showed `tab close t1` and `tab close t2` each returned “not found” even though the following list proved each tab had been removed.

### 5.2 Click 学 from the overview

```json
{ "args": ["scrollintoview", "xpath=//li[contains(@class,'taskItem') and contains(.,'<EXACT_TITLE>')]//div[@data-type='2']"] }
```

```json
{ "args": ["click", "xpath=//li[contains(@class,'taskItem') and contains(.,'<EXACT_TITLE>')]//div[@data-type='2']"] }
```

Wait for the newly active player tab:

```json
{ "args": ["wait", "--fn", "(() => location.hash.includes('/homework/play-videos') && /lessonId=\\d+/.test(location.hash))()", "--timeout", "30000"] }
```

### 5.3 Post-click cleanup: preserve the active player

Run `tab list`. Confirm the `*` active URL contains both `/homework/play-videos` and `lessonId=`. Close one non-active tab, then list again, using the exact loop from Section 5.1 until only the player remains.

Finally:

```json
{ "args": ["get", "url"] }
```

If the final URL is not the player URL, recover with `tab list`, select the player via `tab <PLAYER_TAB_ID>`, and verify again. Do not start playback on an unverified tab.

---

## 6. Start and monitor playback

### 6.1 Start at 2x

```json
{ "args": ["wait", "--fn", "(() => { const v=document.querySelector('video'); return !!(v && v.readyState > 0); })()", "--timeout", "30000"] }
```

```json
{ "args": ["eval", "--stdin"], "stdin": "(() => { const v=document.querySelector('video'); if(!v) return 'NOVID'; v.playbackRate=2; v.play(); return { id:location.hash.match(/lessonId=(\\d+)/)?.[1], t:Math.round(v.currentTime), d:Math.round(v.duration), rate:v.playbackRate, paused:v.paused }; })()" }
```

Good state: correct lesson id, finite duration, `rate: 2`, `paused: false`.

### 6.2 Monitor with one static batch

Compute `TIMEOUT_MS = Math.round(((d - t) / 2 + 60) * 1000)`.

Issue this exact raw batch. Top-level `stdin` must be a **string**, beginning with `[["wait"` and ending with `]]`:

```json
{ "args": ["batch"], "stdin": "[[\"wait\",\"--fn\",\"(() => { const d=document.querySelector('[class*=earnest_check]'); if(d && d.offsetParent!==null){ const s=(d.innerText||'').replace(/\\s+/g,' '); return s.includes('错过了本节课所有的互动检测') ? 'MISSED' : 'CHECK'; } const v=document.querySelector('video'); if(!v) return 'GONE'; if(v.currentTime>=v.duration-3) return 'END'; return false; })()\",\"--timeout\",\"<TIMEOUT_MS>\"],[\"click\",\"[class*=earnest_check] .btn-DOCWn\"]]" }
```

This is not conditional branching. It works because the second command is always safe to attempt:

- after `CHECK`/`MISSED`, it dismisses the visible dialog,
- after `END`, there is no dialog and the click fails benignly.

Read **only `details.batchSteps[0]`** to choose the next state:

| Wait result | Next action |
|---|---|
| `CHECK` | The batch already clicked it. Re-read `{t,d}` and repeat Section 6.2. |
| `MISSED` | The batch dismissed the missed-check notice. Rewatch the same lesson from Section 4. |
| `END` | Ignore the expected step-2 click error. Go to Section 7. |
| `GONE` | Re-enter the same lesson from Section 4. |
| timeout | Run Section 6.3, then monitor again. |

If the model cannot encode batch `stdin` as a string, retry it once only. Then use the mandatory fallback: run the same wait alone; if it returns `CHECK`, the immediate next tool call must be the real click below, with no snapshot or analysis in between:

```json
{ "args": ["click", "[class*=earnest_check] .btn-DOCWn"] }
```

### 6.3 Timeout diagnostic

```json
{ "args": ["eval", "--stdin"], "stdin": "(() => { const v=document.querySelector('video'); const d=document.querySelector('[class*=earnest_check]'); return { t:v?Math.round(v.currentTime):-1, dur:v?Math.round(v.duration):-1, paused:v?.paused ?? null, rate:v?.playbackRate ?? null, dialog:d&&d.offsetParent!==null?(d.innerText||'').replace(/\\s+/g,' ').trim().slice(0,100):'NONE' }; })()" }
```

- paused + dialog: real-click `[class*=earnest_check] .btn-DOCWn` immediately,
- paused + no dialog: use the allowed player-state eval `v.play()`,
- playing but time does not advance: set `playbackRate=1`, resume, and monitor,
- video missing: re-enter from overview.

---

## 7. Verify and continue

Open `OVERVIEW_URL`, wait for counters, and run the counter extraction from Section 3.

For the requested subject:

- `AFTER = BEFORE + 1`: success. Do not write a file. Start the next lesson by re-reading the live counter and rerunning the complete Section 4 load-more gate.
- counter unchanged: rewatch the same exact lesson. Always enter again from the overview `学` button. Allow at most two rewatch attempts, then report the blocker.
- counter changed unexpectedly or account/task identity is uncertain: stop; rediscover this account's task rather than guessing.

When the subject reaches `Y/Y`, stop that subject. For another account, start again at Section 1 and discard every runtime value from this account.

---

## 8. Fixed recovery table

| Symptom | Required response |
|---|---|
| `tab close tN` says not found | Immediately `tab list`; if tN is absent, it succeeded. Never retry an absent id. |
| Batch tab cleanup stops on first close | Do not batch closes. List → close one → list. |
| Lesson list shows `点击加载更多` | Click it and wait for more rows/terminal text. Selection is forbidden. |
| Load state is `MISSING` | Wait/read again; snapshot once if still missing. Never treat as complete. |
| Lesson not found | Reopen subject, finish the load-more loop to `DONE`, extract again. |
| Player URL wrong | `tab list`, select the active player id, `get url`; never use right-side playlist. |
| Monitor batch step 2 fails after `END` | Expected; branch only on batch step 1. |
| Counter unchanged | Rewatch same lesson from overview; never skip. |
| New account starts | Fresh login, fresh task discovery, no shared file or previous values. |

---

## 9. Live selector checks (verified 2026-08)

- Login account: `#login__password_userName` / placeholder `请输入学校提供的账号/手机号/邮箱`
- Login password: `#login__password_password` / placeholder `请输入密码`
- Mandatory agreement: `.privacy__agreement input[type='checkbox']`
- Unrelated auto-login: `#login__password_autoLogin`
- Submit: `#login__password button[type='submit']`
- Task filters: first and second `button` whose exact text is `全部`
- Task detail: first exact `查看详情`
- Subject: `//li[contains(.,'<SUBJECT>') and contains(.,'完成')]`
- Lesson row: `li[class*=taskItem]`
- 学 button: `div[data-type='2']`
- Load more: exact leaf text `点击加载更多` or `加载更多`
- Load terminal: exact text `没有更多必学任务了`
- Player: `video`; URL contains `/homework/play-videos` and `lessonId=`
- Seriousness dialog button: `[class*=earnest_check] .btn-DOCWn`

Small-flow verification on a live 语文 task confirmed: mandatory agreement selector checked before login; task discovery reached homeworkId `10493931`; 语文 showed 30 rows plus `点击加载更多`, then 35 rows plus `没有更多必学任务了`; lesson `T文言文实词课内现汉推断法` opened the correct player with lessonId `116414`; and tab-close false-negative responses still removed the stale tabs as proved by subsequent `tab list` calls.
