# Backlog / Prioritized TODO

用于持续跟踪 `openclaw-channel-dingtalk` 的任务优先级、推进顺序和治理项。

---

## P0

### 1. 消息接收稳定性 / 长时间待机可靠性
相关 Issues：
- [#104 Inbound message missing](https://github.com/soimy/openclaw-channel-dingtalk/issues/104)
- [#164 经常无法接受到消息无法回复](https://github.com/soimy/openclaw-channel-dingtalk/issues/164)
- [#151 过几个小时后经常断线](https://github.com/soimy/openclaw-channel-dingtalk/issues/151)
- [#187 待机太久出现不可用](https://github.com/soimy/openclaw-channel-dingtalk/issues/187)

相关 PRs：
- [#96 fix(dingtalk): harden inbound message handling](https://github.com/soimy/openclaw-channel-dingtalk/pull/96)
- [#167 fix(dingtalk): clean up heartbeat timer on reconnect](https://github.com/soimy/openclaw-channel-dingtalk/pull/167)
- [#183 fix(dingtalk): add stale lock cleanup on send path](https://github.com/soimy/openclaw-channel-dingtalk/pull/183)
- [#249 improve DingTalk connection troubleshooting](https://github.com/soimy/openclaw-channel-dingtalk/pull/249)

任务：
- [ ] 复核现有稳定性问题是否仍可复现
- [ ] 明确各已合并 PR 的覆盖范围与遗漏场景
- [ ] 建立最小回归用例：常规收发、长待机后收发、断线恢复、连接失败排障
- [ ] 形成“已收敛 / 未收敛 / 需新增 issue”结论

### 2. AI Card 发送链路一致性
相关 Issues：
- [#166 回复done问题](https://github.com/soimy/openclaw-channel-dingtalk/issues/166)
- [#197 ai卡片模式下，主动发消息内容不正常](https://github.com/soimy/openclaw-channel-dingtalk/issues/197)
- [#136 同样的内容，会出现两遍](https://github.com/soimy/openclaw-channel-dingtalk/issues/136)

相关 PRs：
- [#191 serialize send pipeline with session-scoped dispatch lock](https://github.com/soimy/openclaw-channel-dingtalk/pull/191)
- [#179 hardening send path](https://github.com/soimy/openclaw-channel-dingtalk/pull/179)

任务：
- [ ] 回归 Done 提前结束问题
- [ ] 回归空回复问题
- [ ] 回归主动通知内容异常问题
- [ ] 回归重复输出问题
- [ ] 判断剩余问题是否需要拆出新的修复任务

### 3. 文件上传 / 文件读取 / 文件预览 / 大文件链路
相关 Issues：
- [#207 通过钉钉无法上传文件给Openclaw](https://github.com/soimy/openclaw-channel-dingtalk/issues/207)
- [#218 群里的文件都以 file 后缀结尾，无法预览](https://github.com/soimy/openclaw-channel-dingtalk/issues/218)
- [#101 支持访问钉盘文件](https://github.com/soimy/openclaw-channel-dingtalk/issues/101)
- [#125 大文件分片上传](https://github.com/soimy/openclaw-channel-dingtalk/issues/125)

相关 PRs：
- [#68 feat(dingtalk): implement file uploading with send image, video and file support](https://github.com/soimy/openclaw-channel-dingtalk/pull/68)

任务：
- [ ] 核对基础文件发送能力的当前边界
- [ ] 拆分文件上传、读取、预览、扩展名处理、大文件、钉盘访问子任务
- [ ] 为每个子任务补最小复现和验收标准
- [ ] 明确哪些属于补尾，哪些需要新增开发

### 4. 图片 / 语音 / 媒体链路补强
相关 Issues：
- [#162 无法发送图片](https://github.com/soimy/openclaw-channel-dingtalk/issues/162)
- [#86 在AI的流式卡片上也实现插入图片/视频音频等](https://github.com/soimy/openclaw-channel-dingtalk/issues/86)

相关 PRs：
- [#182 support local image sending](https://github.com/soimy/openclaw-channel-dingtalk/pull/182)
- [#200 support asVoice for media messages](https://github.com/soimy/openclaw-channel-dingtalk/pull/200)
- [#181 add mediaMaxMb override for inbound media size limit](https://github.com/soimy/openclaw-channel-dingtalk/pull/181)

任务：
- [ ] 回归本地图片发送
- [ ] 回归语音消息发送
- [ ] 回归入站媒体大小限制覆盖配置
- [ ] 评估 AI Card 内媒体一体化展示是否值得推进
- [ ] 明确哪些项已完成、哪些项仍待开发

---

## P1

### 5. 引用消息 / chatRecord / 转发记录解析收口
相关 Issues：
- [#126 引用消息支持](https://github.com/soimy/openclaw-channel-dingtalk/issues/126)
- [#205 chatRecord 消息显示空白](https://github.com/soimy/openclaw-channel-dingtalk/issues/205)
- [#208 优化 chatRecord 空内容提示](https://github.com/soimy/openclaw-channel-dingtalk/issues/208)
- [#227 forward/chatRecord 相关问题](https://github.com/soimy/openclaw-channel-dingtalk/issues/227)

相关 PRs：
- [#128 feat(dingtalk): support quoted messages](https://github.com/soimy/openclaw-channel-dingtalk/pull/128)
- [#202 fix(dingtalk): improve quote and forward handling](https://github.com/soimy/openclaw-channel-dingtalk/pull/202)
- [#209 fix(dingtalk): handle empty forward payload](https://github.com/soimy/openclaw-channel-dingtalk/pull/209)
- [#210 fix(dingtalk): improve quote journal compatibility](https://github.com/soimy/openclaw-channel-dingtalk/pull/210)
- [#233 quote/chatRecord/stream-card hardening](https://github.com/soimy/openclaw-channel-dingtalk/pull/233)
- [#257 clarify empty chatRecord payload behavior](https://github.com/soimy/openclaw-channel-dingtalk/pull/257)

任务：
- [ ] 梳理各 PR 的前后替代关系
- [ ] 明确引用消息解析的最终行为
- [ ] 明确 chatRecord 空内容提示策略
- [ ] 明确转发记录展示策略
- [ ] 做一次集中回归并沉淀结论

### 6. 建立 Issue 提交标准化
任务：
- [ ] 新增 Bug report 模板
- [ ] 新增 Feature request 模板
- [ ] 新增 Regression report 模板
- [ ] 新增 Docs / onboarding 模板
- [ ] 统一必填字段：版本、部署方式、AI Card、messageType、复现步骤、预期/实际、日志、复现稳定性
- [ ] 建立标签标准：stability / streaming / media / files / ai-card / multi-agent / docs
- [ ] 在 README 或贡献文档中补提单说明

### 7. 建立 PR / Issue 自动化流程
相关参考：
- [npm publish workflow](https://github.com/soimy/openclaw-channel-dingtalk/blob/main/.github/workflows/npm-publish.yml)

任务：
- [ ] 新增 PR Template
- [ ] 增加 Issue / PR labeler
- [ ] 增加 stale 自动提醒
- [ ] 增加 needs-info 自动提醒
- [ ] 增加 `Closes #...` / `Refs #...` 校验
- [ ] 增加 PR CI gate：type-check / lint / test
- [ ] 增加 release note automation
- [ ] 规划自动化分阶段落地顺序

---

## P2

### 8. 多账号 / 多 agent / schema 与路由配置收敛
相关 Issues：
- [#130 多 agent 配置相关](https://github.com/soimy/openclaw-channel-dingtalk/issues/130)
- [#132 多账号 schema / ControlUI 兼容相关](https://github.com/soimy/openclaw-channel-dingtalk/issues/132)

相关 PRs：
- [#133 feat(dingtalk): add multi-account schema support](https://github.com/soimy/openclaw-channel-dingtalk/pull/133)
- [#137 refactor(dingtalk): modularize channel implementation](https://github.com/soimy/openclaw-channel-dingtalk/pull/137)

任务：
- [ ] 补齐配置示例
- [ ] 增加配置校验
- [ ] 优化启动时报错
- [ ] 补齐文档说明

### 9. 支持群聊 @人 / @all
相关 Issues：
- [#67 机器人群聊中支持@某人](https://github.com/soimy/openclaw-channel-dingtalk/issues/67)

任务：
- [ ] 明确 @单人 需求范围
- [ ] 明确 @多人 需求范围
- [ ] 明确 @all 需求范围
- [ ] 设计失败降级与兼容行为

### 10. 支持对话打断 / 取消任务
相关 Issues：
- [#76 对话打断功能](https://github.com/soimy/openclaw-channel-dingtalk/issues/76)

任务：
- [ ] 明确用户主动取消的交互形式
- [ ] 明确长任务中断机制
- [ ] 明确中断后状态回收方式
- [ ] 明确 UI / 提示语反馈

### 11. AI Card usage footer / thinking 展示行为可配置
相关 Issues：
- [#111 usage footer](https://github.com/soimy/openclaw-channel-dingtalk/issues/111)
- [#236 思考中如何关闭](https://github.com/soimy/openclaw-channel-dingtalk/issues/236)

相关 PRs：
- [#119 AI Card thinking/tool use streaming](https://github.com/soimy/openclaw-channel-dingtalk/pull/119)
- [#214 make thinking message configurable](https://github.com/soimy/openclaw-channel-dingtalk/pull/214)

任务：
- [ ] 明确 thinking 展示可配置项
- [ ] 明确 tool use 展示可配置项
- [ ] 明确 usage footer 展示策略
- [ ] 补齐默认值与文档说明

---

## P3

### 12. 流式与响应时延
相关 Issues：
- [#238 新版本：延迟20S](https://github.com/soimy/openclaw-channel-dingtalk/issues/238)
- [#260 发现流式是假的，是最终生成完以后才开始流](https://github.com/soimy/openclaw-channel-dingtalk/issues/260)

相关 PRs：
- [#255 make stream keepAlive configurable and default off](https://github.com/soimy/openclaw-channel-dingtalk/pull/255)

任务：
- [ ] 评估 20 秒延迟是否仅在超大规模部署下发生
- [ ] 评估个人场景是否真的需要真流式
- [ ] 评估消息更新 API 配额消耗
- [ ] 评估低更新频率的替代方案
- [ ] 修复 chunk 模式仅显示增量字符的问题
- [ ] 给出“继续投入 / 保持现状”的结论

### 13. README / 截图 / onboarding / 配置说明补齐
相关 Issues：
- [#242 README 能否增加一些图片截图](https://github.com/soimy/openclaw-channel-dingtalk/issues/242)

相关 PRs：
- [#175 docs: align README cardTemplateKey default](https://github.com/soimy/openclaw-channel-dingtalk/pull/175)
- [#199 docs: align onboarding and runtime defaults](https://github.com/soimy/openclaw-channel-dingtalk/pull/199)

任务：
- [ ] 补 README 截图
- [ ] 补 onboarding 示例
- [ ] 补配置说明
- [ ] 补常见问题
- [ ] 补排障说明

---

## Milestone 建议

### Milestone A：基础可用性收敛
- [ ] 消息接收稳定性 / 长时间待机可靠性
- [ ] AI Card 发送链路一致性
- [ ] 文件上传 / 文件读取 / 文件预览 / 大文件链路
- [ ] 图片 / 语音 / 媒体链路补强

### Milestone B：高频消息类型与治理能力补齐
- [ ] 引用消息 / chatRecord / 转发记录解析收口
- [ ] 建立 Issue 提交标准化
- [ ] 建立 PR / Issue 自动化流程

### Milestone C：配置与体验增强
- [ ] 多账号 / 多 agent / schema 与路由配置收敛
- [ ] AI Card usage footer / thinking 展示行为可配置
- [ ] README / 截图 / onboarding / 配置说明补齐

### Milestone D：增强能力与低优先级优化
- [ ] 支持群聊 @人 / @all
- [ ] 支持对话打断 / 取消任务
- [ ] 流式与响应时延

---

## 建议拆分顺序

### 第一阶段
- [ ] 消息接收稳定性 / 长时间待机可靠性
- [ ] AI Card 发送链路一致性
- [ ] 文件上传 / 文件读取 / 文件预览 / 大文件链路
- [ ] 图片 / 语音 / 媒体链路补强

### 第二阶段
- [ ] 引用消息 / chatRecord / 转发记录解析收口
- [ ] 建立 Issue 提交标准化
- [ ] 建立 PR / Issue 自动化流程

### 第三阶段
- [ ] 多账号 / 多 agent / schema 与路由配置收敛
- [ ] AI Card usage footer / thinking 展示行为可配置
- [ ] README / 截图 / onboarding / 配置说明补齐

### 第四阶段
- [ ] 支持群聊 @人 / @all
- [ ] 支持对话打断 / 取消任务
- [ ] 流式与响应时延