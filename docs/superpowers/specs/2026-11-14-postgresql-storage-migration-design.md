# PostgreSQL 全量存储迁移设计（宿主业务库 + LightRAG 教材库）

日期：2026-11-14
状态：经 brainstorming 确认；本文固化需求与设计，尚未实施。
后续独立设计：LangGraph 接入（本设计为其提供 psycopg 与 PostgreSQL 基础，不在本轮范围）。

## 1. 已确认的需求

1. 宿主业务库从 SQLite 迁移到 PostgreSQL，**只支持 PostgreSQL**，删除 SQLite 双后端分支。
2. LightRAG 教材库的 KV、向量、图、文档状态、LLM 缓存全部存 PostgreSQL，**硬切**，不保留文件存储后端。
3. 现有教材索引用**一次性导入**进入 PostgreSQL，不调用 LLM 重新建图；教材原文、doc_id、chunk_id、release 身份和引用坐标不变。
4. 宿主现有数据必须保留：心理学业务库 179.5MB、247 学生、1124 会话、641 学习任务、1106 条 PDF 引用；旧离散数学库 3.3MB、15 学生。
5. 生产与测试都只跑 PostgreSQL；vendor 里 DeepTutor 自己的 `chat_history.db` 等 SQLite 不在本轮范围。
6. PostgreSQL 用本机原生安装（Windows），pgvector 用源码编译；不使用 Docker。

## 2. 环境与库布局

| 项 | 值 |
|---|---|
| PostgreSQL | 18（EDB Windows 安装包），服务 `127.0.0.1:5432` |
| 扩展 | `pgvector 0.8.7`，用 VS `nmake /F Makefile.win` + `nmake /F Makefile.win install`，`PGROOT=C:\Program Files\PostgreSQL\18` |
| `heart` | 旧离散数学业务库，对应现 `ai_tutor.db`（43 表） |
| `heart_psychology` | 当前课程业务库，对应现 `ai_tutor.experimental-psychology-3.db`（49 表） |
| `heart_rag` | LightRAG 全部存储（11 张 `LIGHTRAG_*` 表） |
| Python 驱动 | `psycopg[binary]` 3.3.x，SQLAlchemy 方言 `postgresql+psycopg` |

环境变量：

```
DATABASE_URL=postgresql+psycopg://heart:<pw>@127.0.0.1:5432/heart
COURSE_DATABASE_URL=postgresql+psycopg://heart:<pw>@127.0.0.1:5432/heart_psychology
POSTGRES_HOST=127.0.0.1
POSTGRES_PORT=5432
POSTGRES_USER=heart
POSTGRES_PASSWORD=<pw>
POSTGRES_DATABASE=heart_rag
```

`COURSE_DATABASE_URL` 必须显式指定，保持现有 `course_database_url()` 的"课程库不得与旧库相同"约束。LightRAG 直接读取 `POSTGRES_*`，不新增第二套驱动或配置层。

## 3. 宿主业务库改动

落点与做法：

| 文件 | 改动 |
|---|---|
| `database/database.py` | 删除 SQLite event/`BEGIN IMMEDIATE` 分支；PG 连接池（`pool_pre_ping`、`pool_size`、`max_overflow`）；`session_scope(immediate=True)` 改为事务级 `pg_advisory_xact_lock(2147480001)`，复刻原 SQLite 单写者语义 |
| `database/memory_index.py` | FTS5 换成 `to_tsvector('simple', index_text)` + GIN 表达式索引；检索用 `plainto_tsquery` + `ts_rank_cd`；`index_text` 已是空格分隔中文二元组，`simple` 配置可用 |
| `database/conversation_session_migration.py`、`preference_value_migration.py` | SQL 已是标准 `ALTER TABLE`/`CREATE INDEX`，PG 直接执行 |
| `requirements.txt` | 加 `psycopg[binary]`；移除 `PyMySQL` |
| `.env.example`、`README.md` | 更新连接串与安装步骤 |

`immediate=True` 的 60+ 调用点不改调用方，只在 `session_scope` 内部换实现。这是有意的取舍：本项目并发写入很少且每学生回合已由 `student_turn_lock` 串行化，全局 advisory lock 与当前 SQLite 行为等价。

```python
# shortcut: PG 用单库级 advisory lock 复刻 SQLite 单写者语义；
# 多用户高并发写入成为瓶颈时，改为按 student_id 哈希的细粒度锁。
```

`init_db()` 继续用 `Base.metadata.create_all` + 两个手写迁移，不引入 Alembic。schema 演进方式与现在一致。

## 4. 宿主数据迁移

新增 `scripts/migrate_sqlite_to_postgres.py`：

1. 读源 SQLite（只读），目标 `create_all` 建表。
2. 按 `Base.metadata.sorted_tables` 外键顺序逐表复制，`INSERT ... ON CONFLICT DO NOTHING`，保留主键。
3. 重置所有自增序列到 `max(id)+1`。
4. 校验：逐表行数、`students`/`conversations`/`learning_tasks`/`pdf_citations` 主键集合 sha256、抽样 JSON 字段逐字节一致。
5. 幂等：重复执行不产生重复行、不改变数据。

源文件不删除、不修改，作为回滚底本。`memory_documents_fts*` 影子表不迁移，迁移后重建索引。

## 5. LightRAG 硬切设计

### 5.1 存储绑定

`knowledge/lightrag/engine.build_rag()` 固定传：

```python
kv_storage="PGKVStorage",
vector_storage="PGVectorStorage",
graph_storage="PGTableGraphStorage",
doc_status_storage="PGDocStatusStorage",
```

不新增 `LIGHTRAG_STORAGE` 开关。`PGTableGraphStorage` 用普通表存图，不依赖 Apache AGE；`PGVectorStorage` 初始化时自动 `CREATE EXTENSION IF NOT EXISTS vector`。

workspace 沿用现有 `deeptutor_426733019d24d665`（从 `meta.json` 读取），保证导入后所有 id 与引用身份一致。

### 5.2 文件依赖改走 PG

| 位置 | 硬切后 |
|---|---|
| `knowledge/fixed_book.py:193` 直读 `vdb_entities.json` | 初始化后从 PG 读取实体词表 |
| `knowledge/lightrag/retrieval.py:160` 校验 `index_files.json` | 校验 `lightrag_index_manifest` 表：各 namespace 行数 + id 集合 sha256 + 源 candidate 签名 |
| `knowledge/lightrag/storage.py` 用 `kv_store_doc_status.json`/`vdb_*.json` 判断就绪 | 查 `LIGHTRAG_DOC_STATUS` 是否有成功记录 |
| `knowledge/lightrag/pipeline.py:204` 清点 `vdb_*`/`graphml` 文件 | 清点 PG 各表行数并写入 manifest |
| `knowledge/lightrag/cache_reuse.py` 读 `kv_store_llm_response_cache.json` | 读 `LIGHTRAG_LLM_CACHE` |
| `graph_viewer.py` 解析 graphml + chunks JSON | 查询 PG，页面与接口不变 |
| `scripts/check_lightrag_completion.py` | 查 PG 三类表 |

仍留在磁盘的内容：`input/` 教材原文、`input_manifest.json`、`source_map.json`、`policy.json`、`meta.json`（release 身份）、`__parsed__/` sidecar（drawings/tables/equations）。这些是教材素材与发布身份，不是可变存储。

### 5.3 一次性导入

新增 `scripts/import_lightrag_candidate_to_pg.py`，不调用 LLM：

1. 用 LightRAG 存储类建 11 张表；workspace 取 candidate `meta.json`；用现有 `HostEmbedding`（512 维）初始化，保证 `VECTOR(dimension)` 与源向量一致。
2. KV：`kv_store_full_docs/full_entities/full_relations/text_chunks/entity_chunks/relation_chunks/llm_response_cache/doc_status` 按 namespace upsert。
3. 向量：解码 `vdb_chunks`/`vdb_entities`/`vdb_relationships` 的 NanoVectorDB base64 float32 矩阵，按原 id、原向量写入。
4. 图：graphml 节点/边 → `upsert_node`/`upsert_edge`。
5. 在 `heart_rag` 写 `lightrag_index_manifest`：每个 namespace 的行数、id 集合 sha256、源 `index_files.json` 签名、导入时间。
6. 幂等：按 `(workspace, id)` upsert，重复执行不产生重复行。

### 5.4 对照门禁（删文件分支前必须通过）

固定题集 `tests/data/native_lightrag_latency_cases.json`（10 题，覆盖定义、追问、跨章、选区、图题、测验、研究、无证据、重复读取、长历史）用于检索层对照：

- `mode=naive`：只比 chunk 命中集合，纯向量、无 LLM 关键词，结果可重复。
- `mode=hybrid`：固定 `hl_keywords`/`ll_keywords`（用现有 `ResidentGraph.keywords` 规则产生），比 chunk/entity/relationship 三类命中 id 集合。
- 通过条件：`chunk_id`、`document_id`、引文范围集合完全一致；排序差异允许。
- 不通过则停在文件后端，不切 `.env`、不删分支。

同时核对：知识图谱节点/边数量、doc_status 行数与 `chunks_list`、实体词表条目数。

## 6. 测试策略（PG-only）

新增 `tests/pg_support.py`：

- 读 `TEST_POSTGRES_URL`（默认指向 `postgres` 维护库），为每次测试创建 `heart_test_<uuid>`。
- 设置 `DATABASE_URL`、`COURSE_DATABASE_URL`、`POSTGRES_DATABASE`，用完 drop。
- LightRAG 夹具另建 `heart_rag_test_<uuid>`，避免与业务库互相清理。

改造范围：

- 约 15 个硬编码 `sqlite:///` 的宿主业务库测试改走 `pg_support`。
- 约 6 个 LightRAG 文件夹具测试（`test_lightrag_backend`、`test_lightrag_relocation`、`test_graph_viewer`、`test_fixed_book`、`test_native_course_kb_migration`、`test_output_speed` 的相关用例）改 PG 夹具或改为纯函数测试。
- vendor 的 SQLite 测试（`chat_history.db`、practice store、session store）不动。
- mock 了 `build_rag` 的测试保持不动。

`tests` 下所有用例执行前必须先有可用 PG；这是 PG-only 的直接代价，写进 README 的"本地开发"一节。

## 7. 验收标准

1. 三个库建立完成，`pgvector` 可用；`SELECT extversion FROM pg_extension WHERE extname='vector'` 返回 0.8.7。
2. 宿主数据迁移校验全绿；登录、会话历史、学习任务、Q5 路径、引用记录在页面上可见。
3. `/health` 的 `fixed_book.ready=true`；启动无文件存储读取。
4. 真实教材问题 3 个、划选 1 个、图题 1 个、研究 1 个：回答带合法引用，引用可打开原 PDF 坐标。
5. 检索对照门禁通过（5.4）。
6. `pg_dump` 三库均可成功备份，恢复到空实例后服务可启动。
7. 记录启动耗时与单题耗时；仅记录，不作迁移门槛。

## 8. 回滚

顺序执行：

1. 代码回退到硬切前提交（文件存储分支仍在）。
2. 恢复原 `.env`（SQLite 连接串）。
3. 业务库直接用原 SQLite 文件；LightRAG 用原 candidate 目录。
4. PG 三个库保留不删，作为对照资料。

因为"删文件存储分支"是最后一步，迁移、导入、门禁、切库期间任何失败都可以停在旧代码上。

## 9. 风险与限制

| 风险 | 处理 |
|---|---|
| pgvector 在 PG18 上编译失败 | 先做一次最小编译验证；失败则改用 Docker 的 `pgvector/pgvector:pg18`，连接串与代码不变 |
| LightRAG 导入后检索结果不一致 | 5.4 对照门禁阻断切换 |
| 全局 advisory lock 限制写入并发 | 当前规模可接受；记录为可升级点，不在本轮优化 |
| PG-only 后本地测试依赖 PG 服务 | `pg_support` 统一建库/清理；README 写明前置条件 |
| `llm_response_cache` 24MB 导入耗时 | 单独计时；失败不阻断导入，可从空缓存重建 |
| 文件分支删除后无法用同一份代码 A/B | 对照门禁必须在删除前完成，且保留文件后端 commit |

## 10. 非目标

- LangGraph 接入（另有独立 spec）。
- vendor DeepTutor 的每用户 SQLite（`chat_history.db`、practice/learning store）迁移。
- `knowledge/rag_index.sqlite3` 本地教材向量索引（正在被 LightRAG 取代，本次不动）。
- JSON → JSONB、Alembic、连接池压测、远程 ECS 部署。
