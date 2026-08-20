# ewt-skill · 升学e网通 看课自动化

把「升学e网通（ewt360.com）」假期任务的**视频课（学）**逐课自动看完的 pi skill 与运行配方。
任何学生账号通用：登录 → 动态发现当前任务（homeworkId 不写死）→ 从概览页点「学」进入课程 → 2 倍速播放到底 → 自动通过随机"认真度检测"弹窗 → 验证计数 +1。

> 原始手工方案见 `ewt360-看课自动化操作手册.md`；可执行的 pi skill 与流程提示词见 `skills/` 与 `docs/`。

---

## 目录

```
ewt-skill/
├── README.md                    本说明
├── docs/
│   └── 运行流程提示词.md          ★ 完整流程提示词（系统提示词 + 任务提示词 + 一条命令跑法）
├── skills/
│   ├── ewt360-lessons/          标准 skill（全流程、含弱模型兜底说明）
│   │   └── SKILL.md
│   └── ewt360-lessons-mini/     精简 skill（弱模型 / 8-32K 小上下文专用）
│       └── SKILL.md
└── ewt360-看课自动化操作手册.md 原 Termux 操作手册（历史参考）
```

## 快速开始

1. **安装 skill**：把 `skills/ewt360-lessons/`（或 `ewt360-lessons-mini/`）复制到 pi 的 skills 目录
   （Windows: `C:\Users\<你>\.pi\agent\skills\`），或运行时用 `--skill <路径>` 加载。
2. **准备模型**：在 `~/.pi/agent/models.json` 注册 OpenAI 兼容供应商（见下方示例），
   或运行时用 `--provider/--model/--api-key` 传入。
3. **跑流程**：加载 skill 后直接给账号、密码和目标学科。新版流程不读写共享进度文件，适合顺序执行多个账户。

## 2026-08 修复要点

- 登录前强制勾选 `.privacy__agreement input[type='checkbox']`，与“下次自动登录”区分。
- 选课前强制重复点击“点击加载更多”，直到页面明确显示“没有更多必学任务了”。
- 标签页清理由 `tab list → close 一个 → tab list 复核` 驱动；`tab close` 即使返回 not found，也以下一次列表为准。
- 不再使用 `ewt360-progress.md`，避免多账户串号；进度和计数只读取当前账户的实时概览页。
- `batch` 不能按上一步返回值条件分支；它只能顺序执行（`--bail` 只能失败停止）。监控 batch 使用安全的固定 wait+click，下一状态由模型读取 `details.batchSteps[0]` 后决定。

## 模型配置示例（models.json）

```json
{
  "providers": {
    "ltzy-muse": {
      "api": "openai-completions",
      "baseUrl": "https://api.ltzy.top/v1",
      "apiKey": "<你的_API_KEY>",
      "compat": { "supportsDeveloperRole": false },
      "models": [
        { "id": "meta/muse-glimmer-30b", "name": "Muse Glimmer 30B", "contextWindow": 32768 }
      ]
    }
  }
}
```

## 一条命令（非交互，Windows PowerShell）

```powershell
pi -p --mode json `
  --provider ltzy-muse --model "meta/muse-glimmer-30b" --api-key "<API_KEY>" `
  --skill "skills/ewt360-lessons-mini/SKILL.md" `
  --tools "agent_browser" `
  "账号=<ACCOUNT>，密码=<PASSWORD>，学科=<SUBJECT>。按 mini skill 先测试一节，验证计数 +1；不要读写进度文件。"
```

## 实测记录（2026-08，账号已脱敏）

| 日期 | 内容 | 结果 |
|---|---|---|
| 08-18 | 手工链路验证（本人身环境） | 语文/数学/英语/物理/化学/生物 计数读取、登录、进课、2倍速、弹窗、验证 全通 |
| 08-18 | 生物 DNA是主要的遗传物质（20min） | 5/29 → 6/29 ✅ |
| 08-18 | 生物 DNA的结构及复制（17min，batch wait+click 链路） | 6/29 → 7/29 ✅ |
| 08-18 | **meta/muse-glimmer-30b 全自动跑通**（染色体变异 17min，兜底监控链路） | 7/29 → 8/29 ✅ |

## 免责声明

本项目仅用于学习自动化与 AI Agent 技能开发。请遵守 ewt360 平台规则与学校安排，仅在自己被授权完成的作业/任务上使用。