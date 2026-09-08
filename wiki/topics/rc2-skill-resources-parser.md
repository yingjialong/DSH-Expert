---
title: rc.2 Skill 资源指引与 parser 公开边界
description: resourceBase 不提供资源能力，filesystem frontmatter 与整包完整性分层。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
anchors:
  - packages/skill/skill/src/index.ts#SkillResourceBase
  - packages/skill/skill/src/index.ts#renderSkillContent
  - packages/skill/skill/src/index.ts#SkillProvider
  - packages/skill/skill/package.json
  - packages/skill/skill-filesystem/src/index.ts#FileSystemSkillProvider
  - packages/skill/skill-filesystem/package.json
  - packages/skill/skill-filesystem/README.md
  - packages/skill/skill-filesystem/tests/skill-filesystem.spec.ts
  - packages/skill/tool-skill/src/index.ts
  - packages/skill/tool-skill/package.json
  - packages/skill/tool-skill/tests/tool-skill.spec.ts
  - packages/core/tools/src/index.ts#ToolRuntime
commit: b150a551b8d465e31e418e1b2eaf5e79bbb7d28e
verified_at: 2026-09-08
updated: 2026-09-08
asked_by: agent
related:
  - "[[wiki/packages/skill]]"
  - "[[wiki/topics/rc2-agent-request-attempt-schema]]"
---

条件：TypeScript Host · 官方 SkillRegistry/tool-skill · preset-scoped provider 与工具 · Skill 资源和执行文件系统可不同 · 普通 ToolRuntime · 不调用模型 · 本地/远端路径分别解释 · 固定 rc.2 / `b150a551b8d465e31e418e1b2eaf5e79bbb7d28e`。

## 资源指引与能力

- `SkillResourceBase` 的 directory/path、url/url、opaque/description 三分支分别渲染 base directory、base URL 或资源说明；省略时显示 provider 管理资源。前两者指示模型按 base 解释相对引用，均要求按需加载。
- renderSkillContent 只生成 `<skill_resources>` 与原样 `<skill_instructions>`，不执行 fs/HTTP/部署/脚本。字符串定位不提供跨机器映射、权限、文件树或认证能力。
- 官方 skill 工具只按 scope/cwd/signal 获取正文、检查 invocation policy，返回 `{name,provider,resourceBase?,content}` 并使用该 renderer；不枚举或装载整包资源。
- 公开 SkillProvider 只有 name/list/get；Registry 的 registerProvider/register/list/snapshot/get 无资源读取/部署/脚本 API。locator 由 provider 私有解释，不是标准 bundle handle。
- 条件性推论：已正常挂载的 preset-scope 插件可独立通过 ToolRuntime 注册资源工具，同时保留官方 SkillRegistry/tool-skill。工具自己拥有资源寻址与执行语义、output 合同及取消；Skill 加载不自动使该工具可见或授权。模型工具日志由 AgentLoop 拥有，direct execute 不自动补写。

## filesystem parser

- 必须有首行/结束行 `---`，YAML 顶层是非 null、非 array object。必填 name/description 为非空 string，name 经 kebab-case 检查；body trim。
- 可选字段是 whenToUse、metadata、disable-model-invocation、user-invocable。wrong-type whenToUse/metadata 省略；metadata 仅保留显式 object，不收集未知顶层字段。
- license、compatibility、allowed-tools 与其他未知顶层字段不被解释/复制为官方字段，不产生工具权限或环境检查。例外是 legacy disableModelInvocation/modelInvocable/userInvocable：存在就拒绝整条 Skill。
- invocation 接受 boolean、数值/字符串 1/0、大小写不敏感 true/false/yes/no/on/off；非法值拒绝整条，缺省允许两个 surface。
- description parser/Registry 仅检查 string.length>0，不 trim、不设最大长度，quoted whitespace 不在此被当作空字符串。目录 consumer 另按 catalogDescriptionMaxLength（默认500、整数>=3）折叠空白/trim，并以 JS length/slice 截断加 `...`；不是 parser 长度上限或 UTF-8 byte 限制，不截正文。

## 正式导出与整包限制

精确 rc.2 三个 npm tarball 的 manifest/root `.d.ts`/JS exports 已在内存只读核对，均无 src/。skill root 公开 SkillRegistry、类型、renderSkillContent/name/policy helpers；skill-filesystem root 只有 Config/FileSystemSkillProvider/apply/inject/name；tool-skill root 为 Config/apply/inject/name。无独立 parseSkillFile/parseFrontmatter/parseSkillText 公共入口，manifest 的 ./src/* 无实际文件不能消费。FileSystemSkillProvider 的 list/get 是可复用读取路径。

filesystem 只发现单层 `<name>/SKILL.md` 或 `<name>.md`，不读取整包 manifest。DSH 无由上述 Skill 类型统一定义的文件树、每文件 hash、entrypoint、部署目标或完整性证明；scripts/references/assets 是可被正文引用的资源约定。外部不可变存储不自动变成 DSH 核验过的 bundle 完整性。

证据：[类型/renderer](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/skill/skill/src/index.ts#L171)、[正文工具](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/skill/tool-skill/src/index.ts#L125)、[parser](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/skill/skill-filesystem/src/index.ts#L793)、[字段 helpers](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/skill/skill-filesystem/src/index.ts#L981)、[目录截断](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/skill/tool-skill/src/index.ts#L389)、[资源提示测试](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/skill/tool-skill/tests/tool-skill.spec.ts#L825)、[格式说明](https://github.com/deepseek-ai/deepseek-harness/blob/b150a551b8d465e31e418e1b2eaf5e79bbb7d28e/packages/skill/skill-filesystem/README.md)。

远端固定 tag 已核对；测试只读断言，未运行 parser/部署/脚本或跨机器 E2E，不评价外部项目。
