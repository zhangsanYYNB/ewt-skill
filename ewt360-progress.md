# EWT360 Lesson Automation — Progress Ledger
# Read this before every lesson (Section 8). Update after every verify (Section 6).

account: <your_account>   # 脱敏：勿将账号提交到公开仓库
overview_url: https://teacher.ewt360.com/ewtbend/bend/index/index.html#/holiday/student-task-overview?homeworkId=10493931
homework_id: 10493931
state: idle            # idle | playing | verifying | rewatching

# Last observed counters from the live overview page (source of truth = live page).
counters:
  语文: 31/35
  数学: 27/27
  英语: 20/20
  物理: 36/36
  化学: 24/27
  生物: 8/29

# Next lesson to pick when resuming (pick an incomplete one from the live page).
next_lesson: 生物 — any unfinished 学 video (currently 8/29, 21 left)
last_done: 生物 染色体变异 (17min, lessonId=66548) -> counter 7/29->8/29, verified OK
run_log: D:/ewt-skill/.tmp/muse_full4.jsonl (meta/muse-glimmer-30b completed this lesson via fallback monitor)