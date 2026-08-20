---
name: ewt360-lessons-mini
description: Condensed hardcoded EWT360 lesson recipe for weak/low-context models. Enforces privacy agreement before login, exhaustive 点击加载更多 handling, tab-close verification by tab list, no shared progress file, correct overview 学 entry, seriousness-check monitoring, and live counter verification. Use when the full ewt360-lessons skill is too large.
---

# EWT360 Mini Recipe — weak-model edition

## NEVER BREAK THESE RULES

1. Website actions only through `agent_browser`. One state-changing browser call at a time.
2. Never read/write `ewt360-progress.md`; never use any progress file. Live page only.
3. New account = `sessionMode:"fresh"`, login again, rediscover task, forget old URL/counters/title/lessonId.
4. Before `登录`, always `check .privacy__agreement input[type='checkbox']`.
5. Before selecting a lesson, click `点击加载更多` repeatedly until exact text `没有更多必学任务了` is observed.
6. `tab close tN` may return “not found” after actually closing. After every close, `tab list`; the new list decides success. Never batch tab cleanup.
7. Enter only from overview `div[data-type='2']`. Never click player right-side lessons.
8. Never seek video. Never JS `.click()`. Seriousness buttons use real `click`.
9. Complete means live subject counter +1. Otherwise rewatch; never skip.
10. Batch cannot branch on results. It only runs sequentially; `--bail` only stops. Read batch step 1 and choose the next tool call yourself.

## 1 LOGIN

```json
{"args":["open","https://www.ewt360.com/"],"sessionMode":"fresh"}
```

`snapshot -i`. If visible popup `同意并继续` exists, click it.

Fill:

```json
{"semanticAction":{"action":"fill","locator":"placeholder","value":"请输入学校提供的账号/手机号/邮箱","text":"<ACCOUNT>"}}
```

```json
{"semanticAction":{"action":"fill","locator":"placeholder","value":"请输入密码","text":"<PASSWORD>"}}
```

Mandatory before login:

```json
{"args":["check",".privacy__agreement input[type='checkbox']"]}
```

Verify:

```json
{"args":["eval","--stdin"],"stdin":"(() => document.querySelector('.privacy__agreement input[type=checkbox]')?.checked === true)()"}
```

Only if true:

```json
{"args":["click","#login__password button[type='submit']"]}
```

```json
{"args":["wait","--fn","(() => document.body.innerText.includes('退出登录') || !!document.querySelector('a[href*=\"#/student/homework\"]'))()","--timeout","60000"]}
```

A timeout can be false: eval `document.body.innerText.includes('退出登录')`; true means continue.

## 2 DISCOVER TASK

```json
{"args":["open","https://teacher.ewt360.com/ewtbend/bend/index/index.html#/student/homework"]}
```

Wait for `查看详情`. Click BOTH fixed filters:

```json
{"args":["click","xpath=(//button[normalize-space(.)='全部'])[1]"]}
```

```json
{"args":["click","xpath=(//button[normalize-space(.)='全部'])[2]"]}
```

```json
{"args":["click","xpath=(//span[normalize-space(.)='查看详情'])[1]"]}
```

```json
{"args":["wait","--url","**/holiday/student-task-overview?homeworkId=*","--timeout","30000"]}
```

`get url` = runtime-only `OVERVIEW_URL`. Never save it.

## 3 READ BEFORE COUNTER

```json
{"args":["eval","--stdin"],"stdin":"(() => [...document.querySelectorAll('li')].map(e=>(e.innerText||'').replace(/\\s+/g,' ').trim()).filter(t=>/完成\\s*\\d+\\/\\d+/.test(t)).slice(0,10))()"}
```

Remember requested subject `BEFORE=X/Y` only in this run.

## 4 OPEN SUBJECT AND LOAD EVERYTHING

```json
{"args":["open","<OVERVIEW_URL>"]}
```

```json
{"args":["click","xpath=//li[contains(.,'<SUBJECT>') and contains(.,'完成')]"]}
```

Read gate:

```json
{"args":["eval","--stdin"],"stdin":"(() => { const a=[...document.querySelectorAll('*')].filter(e=>e.offsetParent!==null&&e.children.length===0).map(e=>(e.innerText||'').trim()); return {rows:document.querySelectorAll('li[class*=taskItem]').length,state:a.includes('点击加载更多')||a.includes('加载更多')?'MORE':a.includes('没有更多必学任务了')?'DONE':'MISSING'}; })()"}
```

- MORE: remember OLD rows; click:

```json
{"args":["click","xpath=//*[not(*) and (normalize-space(.)='点击加载更多' or normalize-space(.)='加载更多')]"]}
```

Then wait rows > OLD or text `没有更多必学任务了`; run gate again.
- DONE: continue.
- MISSING: wait/read once, then snapshot if still missing. DO NOT select.

Repeat until explicit DONE. This is mandatory every lesson.

Extract incomplete videos only:

```json
{"args":["eval","--stdin"],"stdin":"(() => [...document.querySelectorAll('li[class*=taskItem]')].filter(e=>e.offsetParent!==null&&e.querySelector('div[data-type=\"2\"]')&&!(e.innerText||'').includes('已学完')).map(e=>{const x=(e.innerText||'').replace(/\\s+/g,' ').trim(),m=x.match(/^\\S+\\s+(.+?)\\s+视频\\s+(\\d+)分钟/);return {title:m?.[1]||x,minutes:m?+m[2]:9999}}).sort((a,b)=>a.minutes-b.minutes))()"}
```

Pick first/shortest unless user named a lesson.

## 5 TAB CLEANUP + ENTER

Before lesson: `tab list`. Active `*` must be overview. For each non-active id:

```json
{"args":["tab","close","<ID>"]}
```

Immediately `tab list` even if close errored. If ID absent, success. Repeat until one overview tab.

```json
{"args":["scrollintoview","xpath=//li[contains(@class,'taskItem') and contains(.,'<EXACT_TITLE>')]//div[@data-type='2']"]}
```

```json
{"args":["click","xpath=//li[contains(@class,'taskItem') and contains(.,'<EXACT_TITLE>')]//div[@data-type='2']"]}
```

Wait player URL contains `/homework/play-videos`, `lessonId=`. Run `tab list`; active `*` must be player. Close every non-active id one at a time with close→list verification. Then `get url`; must still be player.

## 6 PLAY + MONITOR

```json
{"args":["wait","--fn","(() => {const v=document.querySelector('video');return !!(v&&v.readyState>0)})()","--timeout","30000"]}
```

```json
{"args":["eval","--stdin"],"stdin":"(() => {const v=document.querySelector('video');if(!v)return 'NOVID';v.playbackRate=2;v.play();return {id:location.hash.match(/lessonId=(\\d+)/)?.[1],t:Math.round(v.currentTime),d:Math.round(v.duration),rate:v.playbackRate,paused:v.paused}})()"}
```

TIMEOUT_MS = `Math.round(((d-t)/2+60)*1000)`.

```json
{"args":["batch"],"stdin":"[[\"wait\",\"--fn\",\"(() => {const d=document.querySelector('[class*=earnest_check]');if(d&&d.offsetParent!==null)return (d.innerText||'').includes('错过了本节课所有的互动检测')?'MISSED':'CHECK';const v=document.querySelector('video');if(!v)return 'GONE';if(v.currentTime>=v.duration-3)return 'END';return false})()\",\"--timeout\",\"<TIMEOUT_MS>\"],[\"click\",\"[class*=earnest_check] .btn-DOCWn\"]]"}
```

Read batch step 1 only:
- CHECK: batch clicked; eval fresh t/d; repeat.
- MISSED: batch dismissed; rewatch same lesson from overview.
- END: step-2 click error expected; verify.
- GONE: re-enter same lesson.
- timeout: eval paused/dialog; dialog→real click; paused/no dialog→`v.play()`; stuck→rate 1; repeat.

If batch-string validation fails, retry once, then single wait. CHECK's immediate next call must be:

```json
{"args":["click","[class*=earnest_check] .btn-DOCWn"]}
```

## 7 VERIFY

Open OVERVIEW_URL, wait counters, run Section 3 counter eval.

- requested subject AFTER = BEFORE+1: success; no file write; next lesson returns to Section 3/4.
- unchanged: rewatch same title from overview, max 2 retries.
- subject Y/Y: stop.
- next account: forget all runtime values; return to Section 1 fresh.
