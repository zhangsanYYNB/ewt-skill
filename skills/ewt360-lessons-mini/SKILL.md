---
name: ewt360-lessons-mini
description: Condensed version of ewt360-lessons for weak/low-context models. Same rules, only the exact canned commands. Use when the main skill is too much context (small models, 8-32K windows).
---

# EWT360 Mini Recipe (weak-model edition)

STRICT RULES
1. Browser actions ONLY via agent_browser tool. Never bash/read for page state.
2. REAL clicks only. Never jump video via JS.
3. Always enter lessons from the overview 学 button (div[data-type='2']). NEVER use the player's right-side list.
4. Never skip. If a lesson did not count, rewatch it.
5. Keep every reply to ONE short line. No analysis text. No snapshots unless a step fails.

## 1. LOGIN (fixed)
open https://www.ewt360.com/
If dialog: click 同意并继续  ->  args ["click","xpath=//*[contains(text(),'同意并继续')]"]
If login form present (placeholder 请输入学校提供的账号/手机号/邮箱, 请输入密码):
  fill placeholder 请输入学校提供的账号/手机号/邮箱 = ACCOUNT
  fill placeholder 请输入密码 = PASSWORD
  click button 登 录
Wait max 30s for a[href*="#/student/homework"] OR leaf text 退出登录 (snapshot to check).

## 2. DISCOVER TASK
click xpath=//a[contains(@href,'#/student/homework')]
click xpath=//button[.='全部'][1]
click xpath=//span[.='查看详情'][1]
get url  -> must contain homeworkId. This URL is OVERVIEW_URL. Never hardcode it.

## 3. ENTER ONE LESSON
open OVERVIEW_URL, wait --load domcontentloaded
click xpath=//li[contains(.,'生物') and contains(.,'完成')]  (if "outside nested scroll container": scrollintoview same xpath, then click)
Find the shortest INCOMPLETE 学 video in the list (row has 学 with div[data-type='2']; skip rows containing 已学完; read the N分钟 field).
scrollintoview xpath=//li[contains(@class,'taskItem') and contains(.,'<EXACT TITLE>')]//div[@data-type='2']
click xpath=//li[contains(@class,'taskItem') and contains(.,'<EXACT TITLE>')]//div[@data-type='2']
tab list ; close every tab except the active one (tab close tN ; repeat until 1 tab)
get url  -> must contain lessonId=NNN

## 4. START PLAYBACK
wait --fn "(() => { const v = document.querySelector('video'); return v && v.readyState > 0; })()" --timeout 30000
eval --stdin: (() => { const v = document.querySelector('video'); if (!v) return 'NOVID'; v.playbackRate = 2; v.play(); return {id: location.hash.match(/lessonId=(\\d+)/)?.[1], t: Math.round(v.currentTime), d: Math.round(v.duration)}; })()
Note d and t.

## 5. MONITOR UNTIL END (ONE batch = wait + click)
Repeat this batch (only replace TIMEOUT_MS = Math.round(((d - t)/2 + 60) * 1000); recompute after each loop):
{
  "args": ["batch"],
  "stdin": "[[\"wait\",\"--fn\",\"(() => { const d = document.querySelector('[class*=earnest_check]'); if (d && d.offsetParent !== null) return 'CHECK'; const v = document.querySelector('video'); if (!v) return 'GONE'; if (v.currentTime >= v.duration - 3) return 'END'; return false; })()\",\"--timeout\",\"TIMEOUT_MS\"],[\"click\",\"[class*=earnest_check] .btn-DOCWn\"]]"
}
IMPORTANT: stdin must be a STRING. Copy it exactly: starts with [["wait" and ends ]] ; only the number changes. If validation says "stdin must be string" or "not valid JSON", do FALLBACK: 
  (a) wait --fn "<same monitor fn>" --timeout TIMEOUT_MS
  (b) result CHECK -> IMMEDIATE next call: click [class*=earnest_check] .btn-DOCWn ; then loop to (a)
  (c) result END -> go to step 6 ; GONE -> step 3 ; timeout -> eval v.paused/v.play/v.playbackRate=1 then loop to (a)
Click errors from step 2 of the batch are EXPECTED when no popup: ignore them.

## 6. VERIFY (MANDATORY, never skip)
open OVERVIEW_URL
eval --stdin: (() => { return [...document.querySelectorAll('li')].map(e => e.innerText.replace(/\\s+/g,' ').trim()).filter(t => t.includes('完成') && t.includes('/')).slice(0, 8).join(' || '); })()
If 生物 X/Y incremented by 1 vs before -> SUCCESS.
If not -> rewatch: click the same lesson's 学 button again from the overview (step 3), replay, pass checks, verify again. Max 2 rewatches.

## 7. PROGRESS FILE (keep it current)
write D:/ewt-skill/ewt360-progress.md with: counters as read, state: idle.