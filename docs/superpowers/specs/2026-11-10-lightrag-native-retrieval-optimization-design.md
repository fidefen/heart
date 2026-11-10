# LightRAG 统一教材检索与原生预探索优化设计

日期：2026-11-10
状态：用户已在聊天中确认设计边界；本文固化需求，尚未实施。
实施计划：../plans/2026-11-10-lightrag-native-retrieval-optimization.md

## 1. 已确认的需求

1. 所有课程事实性问答必须依据教材，即使用户不需要显示引文，也不能跳过检索后凭模型知识补答。
2. 直接使用已经由 LightRAG 构建的教材库，替换新回答链路中的 legacy 和 local_llamaindex 两路；不重建、复制或删除教材索引。
3. 课程中只挂载一个规范教材库；原有两个教材库引用自动映射并去重，已有历史答案、引用与任务收据保留。
4. LightRAG 原文证据不足时，允许使用原生 ask_user 追问范围或选区；不得自动回退旧引擎，也不得把图谱摘要当原文。
5. 保留 DeepTutor 原生 explore_context，自主选择要读的材料；只优化重复读取和当前回合文本体积，不另写意图分类器、探索 Agent 或教学循环。
6. 本次仅形成设计和实施计划。正文流转发、能力路由提速和逐事件授权查询优化不属于本轮实施范围。

## 2. 证据与现状

| 样本 | 任务创建到答案落库 | 模型累计耗时 | 已识别的主要开销 |
|---|---:|---:|---|
| 2026-11-10 自动入口一句话解释自变量 | 46.925s | 1.334s | 两路教材预检索的等待区间约31.2s |
| 同批次手动普通对话 | 34.472s | 1.873s | 两路教材预检索的等待区间约24.1s |
| 同批次一道测验题 | 133.803s | 59.403s、14次调用 | 三批检索等待约63s |
| 2026-11-09 新会话教材问答 | 冒烟记录117.09s | 22.619s | 原生回合101.853s，多次检索及准备 |
| 2026-11-09 长会话教材问答 | 冒烟记录69.90s | 30.188s | 预探索12次 read_source，最终主回答输入59644 tokens |

证据位置：
- project/integration/discrete-math-tutor/.runtime/native-delivery-live/20261110-103531/{auto-browser.json,course.db,host.log,home/worker.log}
- 同项目 .runtime/deeptutor/heart-bridge/bindings.sqlite3 及相应用户工作区 chat_history.db。
- .superpowers/sdd/2026-11-09-preserve-context-compaction-and-startup-warmup/progress.md。

上述样本并非统计性能承诺。模型累计时间不是墙钟时间；预探索、内部检索模型、并行工具和宿主处理必须分开记录。旧记录出现过261576 tokens的单次输入，其 kb_files 实现已发生后续变更，不能将它当作最新版本必现故障。

已核实的接口事实：
- course_rag_provider.register_course_sources() 同时挂载两个课程KB；CoursePipeline.search() 实际调用 psychology_retrieval.retrieve()。
- 原生 AgenticLoopPipeline._retrieve_kb_seed_block() 会并行查询已挂载的传统RAG库，并等待全部完成。
- 原生 LightRagPipeline.search() 读取其原生发布目录布局，按次 build/initialize/finalize；宿主 LightRagBackend 已适配现有 candidate、原文映射和常驻 ResidentGraph。不能只改 provider 字符串就认为目录、模型绑定和来源结构兼容。
- memory.background_worker.memory_lifespan() 已在宿主启动/关闭 fixed_book；worker 没有同一个进程内 runtime，不能从 worker 直接引用宿主 active()。
- 原生 AgentLoop.run() 的顺序是：材料预探索 → KB seed → 主回答。原生预探索最多5轮；8000/48000字符上限目前只约束它的单次 fallback 路径，不能据此宣称原生工具循环也已限额。
- ReadSourceTool 当前只有 source_id 参数，没有分页参数，默认返回该ID的全部文本。
- 当前 native_executor 允许部分普通请求使用 allow_model_knowledge，必须收紧课程事实发布门禁，才能满足需求1。

## 3. 选定方案及被排除的方案

选定：保留原生 rag/RAGService 的发现、授权、调度与工具协议；沿用 course_rag_provider 作为课程专用薄适配层，连接宿主已经运行的 LightRagBackend。

不选：
- 仅将两个库的 provider 改成 lightrag：不能保证现有 candidate 布局兼容，也会失去现有常驻和Q2映射路径。
- 再在 worker 加载一份图谱：增加冷启动、内存及同一索引多进程所有权问题。
- 为简单/复杂问题再新增分类模型：增加调用成本与误判，用户选择保留原生自主探索。

## 4. 目标架构与接口

课程文本查询：
原生 rag → RAGService → 课程 LightRAG 薄适配 → 宿主受控查询接口 → fixed_book.active().backend.search() → 已有原文映射 → prepare_retrieval() → 原生 sources/read_source → 宿主发布门禁。

### 4.1 单一课程资源

- 规范KB名：experimental-psychology-3-lightrag，产品显示“课程教材 · LightRAG”。
- 课程薄适配 provider 标识：heart-course-lightrag。这只是 RAGService 到已有宿主库的接口标识，不是新检索引擎；实际后端必须标记 lightrag。
- 通用原生 lightrag provider 与用户上传知识库继续由原生实现处理，禁止全局劫持所有 lightrag 请求。
- 两个旧课程KB名只作为输入兼容别名。映射要求课程、受权账户和教材版本一致；其他资源保持原顺序与权限。
- 课程资源、会话偏好、本轮显式KB列表都要覆盖。映射在原始请求哈希核验后形成执行投影，不改旧 request_id 的原始payload、已保存 start、历史消息或教材引用。
- 活跃或待恢复回合继续遵守原收据语义；无法兼容的旧回合明确提示重新发起，不伪造一个新回合冒充重放。新请求不再执行旧课程引擎。
- 只解除课程对两个旧KB的挂载，保留旧注册记录及索引供历史兼容和回滚。重复迁移应无额外变化。

### 4.2 运行时与受控查询

- 常驻图谱继续由宿主 fixed_book 生命周期唯一管理，不迁移索引、不新增图谱服务。
- 新增宿主内部 POST /internal/heart/native/course-search，使用现有桥接令牌和 NativeTaskCheck，复用现有回查规则。
- 请求仅允许任务身份、query（1～4000字符）、mode（现有LightRAG枚举，缺省hybrid）、limit（1～12）以及可选章/节。不得接受客户端文件路径、候选库路径或未授权KB标识。
- 候选路径、模型/embedding绑定、课程来源根均由宿主配置及当前教材快照决定。对普通问答、测验、学习和研究均使用同一入口。
- 查询前后检查账户、任务revision、memory_epoch及material_versions；取消、断连与超时必须释放本次查询/队列资源，保留现有所有权与预算机制。
- worker通过现有 DeepTutorBackend 的本机传输调用，使用 asyncio.to_thread 隔离同步HTTP；沿用禁止代理、重定向和非本机地址的约束。
- 返回结构化原文来源、必要图表信息、状态和耗时。图谱关联只放辅助字段，不获得教材证据身份。
- provider继续用 as_textbook_chunk()、prepare_retrieval()及现有图表映射逻辑适配 sources，保留 source_id、document_id、version、block/quote范围、稳定引用编号和rag_source_id。不能漏掉旧LightRagNavigator里已有的图表版本/sha256核验。
- 目录探测核验 LightRAG 的 manifest/来源绑定，不再以旧rag_index.sqlite3的向量计数判断新库就绪。实际查询还必须检查宿主runtime.ready。

### 4.3 原生预探索与检索共享

- 复用 explore_context、ContextExplorer、ReadSourceTool 和原生主循环，保持最多5轮，不复制实现，不增设另一轮“预探索”。
- 仅有课程KB、没有附加可读材料时：单次 LightRAG seed 提供已核验原文后进入主回答，不因适配层新登记的rag来源额外强制启动材料预探索；原生主循环仍可按需补查。
- 已有显式选区、阅读材料、笔记本或历史材料可探索时：在既有 pre_loop 适配边界准备本轮课程seed，补入可读的教材来源和目录，再调用一次原生预探索。后续原生seed调用直接复用同一结果，不再发相同查询。
- 不把所有历史材料全文直接送入模型。历史来源保留精简目录，原文仍按需可读，保留分支隔离，不按字符串相似度删除材料。
- 来源合并键必须包含原文身份、版本和读取范围；来源别名合并后保留可追溯映射，不跨账号、任务或分支共享。
- 同回合同请求参数的检索共享执行中/已完成结果；不同query、mode、范围、limit或版本不直接复用。取消和异常不缓存为成功。
- read_source已经返回且仍存在当前工作上下文的相同范围，可返回简短“本轮已读”指针；若原结果已不在上下文，则必须重新提供文本，不能生成悬空指针。
- 不以“达到轮数”当作证据充分。原生探索失败或预算不足可降级为已有核验原文，但正式答案仍受证据门禁。

### 4.4 当前回合文本控制

- 优先缩小工具本身的返回体积：kb_files只提供真实文件ID/名称/计数，不附页坐标、全文、embedding或图谱JSON；raw source_id清单不可泄露物理路径。
- 课程文本段上限8000字符，当前聊天回合课程工具可见正文总预算48000字符。使用现有历史默认阈值8000，并以显式不同语义的常量标注“当前回合总预算”，不把fallback旧常量误当成已生效策略。
- 长来源以稳定子来源ID分段，ID映射到文档版本及原文范围；模型仍通过原生read_source(source_id)读取。父项显示分段目录，不能悄悄把被截断文本称为完整原文，也不凭空增加原生分页参数。
- 完整原文及引用映射保留在服务端本轮来源表；模型侧用片段和可继续读取的来源ID，审计/恢复仍可定位原文。schema、tool_calls、tool_call_id及provider response state不做字符串截断。
- 预算计数覆盖本轮预探索和主回答中的课程工具内容；不把Word/exec等无关能力的工具结果统一裁切。研究/测验复用其原生阶段预算和有界课程工具结果，不强制套用聊天总预算来砍掉正式交付。
- 已满预算且仍缺必要材料时走原生ask_user或显式证据不足状态；不能保留残缺引文并正常发布。
- 仍保留现有历史压缩和启动预热约束；本轮不重写历史metadata、不做新的批量压缩。

### 4.5 失败与发布

- no_evidence表示缺乏原文：允许追问具体章节、概念或选区；不自动回退legacy/local_llamaindex。
- not_ready、版本错误、权限错误和后端超时分别记录；服务故障应提示稍后重试，不能都伪装成“请缩小范围”。
- ask_user使用现有持久化澄清/恢复协议，不增加单独问题卡协议。
- 课程事实答案必须具有当前版本、已核验教材来源；关闭普通事实请求的无来源model-knowledge放行。纯追问不当作已完成事实答案，格式化/Word任务保留其原有交付门禁。
- 用户不要求展示编号时可以自然改述，但服务端必须保留证据；不能为了隐藏编号而丢失来源审计。

## 5. 验收标准

正确性先于速度：
- 课程只有一个规范教材KB；本轮默认只发一次同参数LightRAG seed；旧两引擎实际调用次数为0。
- 普通问答、追问、选区、跨章、图表、测验、研究均有来源映射测试；缺证据、故障、越权、教材更新与取消有负例。
- 复用原生预探索、ask_user和模型循环；无重复前置循环，无隐性旧引擎回退。
- 工具结果和当前课程文本预算有可运行断言；必要原文仍可按范围继续读取，工具协议和原生provider state完整。
- 性能记录包括host admission、LightRAG所有权等待/召回/重排/原文映射、探索、模型调用、发布落库；同时报告原生首事件、首正文和最终答案时间。
- 新旧候选在同机器、同模型/资料版本下，用相同题集记录冷启动与预热后耗时。每类至少3次，报告中位数、最小/最大值、tokens和调用数，不用3次样本宣称P95。
- 不预先承诺“回答降到几秒”；预热后中位数未改善或来源覆盖下降则不切换，报告剩余最长阶段。

## 6. 发布与回滚

先在独立临时home和宿主数据库副本验证；真实模型成本与样本数明确记录。先形成可审阅的结果再进行正式8000/18001切换。
回滚恢复上一版代码和新增绑定前的课程/偏好快照；不删除共享LightRAG索引，不整库覆盖，避免覆盖验证后新增消息。
因故障回滚部署属于运维恢复，不属于运行时自动切旧检索引擎。索引不可写、没有显式批准的重建/删除操作。

## 7. 文档自审结论

已核对：三项优化均有边界和验收；LightRAG宿主/worker进程归属明确；原生provider与已有候选格式差异明确；原生fallback上限与工具循环上限已区分；旧收据不重写；只限课程工具，不误裁其他能力。
本文件和实施计划属于本轮交付，不代表功能已经实现或测试通过。
