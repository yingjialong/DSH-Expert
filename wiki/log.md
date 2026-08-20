# 问答与学习日志

> 按时间倒序记录每一次问答、学习、实测。这份日志同时充当**回归检测的替代品**：`/dsh-sync` 时对"锚点被本次上游变更命中"的历史问答做抽查复验（见 `dsh-sync` skill 步骤 6）。

## 条目格式

```
## YYYY-MM-DD · <一句话主题>
- **提问者**：human | agent
- **问题**：<原问题，去项目化>
- **结论**：<一句话>
- **依据锚点**：<文件路径列表>
- **上游基线**：<短SHA>
- **沉淀**：<新增/更新的 wiki 页 + status>
- **实测**：<命令与结果，或"无">
- **覆盖度变化**：<领域 Lx→Ly，或"无">
```

---

## 2026-08-20 · 知识库建立（bootstrap）

- **提问者**：human
- **问题**：建立一个能持续自我更新的 DSH 专家系统
- **结论**：完成 5 轮需求盘问，确立双重身份、四把标尺、八维坐标系、防腐三件套（认知状态 / 锚点失效检测 / 掌握等级封顶）
- **依据锚点**：`packages/README.md`、`docs/module-graph.md`、`python/README.md`、`package.json`
- **上游基线**：DSH `141eb6f`（2026-08-19T23:11+08:00）· Cordis `8cc9e33`
- **沉淀**：`errors.md` E001-E003（3 条命名/版本陷阱，均为 `fact`）
- **实测**：无（冷启动不做 L3）
- **覆盖度变化**：全域 L0 → 冷启动进行中

**建库时核实的关键数字**（均为 T1 源码直接读出）：

| 项 | 值 | 来源 |
| --- | --- | --- |
| 包组数 | **50**（非文档所述） | `ls packages/` |
| 包总数 | **226** | `ls -d packages/*/*/` |
| master 版本 | `0.1.0-rc.8` | `package.json` |
| node 要求 | `^22.19.0 \|\| >=24.0.0` | `package.json` engines |
| packageManager | `pnpm@11.7.0` | `package.json` |
| rc.7→rc.8 变更规模 | **1604 文件 / +54064 / −10533** | `git diff --shortstat` |
| docs 顶层主题 | 19 | `ls docs/*.md` |
| docs/subsystems | 20 篇 | `ls docs/subsystems/` |
| docs/cookbook | 9 篇 | `ls docs/cookbook/` |
| docs/postmortem | 4 篇事故复盘 | `ls docs/postmortem/` |
| apps | `cli`、`web` | `ls apps/` |
