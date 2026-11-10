# LightRAG 统一教材检索与原生预探索优化实施计划

日期：2026-11-10
状态：计划已编写并自审，尚未执行任务。
设计依据：../specs/2026-11-10-lightrag-native-retrieval-optimization-design.md
授权范围：本轮编写文档；以下复选框记录未来实施，不因计划写完而勾选。

## 1. 目标与约束

目标：用已构建的LightRAG教材库替换新回答链路的两路本地教材检索；复用原生预探索；控制当前回合课程工具文本，减少实际耗时并保留教材证据质量。

- 所有课程事实问答必须依据教材；缺证据可追问，不自动回退旧引擎。
- 不重建/复制/删除教材索引，不修改project/vendor/DeepTutor固定副本，不新增Agent循环或意图分类模型。
- 既有rag、RAGService、LightRagBackend、原文核验、原生预探索和ask_user均复用。
- 原始payload、request_id收据、已交付历史与引用不改写；只对新执行投影做旧KB别名映射。
- 本轮不实现正文流转发、能力路由优化、逐事件授权查询优化，也不做全仓库重构。
- 所有现有未提交修改视为用户工作，实施前记录diff；不重置、不混合提交。
- 独立测试数据先验证；正式服务切换单列，不能因写计划而自动重启。
- 单独串行执行本计划；如未来需要委派，由用户明确授权，不因本文件默认启动子代理。

路径约定：下文P=project/integration/discrete-math-tutor，V=project/vendor/DeepTutor。命令从P运行，使用.venv-deeptutor/Scripts/python.exe；node使用本机既有安装。
环境约定：测试设置DEEPTUTOR_HOME与COURSE_DATABASE_URL指向独立目录；仅设置DEEPTUTOR_HOME不够隔离宿主数据。真实测试本机HTTP禁用代理。

## 2. 实施前必须理解的调用链

1. P/integrations/deeptutor/native_bridge.py:_bind → register_course_sources。
2. P/integrations/deeptutor/heart_rag.py → V/deeptutor/tools/rag_tool.py → V/deeptutor/services/rag/service.py → factory/provider_binding。
3. P/memory/background_worker.py:memory_lifespan → knowledge.fixed_book.start/stop；现有常驻图谱属于宿主。
4. P/knowledge/lightrag/retrieval.py:LightRagBackend.search → query_ownership → resident.query → map_chunks/rank_sources；仅输出检索原文和辅助关联。
5. V/deeptutor/agents/loop/agent_loop.py:run：pre_loop_briefings → kb_seed → 主循环。
6. P/integrations/deeptutor/native_grounding.py:prepare_retrieval → native_delivery.publication_sources → native_executor → _publish_gate。
7. V/deeptutor/tools/builtin/__init__.py:ReadSourceTool仅接受source_id，返回完整文本；分段必须在宿主来源适配中实现，不能假定原生已有分页。
8. P/integrations/deeptutor/local_research.py另有provider白名单与KB授权，必须同步单库迁移。

## 3. 依赖顺序

任务0基线 → 任务1受控宿主接口 → 任务2单库接入/迁移 → 任务3证据不足与发布门禁 → 任务4原生预探索复用 → 任务5本轮文本预算 → 任务6组合回归 → 任务7真实性能验收 → 任务8切换。
每个任务必须先有对应失败断言，修改后执行相关测试；出现新失败先定位，不叠加无关修复。

### 任务0：冻结基线与来源契约

文件：读取现有代码、索引manifest和测试记录；新增 P/tests/data/native_lightrag_latency_cases.json。
验证脚本在任务7统一新增，避免先写一套临时测量实现。

- [ ] 记录git HEAD、dirty文件清单、模型/SDK/embedding身份、教材release及candidate签名；输出不含密钥。
- [ ] 只读核对LIGHTRAG_CANDIDATE_PATH现有input_manifest/progress/policy/index，确认教材11章及原文映射完整；不能用原生provider目录探测失败推导“需要重建”。
- [ ] 固定题集：定义、同概念追问、跨章比较、显式选区、指定原图、一道测验、一个子主题研究、无证据问题、同回合重复读取、长会话历史。图表题以实际有效asset_id固定。
- [ ] 对照实际题面和证据对象，记录每题必需来源/章节和允许追问的情形；不能仅以回答中有关键词判定通过。
- [ ] 确认最新基线及测试隔离，实际旧引擎查询仅存在于候选A测量，不能在新链路中作失败兜底。

完成条件：基线、题集、来源断言和对照条件可复用；不能跳过正确性基线直接宣称性能改善。

### 任务1：让原生worker查询宿主现有LightRAG运行时

新增：P/integrations/deeptutor/course_search_api.py。
修改：P/integrations/deeptutor/{client.py,native_contract.py}、P/main.py。
测试：新增 P/tests/test_native_course_search.py。
复用：native_authorization、task_projection、fixed_book、lightrag/retrieval；不改SDK/图谱算法。

- [ ] 先写失败测试：未授权令牌/任务、非本课程、revision/epoch/版本不符、非法query/mode/limit、服务未就绪均拒绝；请求不允许注入路径。
- [ ] 定义CourseSearchRequest复用NativeTaskCheck；实现POST /internal/heart/native/course-search，查询前后核验，调用宿主active().backend.search()。
- [ ] 返回既有结构化原文/图表/graph_context/usage/trace；只裁投影字段，不丢document_id/version/原文范围/映射所需location。
- [ ] DeepTutorBackend新增对应方法，继续使用本机地址校验、无代理、无重定向；响应限额显式限制，超限失败，不能截断JSON后继续处理。
- [ ] 接通原生取消、断连和预算到宿主查询生命周期；不在worker第二次初始化ResidentGraph。测试取消后的下一请求能获得所有权。
- [ ] 校验host→worker→host回查不持有全局登记锁等待模型，避免循环等待。
- [ ] 执行test_native_course_search.py与test_lightrag_backend.py、test_lightrag_acceptance_instrumentation.py。

完成条件：受控真实任务可调用现有后端；失败不返回伪造来源；一次查询只进入一个宿主常驻图谱。

### 任务2：规范单一教材库，迁移旧选择

修改：P/integrations/deeptutor/{course_rag_provider.py,heart_rag.py,native_bridge.py,local_research.py}。
必要时修改同目录native_routing.py的执行选项规范化；保持原始请求收据。
测试：新增 P/tests/test_native_course_kb_migration.py、test_native_course_rag_provider.py。
更新：P/scripts/check_native_course_rag_browser.cjs及check_native_chat_chain.py、check_native_delivery_browser.py中与两引擎绑定的夹具。

- [ ] 写失败测试：默认只有experimental-psychology-3-lightrag；两个旧名、别名“课程教材/实验心理学”均归一；两旧名同时选择只保留一次；非课程KB不被重写。
- [ ] 课程适配provider采用heart-course-lightrag，在既有factory安装边界注册；通用原生lightrag处理不受影响。
- [ ] CoursePipeline.search改为任务作用域下调用任务1接口，不再调用psychology_retrieval；实际后端标记lightrag。通过learning_tool_adapter.current_learning_context()获取身份，缺上下文拒绝。
- [ ] 用as_textbook_chunk与prepare_retrieval转换原文；复用LightRagNavigator中已有的图表sha256/版本校验逻辑，不丢image_refs。
- [ ] 原生CourseService挂载规范KB并解除两个旧课程资源；不删除原注册项/索引。get_catalog/kb_files仍显示真实教材目录而不是候选内部文件。
- [ ] 新会话默认、旧会话preferences、本轮显式选择与研究范围都做规范化和授权；显式选择其他上传KB仍保留，不扩大到未选资源。
- [ ] 迁移幂等，原payload/start/digest不改；旧收据重放返回原结果；无法兼容的活跃回合提示用户按原协议取消后重新发起，不自动结束任务，也不能偷偷调用旧引擎完成。
- [ ] mock旧retrieve_hybrid、local_llamaindex.retrieve为“调用即失败”，证明新课程问答、图表和研究请求都不会进入它们。
- [ ] 更新浏览器脚本“两库数量等于2/逐引擎循环”断言为单库、旧选择归一、真实来源/刷新恢复；旧引擎独立模块测试保留。
- [ ] 执行两个新测试以及test_native_grounding.py、test_native_figure_tools.py。

完成条件：一个课程KB、一个实际LightRAG后端，默认重复查询消失，旧选择可兼容且未扩大权限。

### 任务3：无证据追问与课程事实发布门禁

修改：P/integrations/deeptutor/{native_executor.py,tool_policy.py,course_rag_provider.py}；仅在现有边界需要时修改native_delivery.py。
测试：扩充test_native_executor.py、test_native_grounding.py，新建test_native_course_evidence_policy.py。

- [ ] 写失败测试：普通概念问答、明确“不需要引文”的问答，无来源均不能发布事实答案；有核验来源可自然改述。
- [ ] 区分no_evidence、not_ready、权限/版本错误、超时；向原生工具结果提供明确状态，不把所有错误压成空列表。
- [ ] no_evidence经既有原生ask_user协议追问范围/选区，用户回答后按同一任务revision规则恢复；不自动换旧引擎。
- [ ] 事实性课程结果收紧allow_model_knowledge放行；澄清等待走原生pending分支，不能当成完成答案。Word等非事实性交付继续保持现有门禁。
- [ ] 验证模型忽略缺证据状态并直接作答时，最终门禁仍拒绝；已有来源也不能让无关问题借壳通过。
- [ ] 执行上述测试，加test_deeptutor_native_runtime.py中的问题卡/恢复相关用例。

完成条件：模型自主探索与追问仍由原生完成，宿主负责不可绕过的最终证据边界。

### 任务4：共享本轮检索与原生预探索

修改：P/integrations/deeptutor/tool_policy.py、course_rag_provider.py；按已有适配模式新增native_source_context.py集中保存本轮来源状态，避免把无关逻辑塞入全局工具策略。
测试：新增test_native_source_context.py，扩充test_native_tool_policy.py。
读取：V/deeptutor/capabilities/explore_context/{capability.py,explorer.py}与agents/loop/pipeline.py。

- [ ] 写失败测试：仅有课程KB时seed仅执行一次，不能因新rag来源再强制启动一次explore_context。
- [ ] 在已有显式材料的pre_loop适配边界先准备单库seed，把合法原文与来源目录交给原生pre_loop一次；原生后续seed从同一上下文复用结果。
- [ ] 记录原生材料是否存在于准备seed之前，保持原生活跃能力/独占能力/工具白名单；不要对mastery等声明skip_kb_seed的能力强行预检索。
- [ ] cache限定当前回合，key包括课程KB、版本、query、mode、limit、章/节；相同并发调用合并，不跨请求/用户/分支缓存，不缓存取消/错误为成功。
- [ ] 来源按文档+版本+范围去重，保留原始别名和引用ID；相似文字但不同版本/范围不合并。
- [ ] 同源已读且仍在模型工作上下文时返回短指针，原文不在上下文时重新返回；有效工具调用都保留对应tool_result，不删除tool_call_id。
- [ ] 验证复杂附加材料仍运行原生ContextExplorer，未复制它的5轮循环；无材料/选区/历史混合场景均有覆盖。
- [ ] 执行test_native_source_context.py、test_native_tool_policy.py、test_native_question_history.py和test_native_reading_scope.py。

完成条件：原生预探索得到本轮合法材料，同题seed不重复，复杂问题补读能力保留。

### 任务5：控制当前回合课程工具文本

修改：P/integrations/deeptutor/native_source_context.py、tool_policy.py、course_rag_provider.py；复用native_history.py的8000默认值，不扩展其批量写库行为。
测试：新增 P/tests/test_native_current_turn_budget.py。

- [ ] 先写失败测试：超长kb_files不含页坐标/全文；read_source来源正文超过8000时可通过稳定子来源ID完整分段读取。
- [ ] 分段按原文结构/范围，目录保留来源版本和后续ID；每段不超过8000字符，不能从截断字符串构造“原文证据”。
- [ ] 聊天当前回合课程工具正文累计上限48000字符，包含预探索；新文本超过预算时返回有界状态并允许ask_user，不输出半截正式引文。
- [ ] 研究/测验沿用原生阶段预算，仍使用有界课程工具结果；不以聊天总预算砍掉研究报告、题卡、Word或exec结果。
- [ ] 把服务端完整原文与模型工具投影分开；避免在content和模型可见metadata中重复附全文。已发布来源映射仍指向完整合法原文。
- [ ] 覆盖native工具循环和fallback两条路径，不能只测试原生单次fallback旧常量。
- [ ] 断言tool_calls/tool_call_id一一对应、Responses原生状态及reasoning/thinking blocks保持原样；不全局json字符串截断。
- [ ] 执行test_native_current_turn_budget.py、test_native_history_compaction.py和相关原生provider-state测试。

完成条件：当前回合不再无界累积课程工具全文，必要原文可继续定位，历史压缩与模型协议未回退。

### 任务6：组合功能与恢复回归

修改/更新：P/scripts/check_native_course_rag_browser.cjs；P/scripts/check_native_research_browser.cjs中的课程KB夹具；适当补充tests/test_native_delivery.py。
文档：更新project/docs/research/2026-11-08-deeptutor-complete-runtime中的迁移进度和已实现边界，按hunk隔离既有修改。

- [ ] 运行任务1～5测试及现有native_executor/native_grounding/native_figure_tools/native_delivery/native_question_history。
- [ ] 验证“检索→预探索→补读→正式引用→PDF原文打开→刷新恢复”闭环。
- [ ] 验证旧两库选择的会话发出新请求只调用LightRAG；原request_id重放不会执行新检索。
- [ ] 验证研究outline/continue的KB范围一致，测验/学习能力未被聊天hook破坏；Word纯格式交付不被课程文本预算截断。
- [ ] 覆盖跨账号拒绝、材料更新、删除记忆、取消/重试；并发两回合不串source_index、编号、缓存或模型配置。
- [ ] 测试实际使用独立worker/home/database；pytest路径使用唯一临时basetemp，避免与既有运行并发写同目录。

建议命令组，从P运行，basetemp每次使用未被占用的独立目录：
.venv-deeptutor/Scripts/python.exe -B -m pytest tests/test_native_course_search.py tests/test_native_course_kb_migration.py tests/test_native_course_rag_provider.py tests/test_native_course_evidence_policy.py tests/test_native_source_context.py tests/test_native_current_turn_budget.py --basetemp=.runtime/pytest-native-lightrag-20261110
.venv-deeptutor/Scripts/python.exe -B -m pytest tests/test_native_executor.py tests/test_native_grounding.py tests/test_native_figure_tools.py tests/test_native_delivery.py tests/test_native_tool_policy.py tests/test_native_question_history.py tests/test_native_reading_scope.py tests/test_native_history_compaction.py --basetemp=.runtime/pytest-native-lightrag-regression-20261110

完成条件：核心功能通过；未运行/跳过的测试及原因单列，不能用受控模型测试代替真实模型性能验收。

### 任务7：真实题集与分段性能对照

新增：P/scripts/check_native_lightrag_latency.py。
复用：scripts/check_preservation_smoke.py的SSE读取思路、现有LightRAG usage/trace、native_timing，不复制整套启动器。
输出：P/.runtime/native-lightrag-latency/<run-id>/report.json及审阅用摘要；文件不含密钥。

- [ ] 脚本先有受控事件自测：首事件、首正文、最终答案分别计时；并行区间不重复相加；失败/追问不能计为answered。
- [ ] 用任务0相同题集及版本做候选A/B；冷启动和预热后分开。每类预热后至少3次，研究/测验也明确样本数和模型成本。
- [ ] 输出query_count、实际backend、source_count、重读次数、探索轮数、输入/输出tokens、最大工具文本、所有权等待、召回、重排、映射、模型及发布耗时。
- [ ] LightRAG内部keyword/query模型开销单列；不能只拿native usage_summary当作全链路所有模型耗时。
- [ ] 断言新链路不执行两旧引擎，默认同参数seed一次；澄清、权限拒绝、取消的结果分别统计。
- [ ] 与人工已核验的证据需求逐题比对；简单概念不能靠减少证据覆盖伪造提速，复杂问题不得因限制误判充分。
- [ ] 报告中位数、最小/最大值，不从3次样本估P95；记录本机负载、模型差异和缓存冷热。
- [ ] 预热后中位数未改善、超时增多或质量下降则保留候选并报告剩余最长阶段，不切换正式服务。

完成条件：有原始记录可复核，清楚区分“重复查询已消除”和“端到端实际更快”；不预设没有实测依据的秒数承诺。

### 任务8：正式切换与定向回滚

- [ ] 整理代码diff、测试结果、性能报告、旧KB迁移映射和回滚清单，形成可审阅结果。
- [ ] 在正式数据变更前备份将修改的课程资源/会话偏好；使用幂等迁移，禁止批量清空用户会话。
- [ ] 将正式8000/18001的切换作为单独操作处理；本轮计划编写不执行重启。
- [ ] 切换后核验单库目录、真实LightRAG后端、原文入口、追问及刷新恢复。
- [ ] 若回归，恢复本次代码和受影响绑定；不整库覆盖、不删除LightRAG候选、历史对话或新增数据。
- [ ] 更新进度文档，仅在任务完成后勾选，不把“已写计划”标为“已实现”。

## 4. 风险与处理

| 风险 | 处理与验证 |
|---|---|
| 原生LightRAG目录与现有candidate不兼容 | 用课程薄适配调用已有宿主LightRagBackend，不重建 |
| 同一索引被host/worker重复加载 | 宿主现有runtime唯一执行查询，worker只做受控调用 |
| 同进程查询锁不能代表跨进程互斥 | 不增加第二查询owner；并发/取消测试覆盖现有write/query所有权 |
| LightRAG内部模型、重排或排队仍慢 | 各自记录trace和usage；用实测决定切换 |
| 别名归一破坏幂等或权限 | 原始payload/hash保留，执行投影映射且重新授权 |
| 预检索提前导致所有问题多一次探索 | 区分原有附加材料与seed新增来源，纯KB不强制预探索 |
| 本轮预算把否定/条件截断 | 原文范围分段，不用任意前缀冒充完整证据；不够就追问 |
| graph_context被当教材事实 | 仅原文来源进入prepare_retrieval/publication_sources |
| 控制模型通过、真实服务仍慢 | 功能测试与真实计时分开报告，保留原始样本 |

## 5. 计划自审

- 用户确认的三个范围均有可执行任务；没有混入正文流/路由/授权性能改造。
- 每任务注明真实路径、依赖、失败断言、修改点和完成条件。
- 新增的宿主接口只是复用常驻LightRAG的进程边界，不是新检索引擎。
- 没有自动回退、索引重建、历史收据改写或直接截断provider状态。
- 没有声称已经实施或测试；所有任务保持未完成。
