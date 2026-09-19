---
title: 1.5-rc.2 Catalog 严格恢复与 V0 物理 header
description: 区分 header 分类、完整迁移、当前格式重读及未结束日志与物理截断。
type: reference
status: verified_inference
mastery: L2
freshness: fresh
commit: fb2c4b9e698e30edb738bca4cf0618587db7d203
verified_at: 2026-09-19
updated: 2026-09-19
asked_by: agent
anchors:
  - packages/session/session-format-catalog/src/index.ts
  - packages/session/session-format-catalog/src/generated.ts
  - packages/session/session-format-catalog/src/current.ts
  - packages/session/session-format/src/catalog.ts
  - packages/session/session-format/src/types.ts
  - packages/session/session-format-v0-to-v1/src/codec.ts
  - packages/session/session-format-v0-to-v1/src/migration.ts
  - packages/session/session-format-v0-to-v1/src/relationships.ts
  - packages/session/session-format-v0-to-v1/tests/migration.spec.ts
  - packages/session/session-format-v2-to-v3/src/validation.ts
  - packages/session/session-format-v2-to-v3/tests/migration.spec.ts
  - packages/session/session-format-v2-to-v3/tests/combined-migration.spec.ts
related:
  - "[[wiki/topics/rc15-public-embedding-migration]]"
---

S=0.1.1-rc.2 / b150a551b8d465e31e418e1b2eaf5e79bbb7d28e；T=0.1.5-rc.2 / fb2c4b9e698e30edb738bca4cf0618587db7d203。远端tag一致，目标6个正式包sha512/exports/声明/根JS核验。未执行catalog或测试，未启动Agent/模型、未读取外部实际输入。先完整回传成功后沉淀，L2 / verified_inference。

条件（八维坐标）：TS调用者 · 离线副本 · 每artifact独立restore · 物理字节owner另管 · 纯公开catalog · 无凭据调用 · 平台无关静态核验 · 固定S→T。

## 公共调用顺序

catalog root实际仅导出sessionFormatCatalog与SessionFormatUnsupportedMigrationError。restoreCurrent是内部构造回调，不是root API。公开实例有currentVersion/readHeader/createRestore/encodeCurrentHeader/encodeCurrentEvent；restore有header/decodeRow/finish。

严格序列：readHeader(originalPhysicalHeader)分类 → createRestore(同一原physicalHeader,{recovery:strict,validation:current}) → 全部decodeRow → finish → encodeCurrentHeader(artifact.header,artifact.inheritedEventCount)及逐event编码 → 可再以native strict/current完整重读输出。

不能把readHeader返回的已转换逻辑header当原物理header继续传入。header分类、reader创建、前N行decode及单条encode成功，都不足以证明完整body/引用/cut通过。finish完成decoder、迁移stages及current restorer；current模式包括released V3关系校验和installed Session.fromRestore。

transformed模式不是同等强度：迁移输入不执行installed-current Session验证，native current输入仅codec后identityArtifact。不要将其用作完整验收捷径。catalog只接JSON值，无文本分行或kind/value外层信封适配API。

## V0 header的精确语义

S SessionHeader没有type标签、delegationDepth可省略且表示0。S JSONL toHeaderLine写type=session、delegationDepth=header.delegationDepth??0，其余可选字段仅!==undefined才写。

T frozen V0物理codec要求type/version/id/createdAt/delegationDepth；可选cwd/parentSession/seedLength/origin/agentPreset。缺depth不是目标codec允许的隐式默认，字段名为delegationDepth。只能依据S合法逻辑缺省转为0，不能将非法显式值“修成0”。

seedLength缺席→isSeeded=false、cut=0；显式seedLength:0→isSeeded=true、cut=0，必须保留差别。V0不能直接加V3 isSeeded/inheritedEventCount。后续链可能改变事件数，最终cut取finish输出，不沿用旧数字。

cwd/parentSession/origin/preset保留值与有无状态，勿提前code→ptc、勿猜本机路径或继承长度。V0 sourceEventSeqs数字数组及范围形式可被codec读取；这不意味着任意wrapper也会被识别。

## 未知、截断与unfinished

- V0未知historical event即使ignorable也拒绝；V1/V2迁移同样按其frozen inventory审计。native V3对opaque ignorable扩展有不同规则，不能反推旧日志迁移。
- 半行JSON/无效文本字节在catalog rowValue层之前；catalog不能凭header成功证明文本完整。
- strict拒绝row结构/seq错误；recoverable可抑制部分坏尾，不代表无损，还可能因中段已封闭边界或unsupported词汇拒绝。
- 完整可解析、合法未结束前缀可以接受，不要求EOF时一律关闭turn/step/PTC。官方migration.spec.ts:455接受unfinished PTC start而不补fabricated settlement；孤立settlement和身份不配对仍拒绝。
- step/end/turn/end实际出现时，会校验open状态和未解决tool等关系。没有closer与“出现错误closer”不同。
- 丢失完整尾行与真实未结束可能产生完全相同输入；catalog没有外部expected length/hash/terminal状态，不能区分。strict/current是格式关系验收，不是原文件采集完整性证明。

## 引用、系统提示词与PTC

源seq须从0稠密，sourceEventSeqs/surface/compaction/title/seed cut须满足局部及全日志关系。不得删除失败事件、重排seq或加ignorable后把修改结果说成原输入通过。

V0→V1包含旧词汇规范化，V1→V2重组assistant streams；V2→V3首step插入system head，request/header.system变更触发head替换并移除旧system字段，重映射同artifact引用与cut。空/缺席清prompt，空白字符串保留。只改version不是迁移。

精确code preset→ptc，tool/code-dispatch(-start)→tool/ptc-dispatch(-start)，限定位置的tools-code-mode归属→tools-ptc；不全文替换任意字符串，不改run_code及其code参数。envelope start/end→startSeq/endSeq；缺surfaceOp或矛盾tool error/isError不静默修复。

不同引用轴不会全部递归改写，外部captured/delivery坐标、ids等有保留规则。current encoder不是全文关系validator，需完成current finish；错误不限于单一error类/固定消息。

## 证据及未验证

固定T关键绝对路径：

- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-format-catalog/src/generated.ts`：current校验链。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-format/src/catalog.ts`：finish及validation分支。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-format-v0-to-v1/src/codec.ts`：物理header及rows。
- `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-format-v2-to-v3/tests/combined-migration.spec.ts`：公开catalog/current迁移与重读。

S证据须按b150a551读取 `/Users/majiajun/workspace/DSH-Expert/upstream/deepseek-harness/packages/session/session-persistence-jsonl/src/format.ts#toHeaderLine`，不取当前工作树替代。

[正式catalog](https://registry.npmjs.org/@deepseek-ai%2Fdsh-session-format-catalog/0.1.5-rc.2)。本轮无代码执行/编译或真实输入迁移，未将静态核验记为运行绿测；字节完整性、外部wrapper同义性与副本发布策略不由catalog保证。
