# 后勤售前数字员工技术实施规范 V1.0

> 状态：待实施的编码规格，不代表功能或接口已经实现。  
> 日期：2026 年 10 月 9 日。  
> 业务基线：[后勤售前数字员工系统设计方案 V3.1](./logistics-presales-ai-system-design-v3.1.zh-CN.md)。  
> 基线提交：`b24c786e83b4c02fc8d97f3af8d6e5a152b10878`。  
> 适用仓库：`zhangks93/presables`。

## 0. 使用方式与约束层级

本文件把 V3.1 转换为可交给 Codex 连续执行的工程契约。实现时先阅读本文件与 V3.1，再依照任务包完成代码、迁移、界面、测试和运行说明。本文中的命令、目录及 API 是要求实现的目标，不是对仓库现状的描述。

约束优先级：用户后续明确要求 → V3.1 的业务范围与责任边界 → 本文已定技术决策 → 任务包中的实现细节。本文未给定真实业务数据、财务公式、飞书模板或企业授权；缺失项必须通过明确的依赖阻塞处理，不能自行推断为已就绪。

首期只实现 ST1–ST4；ST5 仅为人工标识。不得实现投资自动决策、正式《合作建议书》、合同、投标、客户自动发送、多机器人互相审批、通用插件市场和多因素财务归因。不要重新引入 V3.0 的多套存储副本或必须维护的审计哈希链。

“完成”包括可运行代码、真实数据库迁移、指定正反测试、实际执行结果和文档；只有接口签名、TODO 或演示假结果不算完成。技术验收与真实业务集成验收分别报告。

## 1. 已定技术基线

| 项目 | 固定选择 | 实现规则 |
| --- | --- | --- |
| 运行时 | Python 3.12 | `pyproject.toml` 声明 `>=3.12,<3.13`；依赖由 `uv.lock` 锁定 |
| HTTP 与契约 | FastAPI、Pydantic v2、Uvicorn | OpenAPI 从代码生成；API 不复制业务规则 |
| 持久化 | PostgreSQL 16、SQLAlchemy 2、psycopg 3、Alembic | 开发、测试、线上同一数据库类型；首版使用同步数据库 Session |
| 页面 | Jinja2、原生 JavaScript、本地 CSS | 同源服务；不依赖 React、Node 构建、公共 CDN 或前端状态库 |
| 后台执行 | PostgreSQL jobs/outbox + 独立 worker | 无 Redis、Celery、Kafka；数据库短事务、外部调用在事务外执行 |
| 文件 | 本地受控文件存储端口，生产换企业对象存储 | 业务表只保存受控引用与哈希，不保存公开下载 URL |
| 材料解析 | pypdf、python-docx、openpyxl | 只解析；扫描 PDF/OCR 单独设能力开关，缺能力返回待处理原因 |
| 模板成果 | Jinja2 内容模板 + python-docx | 首期正式导出 DOCX；PDF 按需启用；保留 Markdown 草稿 |
| Excel 执行 | LibreOffice headless 能力验证，openpyxl 负责输入输出读写 | 必须真实重算；生产模型兼容性和标准答案通过后才可启用 |
| LLM | 一个 `LLMGateway` 端口 + 一个配置化的兼容接口适配器 | 供应商、模型、数据使用范围是实际依赖；不在业务模块直接调用 SDK |
| 测试与质量 | pytest、Ruff、mypy、HTTPX TestClient | 数据库/并发测试使用 PostgreSQL；开发依赖与运行依赖分组 |

Python 依赖先约束上述主版本，首次安装选择兼容版本并提交锁文件；后续 CI 使用 `uv sync --frozen`。不得假定未经验证的最新版本或接口可用。LibreOffice/系统包版本在实际构建后记录，生产执行使用验证过的镜像摘要。

同步 HTTP handler 使用同步 `def` 与同步 Session；worker 使用独立进程。禁止在事件循环内直接运行同步数据库、文件转换或计算。领域函数应尽可能为无 I/O 的纯函数，方便验证状态守卫。

### 1.1 模块边界

| 模块 | 职责 | 允许的依赖 |
| --- | --- | --- |
| `domain` | 类型、单位、状态、权限与业务守卫 | Python 标准库与 Pydantic；不得导入 FastAPI/外部 SDK |
| `application` | 用例、事务、幂等、审计、后台命令完成 | domain、ports；通过 UnitOfWork 提交 |
| `ports` | 数据、文件、LLM、审批、通知、计算、身份协议 | 领域类型 |
| `adapters` | SQL、文件、飞书、LLM 和计算的实际实现 | ports、所需 SDK/客户端 |
| `api` / `web` | 身份解析、参数验证、HTTP/页面输出 | application；不直接修改 ORM 对象 |
| `worker` | 领取任务、心跳、调用执行器、带 fencing token 回写 | application、ports |

模块化单体指共用代码和业务模型，不要求所有工作在同一进程中运行。API、业务 worker、无网络计算执行器使用同一代码版本的独立进程或容器。

### 1.2 外部端口的最小方法

下列名称是本系统内部协议，不能声称是 WorkBuddy 或飞书实际 SDK 方法。方法使用严格类型的输入/输出，不传递不受约束的业务字典。

| 端口 | 必需方法与行为 |
| --- | --- |
| `FileStore` | `put_immutable(stream, project, expected_hash)`、`open(asset)`、`exists_and_verify(ref)`；调用前完成授权 |
| `LLMGateway` | `generate(request, output_schema, budget)`；返回文本、模型/提示词版本、用量和结构化错误 |
| `ApprovalGateway` | `submit(frozen_application, operation_key)`、`fetch_authoritative_state(ref)`、`verify_callback(headers, raw_body)` |
| `ProjectionGateway` | `upsert_project_projection(project_id, revision, summary)`；防旧版本覆盖由适配器和投影记录共同保证 |
| `NotificationGateway` | `send_internal(recipient, kind, reference, dedupe_key)`；只允许同租户内部已授权接收人 |
| `IdentityGateway` | `begin_login(state)`、`exchange_code(code)`、`refresh_if_needed(identity_ref)`；真实协议按配置验证 |
| `CalcEngine` | `validate_model(model)`、`execute(request, cancellation)`；返回真实工作簿、结果和执行日志 |

未配置的真实适配器抛出 `DependencyBlocked(dependency_id, reason)`。它不能返回空成功、模拟审批通过或假的正式计算结果。测试替身放在 `tests/fakes/`，脱敏离线演示适配器仅允许在 `APP_ENV=local/test` 使用。

## 2. 工程布局、环境与启动配置

统一采用 Python 3.12、FastAPI、Pydantic 2、SQLAlchemy 2、psycopg 3、Alembic、PostgreSQL 16、uv、pytest、Ruff 和 mypy。前端使用 Jinja2、局部表单和仓库内 JavaScript，不引入独立 SPA 工程。API 与 PostgreSQL worker 属于同一模块化单体；不引入 Redis、Celery、SQLite、向量数据库或通用插件平台。版本补丁号在首包选定兼容版本并提交 `uv.lock`，后续构建使用锁文件。

拟建目录如下；发现仓库已有同等职责目录时，在不破坏现有代码的前提下合并，并记录映射。

```text
docs/                         # 设计、技术方案、实施记录、真实依赖清单
src/presales/
  app.py                      # 应用工厂，/healthz 与 /readyz
  settings.py                 # 配置验证，禁止输出秘密
  db/                         # models、session、事务辅助
  api/                        # REST 路由与错误映射
  web/                        # 同源页面与表单路由
  domain/                     # 类型、状态、单位、权限与纯业务守卫
  application/                # 用例、事务、幂等、项目/事实/文档/流程
  ports/                      # Repository、FileStore、LLM、审批、计算协议
  adapters/                   # identity、llm、storage、feishu、calculator
  worker/                     # 领取、租约、心跳、恢复、outbox
  calc_runner/                # 独立系统Python/UNO执行器，仅接收固定协议
  templates/                  # Jinja2 页面与文档模板
  static/                     # 本地脚本、样式
  cli.py                      # 配置预检、演示、验收入口
config/{prompts,fields,rules,templates,models}/ # 受管版本配置
contracts/                    # 导出的OpenAPI/JSON Schema，禁止另维护一套语义
evals/                        # 脱敏固定评测集与结果工具
alembic/versions/              # 逐包增量迁移
tests/{unit,integration,acceptance}/
tests/fixtures/               # 脱敏材料、DEMO模型、预期结果、失败样例
scripts/                      # 备份恢复、检查及报告生成
artifacts/                    # 运行产物，gitignore
compose.yaml
Dockerfile
Dockerfile.calc-runner
Makefile
.env.example
pyproject.toml
uv.lock
```

环境变量的值在 `.env.example` 中只能是安全示例。必须区分应用连接、迁移连接和测试连接，测试不得清空开发或生产数据库。

| 变量 | 契约 |
| --- | --- |
| `APP_ENV` | `local/test/production`；生产启用严格启动校验 |
| `DATABASE_URL`、`MIGRATION_DATABASE_URL` | PostgreSQL 连接；应用账号无迁移权限，部署时单独使用迁移账号 |
| `TEST_DATABASE_URL` | 专用测试库；数据库名须以 `_test` 结尾，执行 destructive fixture 前再校验 |
| `STORAGE_ROOT` | 非静态公开目录；本地为 `./artifacts/storage`，生产挂载持久卷 |
| `AUTH_MODE`、`SESSION_SECRET` | 本地 `dev`、生产 `feishu`；开发身份入口必须在生产拒绝启动 |
| `PUBLIC_BASE_URL`、`ALLOWED_HOSTS` | 回调与同源校验地址，不得直接信任请求 Host 生成敏感链接 |
| `LLM_PROVIDER`、`LLM_BASE_URL`、`LLM_MODEL`、`LLM_API_KEY` | 本地默认 `fixture`；真实模式须明确模型及允许的端点 |
| `FEISHU_APP_ID`、`FEISHU_APP_SECRET`、`FEISHU_VERIFICATION_TOKEN`、`FEISHU_ENCRYPT_KEY` | 按实际接入方式校验；缺失时相应连接器禁用，不能伪造凭据 |
| `FEISHU_APPROVAL_CODE`、`FEISHU_BITABLE_APP_TOKEN`、`FEISHU_BITABLE_TABLE_ID` | 实际审批及索引配置；本地模拟器不要求提供 |
| `JOB_LEASE_SECONDS`、`JOB_MAX_ATTEMPTS` | 默认 60 秒、5 次；执行器须按租约规则心跳与幂等 |
| `UPLOAD_MAX_BYTES`、`TASK_TIMEOUT_SECONDS` | 默认 20 MiB、300 秒；按解析与计算任务分别配置覆盖 |

`production` 禁止开发身份、LLM fixture 和模拟审批生效。生产未配置真实 LLM 时设置 `LLM_PROVIDER=disabled`，关闭AI能力并返回 `LLM_CONFIGURATION_MISSING`；人工录入仍可用。未启用 AI/审批的业务可以继续使用；被禁用的功能返回明确依赖错误。正式测算模型是否可用依据模型登记和批准记录判断，不能仅依赖一个环境变量。

配置补充：`JOB_HEARTBEAT_SECONDS=15`、`CALC_TIMEOUT_SECONDS=120`、`CALC_SPOOL_ROOT=/spool`、`MODEL_REGISTRY_ROOT=/registry`、`LLM_ENABLED=false`（本地演示可启用 fixture）。SESSION_SECRET 至少32随机字节，本地首次生成写入gitignored的.env且权限0600，不打印秘密；既有配置不覆盖。STORAGE_ROOT、calc-spool及注册表不可位于静态文件目录。上传与解压限制见材料章节。

从 P04 起 Compose 必须包含 `calc-runner`，并配置 calc-spool 共享卷、只读模型注册表、无网络与资源限制。API/worker不挂Docker socket。runner建议使用Ubuntu 24.04系统Python 3.12、libreoffice-calc和python3-uno，UNO脚本由匹配的系统Python运行；不要将不同ABI的pyuno直接复制进应用虚拟环境。runner协议校验可用标准库并复用schema测试。基础镜像和系统包版本实际构建后锁定，不在文档伪造镜像摘要。

API本地只映射127.0.0.1:8000，数据库不向公网映射。db健康→migrate成功→api/worker启动；calc-runner不可用仅阻断计算能力，不阻断ST1/ST2。首次P01建仓时先执行 `uv lock` 并提交锁文件，然后bootstrap/CI使用frozen；`make bootstrap`不以运行时自动升级依赖代替锁文件。

## 3. 数据模型、事实审核与事务契约

本节为已定的编码契约，不改变 ST1–ST4 业务范围。采用 PostgreSQL 16、SQLAlchemy 2 和 Alembic；开发与测试使用相同数据库。

### 3.1 标识、边界及最小表结构

主键使用服务端生成的 UUID，时间使用 `timestamptz`、保存 UTC。`tenant_id` 从认证会话获得，禁止客户端请求体指定；首期只配置一个组织，但所有业务访问仍按租户过滤。项目子表带 `tenant_id, project_id`；项目表建立 `UNIQUE(tenant_id,id)`，所有子表使用组合外键，禁止只凭对象 UUID 关联。默认不物理删除事实、审核和成果历史。

下表省略公共主键和时间列；`UQ` 表示必须建立唯一约束，角色和状态使用文本列及 SQL `CHECK`，便于迁移。

| 表 | 必须列及约束 |
|---|---|
| `tenants/users/tenant_memberships` | 组织、账号及组织关系；外部账号 `UQ(provider,external_subject)`；成员 `UQ(tenant_id,user_id)`，`active` |
| `projects` | `name,owner_id,stage,stage_version,status,aggregate_version`；阶段 ST1–ST5，状态 ACTIVE/PAUSED/CLOSED，聚合版本初始 0，stage_version 初始 1 |
| `project_memberships` | `user_id,roles text[],active`；`UQ(tenant_id,project_id,user_id)`；角色 OWNER/SALES/PRESALES/EXPERT/LEADER/VIEWER |
| `project_scopes` | `scope_key,name,kind`；`UQ(tenant_id,project_id,scope_key)`；初始 `project`；校区、服务范围由登记记录生成稳定键 |
| `project_periods` | `period_key,kind,start_date,end_date`；同项目键唯一；`all` 表示无期间，年度、学年须明确起止日期 |
| `field_definitions` | `field_key,definition_version,data_type,canonical_unit,is_critical,allowed_period_kinds,confirm_roles,scale,min_value,max_value,requires_exact`；`UQ(tenant_id,field_key,definition_version)`；版本不可改 |
| `materials` | `original_name,file_id,uploaded_by,parse_status`；file_id关联stored_files；相同文件名不得覆盖；SHA不作业务唯一键 |
| `source_fragments` | `material_id,parser_version,locator,extracted_text,content_hash`；定位为 PDF 页码、DOCX 段落或 XLSX 工作表/单元格；原始提取内容不可改 |
| `candidates` | `field_key,scope_key,period_key,definition_id,payload,content_hash,status,supersedes_id`；正文不可改；状态 PENDING/ACCEPTED/REJECTED/SUPERSEDED |
| `candidate_evidence` | `candidate_id,fragment_id,quote,location_check,check_details`；定位状态 PENDING/PASSED/FAILED，检查结果只能由后端写入 |
| `fact_heads` | 字段联合键、`revision_no default 0,current_revision_id nullable`；联合键 `UQ(tenant_id,project_id,field_key,scope_key,period_key)` |
| `fact_revisions` | `head_id,revision_no,candidate_id,definition_id,payload,confirmed_by,confirmed_at,reason`；`UQ(head_id,revision_no)`，候选采纳版本唯一；正文不可变 |
| `fact_conflicts` | `head_id,candidate_id,against_revision_id,status,version,resolved_by,reason`；OPEN/RESOLVED；冲突解决需人工选择或解释，不因接受另一候选自动消失 |
| `review_sessions` | `kind,snapshot_id,reviewer_id,snapshot_hash,status,version`；OPEN/DECIDED/CANCELLED，version 初始 1；一个审核会话由指定审核人作出决定 |
| `review_items` | `session_id,candidate_id,candidate_hash,head_id,expected_revision_no`；会话内候选唯一，条目冻结后不可改 |
| `review_decisions` | `session_id,item_id,decision,reason,acknowledgements,actor_id`；ACCEPT/REJECT；`UQ(session_id,item_id)`，追加保存 |
| `snapshots/dependency_refs` | 快照 `kind,manifest_hash,created_by`；Review/ArtifactRevision/CalcRun 各引用 `snapshot_id`；依赖统一挂在快照下，禁止各模块另建重复依赖表 |
| `idempotency_records` | `tenant_id,actor_id,project_id,command,key,request_hash,http_status,response_json`；`UNIQUE NULLS NOT DISTINCT(tenant_id,actor_id,project_id,command,key)`；使用PENDING/COMPLETED状态；PENDING只存在于未提交事务内，事务退出前必须填充最终响应 |
| `audit_events/outbox_events` | 相同 `command_id` 关联业务修改；审计保存动作、操作者、对象、旧新版本编号；出站事件保存 `event_type,aggregate_version,payload,status` |

`dependency_refs` 设置 `snapshot_id` 外键，并用互斥目标列引用事实头、模型版本、模板版本、知识版本、材料或计算运行；SQL `CHECK(num_nonnulls(fact_head_id,model_version_id,template_version_id,knowledge_version_id,material_id,calc_run_id)=1)`。每个目标有租户（项目资源还含项目）的真实组合外键及反向索引，禁止只用无外键的多态 ID。事实依赖另存 `expected_revision_no,expected_revision_id`，预期 0 时后者 NULL；同快照同目标唯一。快照 JSON 清单是不可变展示与哈希副本，依赖表是检索和新鲜度判断依据，两者同事务创建。

事实当前值只存在 `fact_revisions.payload`；`fact_heads` 仅保存指针。头指针建立 `(tenant_id,project_id,id,revision_no,current_revision_id)` 到版本表 `(tenant_id,project_id,head_id,revision_no,id)` 的组合外键；被引用列组加 UNIQUE，约束 `DEFERRABLE INITIALLY DEFERRED`，保证头、版本号和版本 ID 一致；版本号 0 当且仅当当前指针为 NULL。来源、候选、事实及审核之间同样使用租户/项目组合外键。应用数据库账号禁止更新、删除版本正文和审计；候选状态可变，正文通过触发器禁止修改。迁移账号单独配置。

### 3.2 事实载荷与来源核验

字段身份为 `(project_id,field_key,scope_key,period_key)`。`scope_key/period_key` 禁止 NULL，禁止用展示名称代替稳定键；写入前必须匹配登记表和字段定义。字段类型和单位不因某次 AI 输出而改变。

```json
{
  "state": "KNOWN",
  "data_type": "integer",
  "value": "1200",
  "unit": "person",
  "basis": "SOURCE",
  "knowledge_version_id": null,
  "assumption_context": null,
  "source_note": null
}
```

`state` 为 KNOWN/UNKNOWN/NOT_APPLICABLE；后两者 `value` 必须为 JSON null 且说明原因，KNOWN 必须非 null。KNOWN状态的`basis`为SOURCE/STANDARD/ASSUMPTION；UNKNOWN/NOT_APPLICABLE的basis=null并要求reason，可保留解释来源，但不伪造已知数值的证据；STANDARD 必须引用已批准且范围、期间适用的知识版本；ASSUMPTION 必须记录批准使用的情景和理由。SOURCE 必须关联定位检查通过的证据，或有人工录入的署名来源说明；后者显示「人工提供」，不能显示「文档定位通过」。未知值可以被记录，但不能成为必填测算输入。NOT_APPLICABLE 只有模型映射显式支持时可使用。零必须是 KNOWN。

候选和事实版本共用以下数据库约束；字段类型、单位和标准适用性仍由服务层校验，不能用 SQL 约束替代它们：

```sql
CHECK (jsonb_typeof(payload) = 'object'),
CHECK (payload ?& ARRAY['state','data_type','value','basis']),
CHECK (jsonb_typeof(payload->'state') = 'string'
       AND jsonb_typeof(payload->'data_type') = 'string'
       AND jsonb_typeof(payload->'basis') = 'string'),
CHECK ((payload->>'state') IN ('KNOWN','UNKNOWN','NOT_APPLICABLE')),
CHECK ((payload->>'basis') IN ('SOURCE','STANDARD','ASSUMPTION')),
CHECK ((payload->>'state' = 'KNOWN') = (payload->'value' <> 'null'::jsonb)),
CHECK (payload->>'data_type' <> 'decimal' OR payload->>'state' <> 'KNOWN'
       OR jsonb_typeof(payload->'value') = 'string')
```

payload 列另设 NOT NULL；CHECK 对 SQL NULL 放行，因此对象、键存在和类型约束须完整实现，不能仅保留枚举检查。

所有 Decimal 在 API、JSONB、任务载荷、快照和报告数据中均编码为十进制字符串；禁止科学计数法、NaN、Infinity，禁止经 float 转换。Pydantic接收字符串后构建Decimal；此约束适用于领域与接口，Excel数值边界按第8章有限精度契约处理；按字段定义验证量纲、精度和范围。整数计数也使用十进制字符串并限制小数位为0；data_type为integer。UNKNOWN/NOT_APPLICABLE必须保存reason，候选另存qualifier；KNOWN仅描述候选值已给出，是否正式采纳由审核决定。浏览器仅显示，数值计算与比较交给后端。载荷哈希使用字段定义规定的十进制规范形式和排序后的 UTF-8 JSON，禁止依赖 JSONB 输出顺序。

`location_check=PASSED` 仅表示原片段存在且引用匹配；UI 固定文案为「来源定位通过」，不能表示事实真实。审核页必须并排显示旧值、候选值、原文、单位、范围、期间和依据。审核哈希覆盖候选、旧事实版本、证据摘录和定位结果的冻结展示清单，不能仅哈希候选编号。关键字段逐项勾选 `value/scope/period/unit/basis` 确认，不提供默认勾选或一键全选；数字及口径的最终判断由审核人作出。人工改值产生新候选并重新展示，不能在提交确认时偷偷替换被冻结候选。

关键字段的新候选与同键当前事实不一致时，程序创建 OPEN 冲突；不同范围或期间不互相冲突。审核清单同时冻结该字段的开放冲突编号。resolve_conflict_ids 仅能包含本会话展示过且属于同字段的冲突，逐项保存选择和理由。

### 3.3 权限与 HTTP 命令

认证生成 `Principal(tenant_id,user_id)`，每个服务方法必须接收它，不允许 Repository 绕过组织过滤。无项目访问权限返回 404；已有项目访问权但缺少动作权限返回 403。下载、历史记录、任务、检索和后台执行适用相同规则。组织管理员仅管理配置和成员，除非明确获得项目角色，否则无业务权限。

| 角色 | 默认动作 |
|---|---|
| VIEWER | 查看授权项目、历史和成果 |
| SALES/PRESALES | 上传材料、记录拜访、提交候选、创建草稿；确认字段仍须匹配 `confirm_roles` |
| EXPERT | 确认指定专业字段、复核指定测算、审核文档 |
| OWNER | 管理项目成员、分配任务、改变阶段、发布成果；不能替代模型指定的专业复核 |
| LEADER | 查看项目及其审批事项；内部角色不能伪造外部审批结果 |

除登录与经供应商事件ID去重的webhook外，所有写接口要求 `Idempotency-Key`；已有实体动作要求 body.expected_version（只检查该对象状态版本），不能用项目聚合版本代替字段版本；JSON 使用 Pydantic `extra='forbid'`。接口前缀 `/api/v1`，表中以/projects开头的路径直接附于/api/v1，其他项目资源路径附于/api/v1/projects/{project_id}。

| 方法及路径 | 请求与结果 |
|---|---|
| `POST /projects` | `{name,region}` → 201 项目；创建者成为 OWNER |
| `GET /projects/{id}` | 项目摘要、角色、当前待办；无副作用 |
| `POST /stage-transitions` | `{expected_version,to_stage,reason}`，版本指 stage_version → 200；仅 OWNER，可人工回退并记录理由 |
| `POST /materials` | multipart 文件及用途 → 202 `{material_id,job_id}`；文件存储成功后登记 |
| `GET /facts` | 按 scope/period 过滤，返回当前版本及定位引用 |
| `POST /candidates` | 字段键、载荷和证据 → 201；人工候选同样进入审核 |
| `POST /fact-reviews` | `{candidate_ids}` → 201 冻结审核会话、逐项旧新值、会话 version 和 `snapshot_hash` |
| `POST /fact-reviews/{id}/decisions` | 见下例 → 200 事实版本和决定；一个会话的完整决定原子提交 |
| `GET /facts/{head_id}/history` | 不可变历史及确认依据，游标分页 |
| `POST /conflicts/{id}/resolve` | `{expected_version,expected_revision_no,resolution,reason,review_decision_id}`；resolution为KEEP_CURRENT/ACCEPT_REVIEWED，选择新值须关联已授权事实审核决定，同事务检查事实头版本 |

```json
{
  "expected_version": 1,
  "snapshot_hash": "sha256-of-review-manifest",
  "items": [{
    "item_id": "uuid",
    "decision": "ACCEPT",
    "acknowledgements": ["value", "scope", "period", "unit", "basis"],
    "reason": "与现场负责人核实",
    "resolve_conflict_ids": []
  }]
}
```

返回错误统一为 `{"error":{"code":"FACT_VERSION_CONFLICT","message":"该字段已更新","details":{},"retryable":false},"request_id":"..."}`，列表统一 `{items,next_cursor}`。422 用于类型/口径错误，409 用于版本冲突、业务门槛、幂等键内容不一致；503 用于依赖暂不可用。异步命令返回 202 和任务编号，不能提前返回业务成功。关键错误码固定为 `FACT_VERSION_CONFLICT`、`REVIEW_CONTENT_CHANGED`、`DEPENDENCY_CHANGED`、`IDEMPOTENCY_KEY_REUSED`、`OPEN_CRITICAL_CONFLICT`。

### 3.4 采纳、幂等与依赖检查算法

事实采纳及其他已有项目命令使用一个 `READ COMMITTED` 事务，严格执行以下顺序：

创建项目没有现成项目行，其幂等键使用 project_id=NULL，并在同一事务创建项目和 OWNER 成员。已有项目命令执行（上传请求哈希使用规范metadata与文件SHA-256，不能哈希每次不同的multipart边界）：

1. 按认证租户定位并 `SELECT projects ... FOR UPDATE`，涉及job时随后锁job并检查fencing，所有项目修改、快照冻结、审核和发布统一先取得该项目行锁。随后重新检查组织、项目及字段确认权限，锁住有效成员行（共享锁）；成员撤销事务也遵循此锁顺序。项目行仅在短数据库事务内持有，事务中禁止模型调用、文件 IO、外部请求和计算。
2. 按 `(tenant,actor,project,command,key)` 尝试插入幂等记录；命中记录则比较规范请求哈希。相同且仍有权限时重放原状态码和结果，不同返回 409。唯一键冲突等待原事务提交；不得向尚未授权的人重放旧响应。
3. 锁定审核会话并验证 reviewer、OPEN 状态、expected_version、snapshot_hash 和所有条目。会话创建时已为涉及的字段执行 `INSERT fact_heads ... ON CONFLICT DO NOTHING`，即使该字段从未有值，也有可锁行。
4. 按固定顺序锁候选、字段头及依赖资源；字段头按字段联合键排序 `SELECT fact_heads ... FOR UPDATE`，逐项比较 `expected_revision_no`；首次采纳的预期为 0。第二个并发首次写入者获得锁后将看到版本 1 并失败。任一条目不一致时整批返回 409，不产生部分采纳。
5. 验证候选仍为 PENDING、内容未变、证据可访问、标准仍可用、逐项确认完整；ACCEPT 追加版本 n+1，移动头指针；REJECT 仅写决定及理由。请求可通过 resolve_conflict_ids 明确解决已展示的冲突，要求理由并绑定本次版本/拒绝决定；未明确解决的冲突保持 OPEN。
6. 保存逐项决定，关闭会话并递增其 version；项目 `aggregate_version=aggregate_version+1` 获取本次顺序号；写入审计及出站事件，保存幂等响应，统一提交。数据库回滚时不留下成功幂等记录。

批量审核请求不允许省略会话条目；用户想部分处理时创建较小的新会话。不同字段只检查自身头版本，项目顺序号不能作为全项目事实 CAS 条件。不同字段会短暂等待同一项目事务，但不会因无关字段变更返回 409。所有涉及业务的事务遵循「项目→涉及的job→成员→审核/候选/业务对象→字段头→共享依赖」顺序，死锁可由服务针对同一幂等键有限重试。

测算、文档和审批分别冻结自己的依赖集合，不能只存 `project_version`。快照引用不可变事实版本，不复制可变当前值；还应记录实际使用的模板、规则、知识、计算运行及业务材料版本。模板/模型声明的必需字段，即使缺失，也以版本 0 进入依赖集合，避免之后新增值未被发现。快照构建在一致性事务中完成并计算清单哈希。

审核和发布先取得项目行锁，再按固定顺序锁定相关事实头，比较快照中的版本，并在同一事务中完成结果登记；这样不会发生「检查通过后、正式登记前被改值」的竞态。标准撤销、模型停用和模板更换由对应模块在事务内检查；无关字段变更不影响当前审核。依赖变化返回 409，保留原草稿及审核历史；重新生成/审核使用新快照，禁止仅刷新旧快照哈希来绕过审核。相关字段存在未解决的关键冲突时阻止正式使用。

### 3.5 必须自动化验证的数据用例

使用真实 PostgreSQL 集成测试及并发连接，至少验证：跨租户/跨项目关联被拒绝；未知、零、不适用、假设区别；Decimal 无浮点损失；候选篡改无法采纳；关键字段逐项确认；两人同时首次写同字段只成功一次；不同字段并发均成功；幂等重放仅一个事实版本且异内容 409；撤销权限后无法重放/恢复；来源定位通过不自动采纳；相关依赖变化阻止发布而无关变化不阻止；事务失败时事实、决定、审计、出站事件全部回滚。

### 3.6 跨模块一致性补充

`review_sessions/review_items/review_decisions`首期只处理FACTS审核。文档、参数、结果与尽调使用各自的追加式决定表，统一审核页面不等于把不同决定硬塞进事实candidate外键。事实审核会话没有明确接受时保持OPEN；拒绝不改变事实。KEEP_EXISTING通过拒绝新候选并明确解决所展示冲突表达。

数据类型固定为integer/decimal/text/date/boolean；数值限制与单位转换表由field_definitions版本确定，CNY/万元等转换必须来自批准的单位表。标准有效期按业务期间和确认时间校验，不能仅按上传时间。首次创建项目需有效组织成员及project.create授权；幂等记录的唯一键竞争与项目创建、OWNER成员和审计在一个事务完成。

对共享模板/模型/知识，项目业务事务在项目和本地资源锁之后按 `(资源类型,资源ID)` 排序取得 `FOR SHARE`，到提交前保持；撤销事务只锁共享资源 `FOR UPDATE`、改变状态并发outbox，不反向锁项目。异步STALE标记用于展示；正式执行/复核/发布现场检查批准状态，因此撤销不会留下可继续正式使用的窗口。先取得锁的事务先完成，后续请求看到撤销状态。

快照清单及依赖表在同一事务冻结；对象内容不可变、状态列可受控变化。模型/模板/知识使用DRAFT/APPROVED/RETIRED，正文变更新建版本，撤销使用RETIRED且不复活原版本；需要重新启用时创建新版本及新验证证据。

首包先建实际需要的基础表；P02加入snapshots与最小knowledge/template版本登记；P04加入模型/运行目标列及其真实外键。每次迁移只引用已经创建的表，不能用悬空UUID或无FK占位绕开尚不存在的目标。依赖表及CHECK随目标类型增量迁移，不一次迁移引用未来表。

## 4. 补充实体、应用命令与 HTTP 接口

本节补齐事实/审核之外的存储与接口。所有项目实体同样携带 `tenant_id/project_id` 组合外键；资源 ID 由服务端生成 UUID。若前文已定义某表，本节为补充列，不能重复创建第二套同义实体。

### 4.1 必须补齐的持久化对象

| 表 | 关键列及数据库约束 |
| --- | --- |
| `stored_files` | `id,tenant_id,project_id,purpose,storage_key,sha256,mime,byte_size,lifecycle,created_by`；对象键唯一；lifecycle 为 STAGING/READY/ORPHANED；外部路径不可由用户指定 |
| `visits` | `id,occurred_at,participants_json,status,version,confirmed_by`；DRAFT/CONFIRMED；`visit_materials` 关联材料，同项目组合外键 |
| `execution_authorizations` | `id,principal_kind,user_id,rule_version_id,allowed_action,subject_type,subject_id,granted_by,active,expires_at`；HUMAN/RULE 二选一；不能赋予规则审批/发布/阶段变更权限 |
| `jobs` | 使用执行章节定义；补 `tenant_id,dedupe_key,input_hash,version,cancel_requested_at,supersedes_job_id`；唯一键为 `(tenant_id,project_id,kind,dedupe_key)` |
| `business_tasks` | `type,title,owner_id,subject_type,subject_id,reason_fingerprint,status,version,blocked_reason,completed_by,completion_ref,due_at`；owner 可空表示待分派；完成须有业务依据 |
| `action_suggestions` | `type,subject_type,subject_id,basis_hash,status,created_by_job_id,decided_by,reason`；PROPOSED/ACCEPTED/REJECTED；同项目同类型同主体同依据唯一 |
| `due_diligences` | 状态见执行章节；补 `version,current_approval_instance_id,materials_snapshot_id,handover_snapshot_id,policy_version_id` |
| `approval_instances` | `dd_id,attempt_no,external_operation_id,external_instance_id,approval_basis_snapshot_id,external_status,basis_validity`；同dd尝试号唯一、供应商实例号唯一；重新提交新建实例，不覆盖旧实例 |
| `approval_slots` | `approval_instance_id,slot_code,required_user_id,delegate_user_id,decision,provider_decision_id,decided_at,evidence_file_id`；同实例席位唯一；非空供应商决定编号同实例唯一 |
| `dd_decisions` | 追加式记录 `dd_id,kind,snapshot_id,actor_id,decision,reason,created_at`；记录材料审核、交接和现场操作，不能覆盖旧决定 |
| `template_versions` | `template_key,version,mode,status,file_id,sha256,required_fields,sections_json,required_review_role,requires_calculation`；DRAFT/APPROVED/RETIRED，正文不可变；同租户模板键与版本唯一 |
| `artifacts` | `artifact_type,instance_key,current_published_revision_id,version`；同项目类型和实例键唯一；拜访实例键必须是 visit_id |
| `artifact_revisions` | `artifact_id,revision_no,snapshot_id,template_version_id,body_file_id,docx_file_id,content_hash,status,freshness,quality_report_id,version`；同 artifact 修订号唯一；状态和时效见执行章节 |
| `artifact_reviews` | `revision_id,content_hash,snapshot_hash,reviewer_id,decision,reason`；追加式；审核只能引用所见文件摘要 |
| `calculation_models` | 每行是不可变模型版本：`model_key,version,mode,status,workbook_file_id,workbook_hash,mapping_json,mapping_hash,engine_id,engine_digest,approved_by,approved_at,golden_report_file_id`；同租户模型键与版本唯一 |
| `calculation_runs` | 执行章节字段；补 `version,job_id,input_confirmation_id,result_manifest_file_id,iteration_no`；模型、模式、scenario、基线创建后不可改；input_status=DRAFT/CONFIRMED/SUPERSEDED，输入确认后不可改，DRAFT编辑产生新快照并递增version；mode 从项目和模型推导 |
| `calculation_confirmations` | `run_id,snapshot_hash,parameter_manifest_hash,actor_id,confirmed_items_json`；不可变；一次确认绑定完整输入清单 |
| `calculation_reviews` | `run_id,result_manifest_hash,snapshot_hash,reviewer_id,decision,reason`；追加式；只能审核已生成结果 |
| `knowledge_versions` | `knowledge_key,version,kind,status,scope_json,effective_from,effective_to,source_file_id,content_hash,approved_by,visibility,project_id_nullable`；RULE/CASE，DRAFT/APPROVED/RETIRED，正文不可变 |
| `rule_versions` | `rule_code,version,enabled,trigger_event,task_type,owner_policy,coalesce_policy,authorization_id`；首期规则由代码实现并通过静态配置启用，不执行用户提交的表达式/脚本 |
| `job_authorization_events` | `job_id,authorization_id,actor_id,reason,created_at`；追加式，保留原授权引用，运行时按最新有效授权并校验当前权限 |
| `visit_revisions` | `visit_id,revision_no,summary,material_refs,confirmed_by,confirmed_at`；确认后修改新建修订，原修订不可覆盖 |
| `calculation_outputs` | `run_id,metric_key,raw_decimal,quantized_decimal,unit,scope_key,period_key`；同运行同指标唯一，成功后不可改 |
| `external_operations` | 执行章节字段；补 `tenant_id,request_file_id,error_code,version`；租户/供应商/kind/business_key 唯一；原始令牌不得进入记录 |
| `projection_states` | `project_id,destination,local_revision,confirmed_remote_revision,remote_record_id,status,last_error,last_checked_at`；项目和目标唯一 |
| `consumed_events` | `consumer,event_id,processed_at`；联合唯一，用于本地消费者去重 |
| `quality_reports/skill_executions/evaluation_runs` | 分别保存文件检查项、AI 调用输入/版本/成本/输出、固定样本集评测结果；引用文件与快照，禁止另建事实来源 |

stored_files增加 `scope=PROJECT/ORGANIZATION`，CHECK确保PROJECT必须有project_id、ORGANIZATION的project_id为空；所有项目文件采用组合外键。`stored_files` 是统一文件元数据；`materials.file_id` 指向它，原材料保留 original_name、uploaded_by、parse_status，不再重复维护一套 storage_key/hash。文件共享仅限同一授权项目；内容相同不意味着自动共享权限。模型/模板/知识的组织级文件用组织资源关系授权，不能借项目文件下载接口绕过。

补齐 `projects.mode=DEMO/PRODUCTION`、`state_version`、`current_verified_calc_run_id`：mode 创建后不可修改；普通在线创建固定 PRODUCTION；DEMO 项目仅由本地/测试种子命令创建。state_version 保护 ACTIVE/PAUSED/CLOSED 的动作，stage_version 保护阶段，aggregate_version 仅排序。生产模式与生产部署是两个概念：测试库可构造 PRODUCTION 模式来测试守卫，但测试记录不得迁移或计入真实上线证据。

有版本状态的实体初始 version=1；项目 aggregate_version 在创建命令后为 1。数据库版本历史、内容哈希和真实来源同样需要保存，不以 version 整数取代证据。

### 4.2 权限的具体补充

项目 OWNER 负责阶段、成员、分配和正式发布；字段采纳按字段定义授权。测算参数确认、结果复核只允许项目指定的 PRESALES/EXPERT，角色匹配且在项目配置的 reviewer 列表中。模板审核人由模板策略确定；OWNER 无权绕过专业复核。尽调申请、材料审核、交接责任人由冻结的尽调策略指定，领导的真实外部决定只由可信适配器导入。

项目只读者可读授权历史、含过期标记的成果；只有“作为当前正式成果使用”需要新鲜度门槛。标准/模型撤销不等于抹除历史下载权限。完整敏感执行日志默认仅项目指定运维/审核者可读，普通界面只显示脱敏错误。

### 4.3 业务接口清单

除登录和外部 webhook 外，写接口均使用前文幂等约定。已有实体动作需 expected_version；后台任务的请求模式和目标角色由后端确认，不从浏览器直接接受。以下 `P` 表示 `/api/v1/projects/{project_id}`，`J` 表示 `/api/v1/jobs/{job_id}`。

| 方法与路径 | 请求核心字段 | 结果及约束 |
| --- | --- | --- |
| `GET /api/v1/projects` | cursor、limit | `{items,next_cursor}`，默认50，最大100；只返回参与项目 |
| `POST P/status` | expected_version、status、reason | 检查 state_version；暂停不抹去历史，阻止新业务副作用 |
| `POST P/visits` | occurred_at、participants、material_ids | 201 DRAFT 拜访；发生时间由用户提供，不能使用上传时间冒充 |
| `POST P/visits/{id}/confirm` | expected_version、summary | 确认本次拜访；不会自动采纳其所有字段候选 |
| `POST P/analyses` | kind、material_ids、visit_id 可选 | 202 job；kind 固定为 EXTRACT/OPPORTUNITY/COOPERATION/DD_REPORT/EXPLAIN_CHANGE |
| `GET J` | 无 | job 状态、脱敏错误、result_refs；授权每次检查 |
| `POST J/cancel` | expected_version、reason | 请求取消并失效 fencing token；外部真实结果仍需核对 |
| `POST J/reauthorize` | expected_version、reason、authorization_ref | 检查现有权限与原动作范围，追加授权事件，不能以历史权限恢复 |
| `POST J/resume` | expected_version、reason | 仅 BLOCKED，重新授权后回 QUEUED；FAILED 则创建 superseding job |
| `GET P/tasks` | status、cursor | 待办与建议分组返回，读取无副作用 |
| `POST P/tasks/{id}/actions` | expected_version、action、owner_id 或 completion_ref | claim/start/block/complete/cancel；审核任务只能由关联业务命令完成 |
| `POST P/suggestions/{id}/decisions` | decision、reason、owner_id | 接受时创建或复用去重后的业务待办 |
| `POST P/artifacts/drafts` | type、instance_key、template_version_id、source_refs | 202 job；冻结快照并生成新修订 |
| `POST P/artifacts/{id}/revisions/{rid}/review` | expected_version、content_hash、snapshot_hash、decision、reason | 审核具体文件；退回保留意见，后续生成新草稿修订 |
| `POST P/artifacts/{id}/revisions/{rid}/publish` | expected_version、content_hash、snapshot_hash | 200 正式登记；version 为 artifact.version，避免并发分配重复修订/当前指针 |
| `GET P/artifacts/{id}/revisions` | cursor | 全部修订、审核、时效与文件引用 |
| `GET /api/v1/files/{file_id}/download` | 无 | 检查关联项目/组织权限与文件可见状态；服务器流式下载，无物理路径 |
| `POST P/dd` | purpose、policy_version_id | 201 尽调准备记录；无审批副作用 |
| `POST P/dd/{id}/submit` | expected_version、snapshot_hash | 202 提交 job；只在授权用户明确操作后调用实际审批接口 |
| `POST P/dd/{id}/materials-review` | expected_version、snapshot_hash、decision、reason | 记录当前材料版本的审核 |
| `POST P/dd/{id}/handover` | expected_version、snapshot_hash、checklist | 接收责任人确认交接 |
| `POST P/dd/{id}/fieldwork` | expected_version、action、reason | start/pause/resume/complete；start/resume 必须通过联合守卫 |
| `GET P/dd/{id}` | 无 | 每个独立状态、席位、依据与阻塞列表；不能只返回“尽调完成” |
| `POST P/calculations` | model_version_id、scenario、baseline_run_id、adopted_fact_refs | 201 PENDING 输入草案；返回完整参数清单/摘要；暂不运行 |
| `POST P/calculations/{id}/confirm-inputs` | expected_version、snapshot_hash、parameter_manifest_hash、confirmed_items | 202；确认完整输入后，同事务创建执行 job |
| `POST P/calculations/{id}/inputs` | expected_version、selections | 仅DRAFT，选择已采纳事实/标准/假设，保存新输入快照；禁止裸数字绕过事实/假设批准 |
| `POST P/calculations/{id}/revise` | expected_version、reason | 从已确认运行创建新DRAFT；旧记录保留，展示变更 |
| `GET P/calculations/current` | 无 | 当前有效正式结果或current=null及last_formal、stale_reason；路由注册在动态id之前 |
| `GET P/calculations/{id}` | 无 | 参数、执行/复核/时效状态、阻塞、结果文件与依据 |
| `POST P/calculations/{id}/review` | expected_version、result_manifest_hash、decision、reason | 成功后指定专家复核；依赖仍有效才更新正式当前指针 |
| `GET P/calculation-differences` | base_run_id、target_run_id | 确定性差异；不能由LLM提供数值 |
| `GET P/history` | cursor、object_type 可选 | 脱敏审计及业务历史；分页不能突破项目权限 |
| `POST /api/v1/integrations/feishu/events` | 原始签名 headers/body | 验签→事件去重→真实查询任务；成功接收不代表审批通过 |

schema 文件不能与实现各维护一份：Pydantic 为请求/响应真源，导出 `contracts/openapi.json` 与关键 JSON Schema。CI 检查导出与提交版本一致。状态枚举在 Python `domain/enums.py` 唯一定义，迁移显式列出数据库 CHECK；增加枚举时同一变更更新迁移和测试。

### 4.4 错误与业务守卫补充

422：结构、类型、数值格式、单位/期间不合法；409：版本/摘要过期、工作流门槛或未解决冲突；403/404：授权问题；503：暂时不可用的外部服务。永久缺配置也必须携带 `dependency_id` 与 `retryable=false`，不能无限重试。后台 BLOCKED 不伪装成 HTTP 成功业务结果，202 仅表示请求已入队。

补充错误码：`AUTH_REVOKED`、`PROJECT_NOT_ACTIVE`、`APPROVAL_GATE_UNMET`、`APPROVAL_CONFIGURATION_MISSING`、`EXTERNAL_RESULT_UNKNOWN`、`MODEL_NOT_READY`、`CALC_INPUT_INCOMPLETE`、`CALC_EXECUTOR_UNAVAILABLE`、`CALC_OUTPUT_INVALID`、`CALC_REVIEW_REQUIRED`、`ARTIFACT_STALE`、`TEMPLATE_UNAVAILABLE`、`PARSE_UNSUPPORTED`。前后端使用统一映射，向用户说明下一步；秘密、访问令牌、供应商原始错误全文不返回浏览器。

ACTIVE 项目允许新业务写入；PAUSED 允许读取、补充材料、纠错和授权恢复，阻止新审批提交、正式现场启动和发布；CLOSED 只读，重新开放必须 OWNER 显式操作并记录理由。历史回退阶段需 OWNER、原因和 stage_version；不会删除已完成动作或自动撤销外部审批。


项目阶段仅允许OWNER显式命令：创建ST1；ST1→ST2需已确认商机跟进决定；ST2→ST3需创建尽调准备记录；ST3→ST4需人工确认移交测算并记录理由；ST4→ST5只记录人工标识。ST3→ST4不等于已完成现场工作，现场与测算仍各自受门槛约束；所有非相邻前进/回退需明确原因并展示未完成事项。stage_version与state_version分别维护，不以aggregate_version替代。

`calculation_runs.execution_status`为读取视图而非重复维护的事实列：未确认无job为PENDING；QUEUED/WAIT_RETRY/BLOCKED映射PENDING并返回原job_status、阻塞原因和重试时间；有效租约RUNNING映射RUNNING，过期租约显示PENDING/RECOVERY_REQUIRED；job成功且完整结果登记后为SUCCEEDED，其余终态对应FAILED/CANCELLED。review_status和freshness独立存储；输入确认记录不可充当结果复核。工作进程不能只更新一个副本而留下另一个RUNNING状态。

执行授权正文不可变、active/撤销时间受控更新。重新授权追加新记录及job_authorization_events，保留原发起人与原授权；新授权人必须有同一动作权限。规则服务账号仅能执行明确白名单中的解析、草稿、内部待办/通知和投影，不能从系统任务推导出领导、专业复核或发布权限。

## 5. 材料、AI、知识与成果质量的实现契约

### 5.1 文件接入与来源定位

首版支持 UTF-8 TXT、具有文本层的 PDF、DOCX、XLSX。扫描 PDF 和图片仅在配置了真实 OCR 适配器后启用；无法提取时返回 `PARSE_UNSUPPORTED/OCR_REQUIRED` 和材料编号，不能把空结果解释为没有业务信息。首期不接收任意网页 URL 或执行附件中的脚本。

工程默认上限：单文件 20 MiB，PDF 200 页，ZIP 解压总量 100 MiB、条目 10,000、单项压缩比 100，文本提取总量 2,000,000 字符；均允许管理员配置。上传前检查扩展名、MIME 与文件特征，拒绝加密、损坏、路径穿越和不支持的格式。限制是工程初值，不是容量承诺，调整时更新边界测试。

文件先在事务外校验并写入临时对象，核验 SHA-256 后在短事务登记 stored_files/materials 与 parse job；事务失败产生可清理孤儿，不能出现“材料已保存但实际文件不存在”。原文件和定位片段不可改，同名重传新增材料编号。

解析器输出 `SourceFragment(material_id,parser_version,locator,text,content_hash)`：

| 格式 | locator 固定结构 | 规则 |
| --- | --- | --- |
| TXT | `{kind:"text_span",start,end}` | 字符索引半开区间，以解码后的原文为准 |
| PDF | `{kind:"pdf_page",page,span_start,span_end}` | page 从1开始；定位到该页提取文本，未获得真实坐标时不输出伪造 bbox |
| DOCX | `{kind:"docx_paragraph",paragraph_index}` 或 table/row/cell | 段落、表格编号从1开始；原 XML 顺序固定，表格不能全部扁平化后丢定位 |
| XLSX | `{kind:"sheet_cell",sheet,cell}` | 保存原值、公式文本、缓存值是否存在；解析时不执行客户文件公式 |

按页、段落和工作表块分段，单个提取请求至多 8,000 字符；大段落按确定性字符边界细分并保留原始偏移。合并、规范化和去重由程序执行，同字段不同期间不合并。重复内容不能丢失不同来源的证据。

### 5.2 提取请求与候选输出

AI 读取授权项目快照和本次允许访问的片段，调用前裁剪不需要的联系人等个人信息。对话历史只能帮助解析意图，不作为已确认数据。

提取能力固定为 EXTRACT/OPPORTUNITY/COOPERATION/DD_REPORT/EXPLAIN_CHANGE。HTTP 的 kind 映射到已注册函数；用户文本、附件和模型均不能注册函数、扩展权限或传入 Python import 路径。

以下为提取响应的有效形状；这里只产生候选，KNOWN 不代表项目已采纳。服务端根据 field_definitions 验证并补齐候选的实际 ID、版本与审核级别。

```json
{
  "schema_version": 1,
  "status": "REVIEW_REQUIRED",
  "candidates": [
    {
      "field_key": "school.student_count",
      "scope_key": "project",
      "period_key": "school-year-2026",
      "payload": {
        "state": "KNOWN",
        "data_type": "integer",
        "value": "1200",
        "unit": "person",
        "basis": "SOURCE",
        "knowledge_version_id": null,
        "assumption_context": null,
        "source_note": null
      },
      "qualifier": "EXACT",
      "evidence": [{"fragment_id": "00000000-0000-4000-8000-000000000001", "quote": "2026学年学生人数为1200人"}]
    }
  ],
  "questions": [],
  "warnings": []
}
```

候选额外保存 `qualifier=EXACT/APPROXIMATE/RANGE/UNSPECIFIED`；“约1200人”不得自动变成精确人数。用于要求精确值的测算时，须人工补证或明确采纳为对应情景的假设。未知/不适用载荷包含非空 `reason`，value=null；整数和decimal都用十进制字符串，日期为 ISO 日期、文本为字符串、布尔为 JSON 布尔，禁止混用。

模型输出不允许包含 confirmed_by、approved、role、publish、SQL 或工具调用；Pydantic `extra='forbid'` 后仍需字段与引用授权验证。只检出同一个数字不足以使证据通过：程序匹配片段及引用原文、可确定的单位和期间，并保留上下文；口径真实性与对象归属仍交由审核人判断。无引用、不匹配或近似来源不能显示“来源定位通过”。

字段缺失生成问题或 UNKNOWN 候选，不制造0。未登记字段只留作上下文/待补字段建议，不进入 fact_heads 或测算。冲突检测程序比较已规范化的同键候选与当前事实，并建立 fact_conflicts；LLM只解释冲突。

### 5.3 LLM 网关与失败处理

本地 `LLM_PROVIDER=fixture` 仅允许DEMO项目，对仓库固定脱敏输入返回固定测试响应；遇到不认识的输入必须报测试能力不支持，不虚构内容。PRODUCTION模式契约正例只允许测试进程在隔离测试库注入tests/fakes替身，报告必须是contract，不能以运行配置开启生产fixture。真实适配器为 `openai_compatible`，通过 HTTPX 调用经过配置的接口；使用何种实际供应商、模型和结构化输出能力必须记录真实验证结果。

配置包括 LLM_BASE_URL、LLM_MODEL、LLM_API_KEY、请求预算和超时；地址仅来自受管配置。模型无 shell、数据库、任意网络或审批工具。引用链接仅允许已存储授权来源，渲染时过滤 HTML/图片和未知 URL，文档内容不能改变系统提示词。

默认连接超时5秒、单调用读取超时60秒、单分析 job 总预算300秒；最多一次无副作用的结构修复调用。先解析JSON，再校验schema和所有来源引用；最终失败写 job错误，当前事实不变。429/暂时故障受 job重试总预算约束，不能每层无限重试。原始返回仅在授权调试记录中留存，不进入普通日志。

每次调用保存 skill/prompt/model/parser 版本、输入快照哈希、来源集合、耗时、用量和费用；供应商未返回用量时写 unknown，不能当作零费用。提示词在 `config/prompts/` 版本控制，评测数据独立于提示词，禁止为通过固定样本而硬编码真实适配器结果。

### 5.4 知识与文档内容

首期用 PostgreSQL 元数据过滤加受控全文/关键词检索；顺序为组织/项目授权→区域和服务范围→有效期→已批准状态→关键词。案例必须脱敏分类后才能组织共享；项目材料默认仅项目可见。引用精确 knowledge_version_id，不能在生成后偷偷换成“最新知识”。

知识提交先 DRAFT，专业责任人批准为 APPROVED；AI不得自动发布规则。撤销/到期使新的正式使用被阻断，历史文件保留原依据和状态说明。过期知识不自动替换成来源不明的新版本。

文档按“模板固定章节＋已确认数据表＋有依据的解释”生成。AI不得在文字中另造财务数字；财务表直接由确定性结果渲染。区域资料不足返回 NEEDS_SOURCE 并列缺口，允许保留标明缺失的内部草稿。

质量检查分为机器可验证项和人工审核项：文件可打开、必需章节、模板占位符、结构化数字一致、引用有效、跨项目引用、测算有效性由代码检查；结论是否受到证据支持、业务建议是否合理由审核人确认。不能宣称仅靠字符串匹配完成全文事实核验。`quality_reports` 每项记录 PASS/FAIL/REVIEW_REQUIRED 与具体位置；机器阻断项失败时不允许进入发布。

## 6. 后台任务、尽调、测算与发布

本节是编码约束。所有枚举保存为字符串并设置数据库 `CHECK` 约束；业务命令必须在服务层执行，HTTP 路由、定时任务和 worker 不得各自复制状态迁移逻辑。时间使用PostgreSQL的UTC时间；浏览器按Asia/Shanghai默认显示并保留时区，可按组织配置调整。下文默认值写入配置，可按压测结果调整。

### 6.1 数据库任务队列

`jobs` 至少保存 `id/project_id/kind/payload/schema_version/input_snapshot_id/requested_by/authorization_id/status/available_at/attempt/max_attempts/lease_owner/lease_token/lease_expires_at/heartbeat_at/checkpoint/error_code/error_detail/result_ref/created_at/finished_at`。`payload` 只保存经校验的参数和对象引用，不存令牌及完整敏感材料。`lease_token` 为单调递增整数。`(tenant_id, project_id, kind, dedupe_key)` 唯一；相同键但不同输入摘要返回冲突。

| 状态 | 进入与退出条件 |
| --- | --- |
| `QUEUED` | 命令事务创建；到期后允许领取 |
| `RUNNING` | 领取成功；持有效租约才能写检查点或提交结果 |
| `WAIT_RETRY` | 明确可重试的暂时错误；到 `available_at` 再领取 |
| `BLOCKED` | 授权失效、配置缺失或外部结果不明；保存具体原因，禁止自动重复执行 |
| `SUCCEEDED` | 业务结果、审计和任务完成在同一事务提交 |
| `FAILED` | 永久错误或重试耗尽；重开须创建带原任务引用的新任务 |
| `CANCELLED` | 用户取消或业务依据撤销；不得登记后续业务结果 |

实现 `claim_one(worker_id)`：短事务中 `SELECT ... FOR UPDATE SKIP LOCKED` 领取 `QUEUED/WAIT_RETRY` 且 `available_at <= clock_timestamp()` 的一条，按 `available_at, created_at` 排序；增加 `attempt` 和 `lease_token`，租约置为当前时间后 60 秒；提交后才开始解析、模型调用或计算。禁止在执行期间持有数据库事务。

worker 每 15 秒心跳。心跳、检查点和完成更新必须同时匹配 `id + lease_owner + lease_token + RUNNING`，并要求租约未过期。更新零行即失去执行权：停止子进程，不写业务结果。模型响应和计算文件只是临时产物，须通过相同 fencing 校验后才能登记。租约过期后旧 worker 无权自行续租。

恢复扫描每 15 秒运行，使用行锁处理过期租约：增加 `lease_token` 立即封锁旧 worker；纯计算、解析和生成任务未耗尽次数时按退避进入 `WAIT_RETRY`；attempt已达max_attempts则FAILED；可能已发送外部请求的任务转 `BLOCKED/EXTERNAL_RESULT_UNKNOWN` 并创建只读核对任务。取消任务同样增加 token，向子进程发送终止信号；已发生的外部操作不能被本地取消“撤销”，继续核对后记录实际结果。

暂时错误限定为已确认无副作用的网络失败、429、可恢复的服务故障；校验失败、权限失败、非法公式、缺配置不重试。默认最多 5 次执行，退避 `min(5 × 2^(attempt-1), 300)` 秒加 0–20% 抖动；429 尊重服务端 `Retry-After`。显式重试记录操作者和原因，不清除旧错误。

`checkpoint` 仅记录已持久化的不可变文件和版本摘要。恢复时校验文件存在、摘要一致和输入快照一致；否则从安全步骤重跑。内存对话历史不作为恢复依据。worker 收到停止信号后停止领取，最多等待 30 秒让正在运行的任务交出检查点，随后终止子进程。

在领取后、恢复后、读取受限材料前、调用外部接口前和提交业务结果前，调用同一 `AuthorizationService`。人工任务绑定发起人及其授权范围；规则任务绑定已启用的规则授权和指定服务身份。用户被移出项目、规则被停用或令牌被撤回时转 `BLOCKED/AUTH_REVOKED`，不自动换用管理员身份。恢复要求有效重新授权并审计，不能修改原授权快照。

### 6.2 Outbox、外部调用和业务待办

业务写入事务同时追加 `audit_events` 与 `outbox_events`，不得先提交业务再直接发通知。Outbox 使用相同租约机制；状态为 `PENDING/IN_FLIGHT/DELIVERED/WAIT_RETRY/BLOCKED`。消费者按 `event_id` 去重；本地消费结果与已消费记录同事务提交。

每次有外部副作用的调用先登记 `external_operations`：`operation_id/provider/kind/project_id/business_key/request_hash/provider_idempotency_key/provider_object_id/status/last_checked_at`，业务键唯一。状态为 `PREPARED/SENT/CONFIRMED/REJECTED/UNKNOWN`。操作记录在网络调用前落库；请求中使用固定 `operation_id` 作为供应商支持的幂等键，重试不得换键。

收到成功响应后登记供应商对象编号，再完成本地业务状态；若发送后超时、断连、worker 崩溃，状态置 `UNKNOWN`，绝不凭超时推断失败。核对流程为：优先按供应商对象编号查询，其次按业务唯一关联键查询；找到唯一对象则恢复映射并确认。只有供应商提供可靠幂等保证，或权威查询明确证明操作没有发生且不会迟到执行，才能重发。无法查询或存在多条匹配时保持阻塞，生成核对待办。Mock 返回值不能消除生产阻塞。

外部通知若接口没有幂等和查询能力，遇到不明结果停止自动重发，界面显示“发送结果待核对”。不声称跨数据库和飞书可以获得普遍的 exactly-once 保证。

多维表格按项目串行同步；事件只触发同步最新本地投影，不把旧事件携带的旧字段直接写出。保存 `local_projection_revision`、远端记录编号和最近确认版本；结果不明时先核对再继续。若接口缺少条件更新能力，增加周期读取核对及最新值修复，明确提供最终一致性，不能保证在途旧请求从未短暂覆盖。索引失败不回滚业务事实。

`business_tasks` 与 `jobs` 是两张表。待办状态为 `OPEN/IN_PROGRESS/BLOCKED/DONE/CANCELLED`。规则待办键为 `(project_id, rule_code, subject_type, subject_id, reason_fingerprint)`；建立活跃状态的部分唯一索引，将同时触发的同事项合并。`reason_fingerprint` 只包含影响该事项的字段版本/缺口，不包含无关项目版本。生成规则必须配置负责人，缺负责人则创建“待分派”记录，不能随机指定。

AI 可选建议另存 `PROPOSED/ACCEPTED/REJECTED` 及理由摘要；同依据被拒绝后保持抑制，只有依据实质变化才生成新建议。审核采纳、交接完成等业务命令在同一事务关闭对应待办。不得只根据“后台任务成功”关闭业务审核任务。

### 6.3 尽调审批与执行守卫

due_diligences保存材料/交接/现场状态，external_status与basis_validity通过current_approval_instance_id派生，真实值只存approval_instances；展示枚举为`external_status=NOT_SUBMITTED/PENDING/APPROVED/REJECTED/CANCELLED/UNKNOWN`，`basis_validity=UNSET/VALID/CHANGED`，`materials_status=DRAFT/IN_REVIEW/APPROVED/CHANGES_REQUESTED`，`handover_status=PENDING/COMPLETED`，`fieldwork_status=NOT_STARTED/IN_PROGRESS/PAUSED/COMPLETED`。禁止把依据变化写成供应商审批被撤销。

每次提交建立新的approval_instances，due_diligences只指向当前实例；审批依据是该实例冻结的 `approval_basis_snapshot_id`，包含配置版本、申请材料文件摘要和实质字段版本。两个席位保存独立 `slot_code`、配置的责任人/有效代理和供应商决定编号；同一决定编号在一个实例中唯一，不能填充两个席位。人员及代理规则由真实审批配置决定。

实现以下命令：

| 命令 | 必须满足的前置条件与效果 |
| --- | --- |
| `submit_approval` | 授权人明确提交；生产审批配置可用；冻结依据，创建外部操作并异步提交 |
| `refresh_approval` | 查询权威实例及席位结果；更新真实状态，依据有效性独立计算 |
| `approve_materials` | 当前实例和两个席位已批准、依据有效；授权审核人对明确材料版本审批，保存审核清单和版本摘要 |
| `complete_handover` | 材料通过，交接清单完整；接收责任人确认并保存相应材料和依据版本 |
| `start_fieldwork` | 全部执行守卫满足；事务内再次检查并转 `IN_PROGRESS` |
| `resume_fieldwork` | 暂停原因解除、全部守卫重新满足；显式人工执行 |

统一守卫 `can_execute_fieldwork` 要求两个规定席位在真实实例中均批准、实例总体批准、依据为 `VALID`、材料审核和交接均针对当前有效版本完成、项目与操作者获授权。缺任一项返回具体阻塞码及待处理对象，不允许界面绕过服务层。

飞书回调先按官方协议验证签名、时间和解密，依据供应商事件编号去重；回调作为触发信号，使用真实接口查询实例及席位后落库。实例必须通过已登记external_operation关联到本租户和项目，不能相信回调自报project_id；按冻结策略规范化获批表单数据，与approval_basis_snapshot中的申请数据摘要核对。不一致或无法取得必要依据时保持BLOCKED，不能借其他项目的批准放行。不得猜测接口地址、字段、席位映射或代理语义。适配器使用第1章统一的 `submit/fetch_authoritative_state/verify_callback` 协议；缺真实能力时实现 `DisabledApprovalAdapter`，返回 `APPROVAL_CONFIGURATION_MISSING`。开发 Mock 只产生独立演示记录，不能写生产审批批准或交接完成。

实质字段变更时，在同一事实更新事务中将匹配依赖的依据置 `CHANGED`；原审批和原审核记录保留，材料/交接的有效性由绑定版本重新判断。已开始但未完成的现场任务转 `PAUSED` 并创建处理待办；系统只能记录和阻断后续操作，不能宣称线下人员已实际停工。已完成历史不回写状态。补充、撤回或重新审批依照配置；未提供正式规则则关闭执行守卫，不能将依据自行改回有效。

### 6.4 测算输入、真实执行及双重门槛

`calculation_models` 保存不可变 `model_version`、`mode=DEMO/PRODUCTION`、原工作簿摘要、mapping 版本、输出精度、批准人、标准样例及执行器镜像摘要。mapping 明确每个输入/输出的字段、sheet、cell 或 named range、类型、单位、期间、范围、必填性及“不适用”处理。生产工作簿、映射、成本排重和标准答案缺失时返回 `MODEL_NOT_READY`，不得用假公式补全。

每次 `calculation_runs` 保存独立编号、`scenario=V0/V1/V2`、`baseline_run_id`、输入快照与依赖、模型版本及 `mode`。执行状态为`PENDING/RUNNING/SUCCEEDED/FAILED/CANCELLED`，按第4章从job和结果派生、不重复维护另一真源；复核状态为 `PENDING/APPROVED/REJECTED`；时效状态为 `CURRENT/STALE`。三者分离，历史成功不能因项目后续变化改写为失败。

输入构建采用确定性步骤：

1. V0 无基线；V1/V2 必须显式选择同项目、同 mode、同模型版本、已成功且已批准的基线。继承完整输入及每项依据，不读取“隐式最新测算”；首期不实现跨模型版本的输入迁移。
2. 仅以明确选中的已采纳事实、批准标准或本情景批准假设覆盖基线；变化展示为参数差异。基线已过期的依赖须重新核实，不能悄悄沿用或静默替换。
3. 校验必填、类型、单位、期间、范围、标准有效期及配置的成本排重约束。`UNKNOWN` 必填阻断；确认的零合法；`NOT_APPLICABLE` 仅按映射明确规则处理。
4. 领域专家确认完整参数表及输入摘要；记录每项来源和所见版本。冻结后不可编辑；修改创建新运行，复核状态重新开始。

生产执行前门槛：项目为生产模式，模型为 `PRODUCTION` 且批准，业务标准答案在指定执行器通过，参数已确认，无必填缺失或未解决关键冲突。演示项目、模型和运行必须同时为 `DEMO`；所有文件加演示标记，不能通过修改请求参数转生产。

首个 Excel 执行器采用 LibreOffice headless，但必须先通过能力验证，不能承诺兼容任意 Excel。`openpyxl` 只负责读取结构和向获准输入单元格写值。实现 `SpreadsheetExecutor.recalculate(input_path, output_dir, timeout)`：使用本机管道连接预置 LO UNO 进程，打开后调用 `calculateAll()`，保存到新路径再关闭；独立 LO 用户配置目录，输入输出目录分离。不能把只设置 `fullCalcOnLoad` 或只运行格式转换当作可靠的强制重算。

使用 Docker Compose 专用 `calc-runner` 容器，`network_mode: none`、`read_only: true`、`cap_drop: [ALL]`、`security_opt: [no-new-privileges:true]`，配置非 root UID、`/tmp` 限额 tmpfs 和专用 `calc-spool` 卷。worker 仅挂载同一卷，不挂 `docker.sock`；runner 不连接数据库，也不接收对象存储、LLM 或业务凭据。容器配置限制为 2 CPU、2 GiB 和 64 PID，默认单 runner 顺序执行。

共享卷协议必须实现为固定目录和 JSON 契约：worker 为每次租约创建独立 `execution_id`，先写 `/spool/staging/{id}/input.xlsx` 和 `request.json`，再原子重命名到 `/spool/ready/{id}`。manifest仅允许第8章request.json列出的字段，包含model_version_id/mapping_sha256/probe_nonce及各输入身份；不允许 shell 命令、下载 URL 或任意路径。runner 以原子重命名领取到 `/spool/running/{id}`，拒绝符号链接、非普通文件、未知 manifest 字段和越界大小，校验摘要后调用固定执行器。

runner 将 `output.xlsx`、日志和 `result.json` 先写临时目录，再原子发布到 `/spool/completed/{id}`；结果包含状态、输出摘要、执行器版本、开始/结束时间和错误码。worker 轮询结果，同时独立续租，验证 execution/token/摘要后将文件写入正式文件库。失去租约时写取消标记；runner 每秒检查并终止对应 LO 进程组。旧租约结果即使完成也不能提交。

runner 启动时将遗留 `running` 请求输出为 `RUNNER_INTERRUPTED`，不擅自重跑；worker 按任务规则以新 execution_id 重试。完成文件由 worker 登记后清理；无引用的残留目录按保留期回收。M0 必须实际验证当前部署支持这些资源限制与 LO 能力；容器或能力不可用时返回 `CALC_EXECUTOR_UNAVAILABLE`，不可退回在 API/worker 进程直接执行 Excel。DEMO 工作簿明确为 synthetic fixture，用简单已知公式验证链路，不代表任何业务测算口径。

首期只接受 `.xlsx`；拒绝宏、外链、外部数据连接、嵌入执行内容和未支持的公式特性。默认执行超时 120 秒，超时杀死进程组。工作簿先检查大小和 ZIP 解压总量，防止超量展开。

保存后重新打开输出工作簿，验证模型版本、输入回读、公式保留、允许的结构、所有规定输出及依赖区域的公式错误；使用data_only=True检查缓存及错误类型；金额等数值以OOXML的v原始文本转Decimal，详见第8章。受控执行器只向模板预注册的诊断位置写随机nonce，并使用预批准的诊断公式验证本轮重算；不得运行时增加生产公式或工作表，完整mapping及数值精度规则见第8章。nonce 只能证明重算发生，业务公式正确性仍由批准样例及允许功能验证保证。没有公式、缺输出缓存、错误值或执行器不兼容时失败；不能回用旧缓存充当新结果。

保存输入、输出、实际重算工作簿、stdout/stderr 摘要、执行器与模型版本。复核人必须在成功后看到这些结果再确认。生产结果作为正式测算及文档输入的第二道门槛：`mode=PRODUCTION && execution=SUCCEEDED && review=APPROVED && freshness=CURRENT`，并且依赖及模型批准仍有效。

事实变化只使实际依赖运行变为 `STALE`，同时使依赖它的文档待更新；历史结果、复核意见和文件均保留。正式当前指针不能继续表示该运行适用于最新条件，UI 保留“上一正式结果，已过期”链接。重复执行不能自动移动当前正式指针，只有专业复核命令可以移动。

差异只比较明确选择的两次成功运行；同项目、mode、模型版本和口径一致才生成数值差异，否则返回不可比原因。数值用 `Decimal` 和 mapping 定义的舍入方式；`delta=new-old`，`pct=delta/abs(old)`，`old=0` 返回 null 并显示“不适用”。输出逐项输入差异、依据与结果差异，不实施多因素归因。

### 6.5 依赖失效与文件发布

所有冻结对象使用第3章唯一的 `snapshots/dependency_refs`；禁止另建artifact_dependencies，来源反向索引也建在dependency_refs。依赖包含实际使用的事实、标准、知识、模板及测算；不使用整个项目版本代替具体依赖。草稿、审批依据、参数确认、测算复核和文件审核均绑定确定的内容摘要。

采用一条明确的事务锁约定：修改事实、冻结快照、确认审核、发布及改变当前指针时先锁项目行；项目业务检查共享许可时再按固定顺序取共享锁；组织级撤销只锁该共享资源并写outbox，不回锁项目，具体遵循第3.6节。网络、解析和文件生成均在事务外。字段并发冲突仍比较各自 `expected_revision_no`，无关字段更新不报冲突。所有调用方遵循相同锁顺序，避免检查依赖后到提交间发生竞态。

字段采纳事务追加事实历史、更新当前事实，并沿反向依赖将相关对象标为待更新，同时写审计、规则待办和 Outbox。即使异步展示尚未刷新，发布和正式测算读取入口仍现场比较依赖版本。被撤销的标准/模板/模型不得通过绑定旧版本继续绕过限制。

文件版本状态为 `DRAFT/IN_REVIEW/APPROVED/PUBLISHED`，`freshness` 独立保存；退回产生带意见的新草稿修订，已发布文件永不覆盖。人工修改正文或重新生成后原审核失效，新的内容摘要必须重新审核。

发布按以下顺序实现：

1. 基于冻结快照生成文件到临时目录，执行章节、占位符、数字、引用和依赖检查；人工审核的必须是同一份文件摘要。
2. 文件写入不可变随机键；本地存储使用同文件系统临时文件、`fsync` 后 `rename`，对象存储使用唯一键、校验摘要并确认写入成功。`STAGING` 文件不提供正式下载。
3. 短事务锁项目与资源行，重新核对发布权限、审核摘要、依赖与模板有效性；仅template.requires_calculation=true的生产测算类文档还需通过生产测算第二道门槛；通过后登记 `PUBLISHED`、文件引用和当前版本指针，同时写审计和 Outbox。
4. 事务失败则文件保持未发布孤儿，后续清理；不得登记成功链接。响应丢失重试返回同一已发布版本。同步失败只使同步待重试。

文件库与 PostgreSQL 没有跨系统原子事务；这里保证的是“完整文件存在后才原子公开业务引用”。下载只能通过服务端鉴权并检查文件状态，不能公开存储桶或把临时路径交给用户。清理器仅删除超过 24 小时且未被任何活跃任务、草稿或正式版本引用的临时/孤儿文件；删除前再次检查引用。正式文件缺失触发运行告警，不自动重新生成或伪造已发布成果。

### 6.6 必须覆盖的执行验收

- 两个 worker 同时领取、租约过期后旧 worker 返回、取消后计算返回：只能有一个有效业务提交。
- 审批提交已成功但响应丢失：恢复只能核对，不能创建第二个实例；无查询能力时明确阻塞。
- 授权在任务执行中撤销：提交被拒绝，临时结果不成为可见业务成果。
- 第二席位缺失、回调伪造、重复决定、审批依据变化：正式现场执行均被拒绝。
- 专家确认参数后事实被修改：执行/复核/发布按实际依赖阻断，历史结果不覆盖。
- Excel 只有旧缓存、未重算、公式不支持、输出错误、执行超时：均失败；演示结果不能进入生产文档。
- 文件已写但数据库提交失败、发布响应丢失、发布同时修改依赖：无假发布、无重复版本、无过期依据发布。

上述并发测试必须使用真实 PostgreSQL 两个连接和同步屏障，不能用顺序调用伪装并发。真实飞书与业务模型未提供时，对应验收标记“依赖阻塞”；Mock 合同测试通过不能改记为生产通过。

### 6.7 锁顺序、恢复与环境隔离的补充

claim/heartbeat以及只调整租约的reclaim事务仅锁jobs，不取得project锁。需同时修改业务和job的完成/取消/恢复命令，统一先project再job，再成员、业务对象、字段头和共享资源；不能先锁job后等待project。最终事务持job锁重新检查token/owner/status/expiry直至提交。恢复扫描要产生核对待办时先写恢复事件，由项目命令消费，不能反向获取项目锁。fencing检查不只在执行前做一次。

领取同样检查attempt<max_attempts。外部操作PREPARED→SENT须在网络发送前持久化；SENT之后响应未知按UNKNOWN核对。若进程在实际发送前崩溃但记录已经SENT，也保守核对，不凭本地猜测重发。审批配置缺失为BLOCKED；实际请求明确被业务拒绝为REJECTED业务结果，不能自动重试以绕开拒绝。

业务待办OPEN→IN_PROGRESS/BLOCKED/DONE/CANCELLED，IN_PROGRESS→BLOCKED/DONE/CANCELLED，BLOCKED→OPEN/CANCELLED；DONE/CANCELLED终态，重开新建关联记录。普通待办complete需负责人及completion_ref；审核类只能由对应决定成功后自动完成。活跃待办在主体相同且原因被新版本替代时更新显示或关闭旧项，不能把基于旧快照的事项标为已完成。

DEMO文档可以使用DEMO模板经过相同审核形成带演示水印的PUBLISHED修订，供本地下载验证，但只更新DEMO项目的成果指针；不得出现在PRODUCTION项目、正式业务索引或真实客户交付。非测算的生产拜访/商机文档不强制要求存在CalcRun；测算类正式文档必须通过第6.4节的第二道门槛。测试库构造的生产模式记录仍是测试证据，不构成真实生产批准。

生产模型或知识撤销时采用第3.6节共享锁协议；STALE是便于展示的缓存，正式使用始终现查依赖。规则与参数/模型变更不能将历史STALE自行改回CURRENT；若业务重新核实应产生新输入确认或新运行。过期历史继续可按当前访问权限读取，当前正式查询返回current=null及last_formal，不静默返回已失效结果。

## 7. 审核页面、认证与可观察性

### 7.1 首期页面与交互

| 页面 | 必须提供的操作 |
| --- | --- |
| `/login` | 本地开发身份或实际企业登录；明确当前环境 |
| `/projects` | 授权项目列表、创建项目、阶段与阻塞摘要 |
| `/projects/{id}` | 项目事实、独立拜访、材料上传、待办、尽调状态、测算和成果入口 |
| `/projects/{id}/reviews/{rid}` | 新旧值、原文定位、单位/范围/期间/依据并排展示，逐项确认，拒绝理由 |
| `/projects/{id}/artifacts/{id}` | 草稿与历史、质量检查、指定文件审核、发布、下载 |
| `/projects/{id}/calculations/{id}` | 完整输入和依据确认、执行进度、结果文件、专业复核、明确基线的差异 |

复用少量页面和局部面板，不开发独立大屏或管理平台。客户端轮询 job状态默认2秒，逐步退避到10秒；关闭页面不取消任务。HTTP403/409必须显示原因并刷新相关审核面板，禁止在旧确认按钮上悄悄换用新snapshot_hash。

关键字段的确认项不预选；用户可以一次提交已逐项检查的多个决定，但不提供跨关键字段“一键全选”。项目切换清空当前审核引用，服务器仍以URL项目和对象归属复核。新鲜度、DEMO、未知和假设同时有文字标识，不仅用颜色。

### 7.2 认证与 session

`APP_ENV=local/test/production`；`AUTH_MODE=dev/feishu`。本地默认dev，只能选择服务器种子中已登记的测试用户ID，不能从请求接收角色。生产若设置dev直接启动失败。正式飞书身份尚未配置时，生产登录不可用，不能自动切换dev。

增加 `auth_sessions(id,token_hash,user_id,tenant_id,expires_at,revoked_at,csrf_secret_hash)`。session token使用加密安全随机数，数据库仅存摘要；8小时有效、退出即撤销，cookie设HttpOnly/SameSite=Lax，生产Secure。开发HTTP例外仅限绑定127.0.0.1。所有浏览器写请求验证CSRF与同源；API不要接受浏览器传入的tenant_id/roles。

OAuth回调验证state及适用协议的nonce，令牌只存服务端加密存储，实际过期时间以响应为准。身份成功不等于项目授权：首次登录只建立组织账号，不自动授予所有项目。成员撤销及时使后台授权失效。

### 7.3 日志、健康和备份

结构化日志字段：request_id、command_id、project_id、job_id、attempt、lease_token、operation_id、耗时、错误码、所用版本；不记录凭据或完整材料正文。敏感内容的业务审计通过授权对象引用查看。`/healthz`只检查进程；`/readyz`检查数据库、迁移版本和必需文件存储，返回脱敏依赖状态。可选LLM/审批未启用不使整个进程不健康，但相应业务入口必须显示阻塞。

首期指标包括任务队列长度、超时/重试/阻塞、审核积压、计算失败、外部UNKNOWN、同步延迟及AI用量。输出可供采集的结构化计数即可，不以前置购买监控平台为条件。

实现备份/恢复脚本，覆盖PostgreSQL、受控文件、模板/模型注册表和配置版本；恢复在独立环境校验引用和哈希。演练不连接生产目标。RPO/RTO/SLA和数据保留周期由DEP-05确定，未验证前不得填入保证数值。

## 8. 模型注册、数值精度与计算协议样例

本节为前述测算执行器的正式补充契约。本章所有样例均为合成链路验证；单价乘数量不代表学校后勤业务公式。

### 8.1 必须补齐的实现约束

1. **诊断位置提前批准。** nonce 所在工作表、写入单元格、诊断公式及工作簿结构都必须存在于注册模板及 mapping。禁止在运行时向已批准生产模型擅自新增 sheet、公式或命名范围。没有诊断位置的模型，须重新注册带诊断位置的版本，或采用另外已验证且能证明重算的执行器契约；默认阻塞。
2. **结果正确和确实重算分别验证。** nonce 只证明本轮执行，不能证明业务公式正确；注册验证至少运行两组输入不同、预期输出不同的独立 golden case。标准答案不得由同一 LO 执行器临时计算后自证。生产 golden 必须来自业务负责人批准的独立依据。
3. **传输 Decimal 不代表 Excel 任意精度。** JSON 数值契约使用十进制字符串；Excel/LO 计算仍是有限精度浮点。每个输入最多 15 位有效数字，超过即拒绝，不自动截断；模型注册还须验证其量级与舍入要求。读取 `.xlsx` XML 的数值缓存 `<v>` 原始文本构建 `Decimal`，不先变成 Python `float` 再构建 Decimal。文档格式和数值单元格的缓存值分别处理。
4. **明确误差和舍入。** 输出按 mapping 的 scale、rounding 量化；golden 同时验证原始缓存的误差界及量化值。误差界为 `max(abs_tolerance, abs(expected) × rel_tolerance)`，且量化值必须等于批准预期量化值。误差规则不得由 LLM 推断。mapping 的舍入仅定义数据接口；业务中间步骤需要舍入时必须体现在原公式。
5. **补上无 DB runner 的可信注册表。** runner 除共享 spool 外，挂载只读 `/registry`（注册模板、mapping、golden、状态清单）；worker 不得写此挂载。部署/模型登记流程按已审核版本发布该目录。request 加入 `model_version_id`、`mapping_sha256`、`probe_nonce`。runner 校验 registry、执行器版本、允许结构和允许写单元格；不根据用户请求装载任意路径或执行器。没有有效注册表时返回 `MODEL_REGISTRY_NOT_READY`，服务端将 job 置 `BLOCKED`。
6. **区分模板摘要和输入摘要。** `model_sha256` 指注册原模板，`input_sha256` 指已填参数及 nonce 的待执行文件，二者通常不同。runner 以允许写单元格为白名单比较输入文件与模板：允许输入值变化，禁止业务公式、其他单元格值、sheet 清单、外链及命名范围被改变。ZIP 元数据/序列化顺序不能直接作为语义结构变更，使用规范化工作簿结构比较；原模板/输入完整文件仍分别留存 SHA-256。
7. **模式不能被参数升级。** mode 来自项目和注册模型，API 不接受任意覆盖。DEMO 的模型、运行、文件始终传播 DEMO 标记。填写 engine/hash 只意味着完成技术验证，不构成业务生产批准。

### 8.2 完整 `schema_version=1` DEMO 模型注册及 mapping 示例

此 JSON 可由 Codex 用作 Pydantic 模型及合成 fixture 的基准。它是 **DRAFT / NOT_READY**；null 摘要、执行器及审批信息必须使运行门槛关闭。生成 fixture 后填写实际文件摘要，验证后登记 DEMO 技术状态，不得补造生产批准。

```json
{
  "schema_version": 1,
  "model_version_id": "00000000-0000-4000-8000-000000000101",
  "model_key": "synthetic_quantity_price",
  "version": "1.0.0",
  "mode": "DEMO",
  "synthetic": true,
  "status": "DRAFT",
  "readiness": "NOT_READY",
  "display_name": "合成测算：数量乘单价",
  "disclaimer": "仅用于技术链路验证，不是业务测算公式",
  "applicability": {
    "scope_key": "project",
    "period_key": "demo-period"
  },
  "workbook": {
    "file": "template.xlsx",
    "format": "xlsx",
    "template_sha256": null,
    "allowed_structure": {
      "sheets": [
        "Calc",
        "__diag"
      ],
      "hidden_sheets": [
        "__diag"
      ],
      "defined_names": [],
      "writable_cells": [
        "Calc!B2",
        "Calc!B3",
        "__diag!A1"
      ],
      "formula_cells": {
        "Calc!B4": "=B2*B3",
        "__diag!A2": "=A1*2+1"
      },
      "formula_dialect": "EXCEL_A1",
      "fixed_cells": {
        "Calc!A1": "SYNTHETIC DEMO ONLY",
        "Calc!A2": "quantity",
        "Calc!A3": "unit_price",
        "Calc!A4": "amount"
      },
      "number_formats": {
        "Calc!B2": "0",
        "Calc!B3": "0.00",
        "Calc!B4": "0.00",
        "__diag!A1": "0",
        "__diag!A2": "0"
      },
      "unlisted_cells": "MUST_BE_EMPTY",
      "macros": "FORBIDDEN",
      "external_links": "FORBIDDEN",
      "data_connections": "FORBIDDEN",
      "embedded_objects": "FORBIDDEN"
    }
  },
  "inputs": [
    {
      "key": "quantity",
      "label": "合成数量",
      "field_key": "demo.quantity",
      "location": {
        "sheet": "Calc",
        "cell": "B2"
      },
      "type": "integer",
      "unit": "item",
      "required": true,
      "minimum": "0",
      "maximum": "999999",
      "scale": 0,
      "max_significant_digits": 15,
      "zero_allowed": true,
      "unknown_policy": "BLOCK",
      "not_applicable_policy": "BLOCK",
      "allowed_basis": [
        "SOURCE",
        "ASSUMPTION"
      ],
      "unit_conversion": "NONE",
      "scope_key": "project",
      "period_key": "demo-period"
    },
    {
      "key": "unit_price",
      "label": "合成单价",
      "field_key": "demo.unit_price",
      "location": {
        "sheet": "Calc",
        "cell": "B3"
      },
      "type": "decimal",
      "unit": "CNY/item",
      "required": true,
      "minimum": "0.00",
      "maximum": "999999.99",
      "scale": 2,
      "max_significant_digits": 15,
      "zero_allowed": true,
      "unknown_policy": "BLOCK",
      "not_applicable_policy": "BLOCK",
      "allowed_basis": [
        "SOURCE",
        "ASSUMPTION"
      ],
      "unit_conversion": "NONE",
      "scope_key": "project",
      "period_key": "demo-period"
    }
  ],
  "outputs": [
    {
      "key": "amount",
      "label": "合成金额",
      "location": {
        "sheet": "Calc",
        "cell": "B4"
      },
      "type": "decimal",
      "unit": "CNY",
      "required": true,
      "formula_required": true,
      "cache_required": true,
      "minimum": "0.00",
      "scale": 2,
      "rounding": "ROUND_HALF_UP",
      "max_quantized_significant_digits": 15,
      "abs_tolerance": "0.000000001",
      "rel_tolerance": "0.000000000001",
      "quantized_golden_match": "EXACT",
      "scope_key": "project",
      "period_key": "demo-period"
    }
  ],
  "probe": {
    "enabled": true,
    "registered_in_template": true,
    "input": {
      "sheet": "__diag",
      "cell": "A1"
    },
    "output": {
      "sheet": "__diag",
      "cell": "A2"
    },
    "nonce_type": "INTEGER",
    "nonce_minimum": "1",
    "nonce_maximum": "2147483647",
    "formula": "=A1*2+1",
    "expected_rule": {
      "kind": "AFFINE_INTEGER",
      "multiplier": "2",
      "offset": "1"
    },
    "comparison": "EXACT_INTEGER"
  },
  "cost_constraints": {
    "status": "NOT_APPLICABLE_SYNTHETIC_MODEL",
    "mutually_exclusive_input_groups": [],
    "required_components": [],
    "explanation": "本合成模型无业务成本项，空约束不得复制为生产成本规则"
  },
  "numeric_policy": {
    "json_encoding": "DECIMAL_STRING",
    "input_max_significant_digits": 15,
    "excess_precision": "REJECT",
    "cache_reader": "OOXML_NUMERIC_V_TEXT_TO_DECIMAL",
    "raw_cache_precision_policy": "PRESERVE_ALL_DIGITS_FOR_TOLERANCE_CHECK",
    "non_finite_values": "REJECT",
    "formula_errors": "REJECT",
    "engine_arithmetic": "SPREADSHEET_FLOATING_POINT",
    "business_arbitrary_precision": false
  },
  "golden_cases": [
    {
      "id": "demo-case-a",
      "inputs": {
        "quantity": "3",
        "unit_price": "9.90"
      },
      "expected_outputs": {
        "amount": "29.70"
      },
      "probe_nonce": "101",
      "expected_probe": "203",
      "expected_source": "INDEPENDENT_SYNTHETIC_ARITHMETIC"
    },
    {
      "id": "demo-case-b",
      "inputs": {
        "quantity": "7",
        "unit_price": "12.50"
      },
      "expected_outputs": {
        "amount": "87.50"
      },
      "probe_nonce": "202",
      "expected_probe": "405",
      "expected_source": "INDEPENDENT_SYNTHETIC_ARITHMETIC"
    }
  ],
  "engine": {
    "profile": "libreoffice-uno-v1",
    "image_digest": null,
    "libreoffice_version": null,
    "runner_build_id": null,
    "verified_at": null,
    "golden_status": "NOT_RUN",
    "golden_report_sha256": null
  },
  "approval": {
    "technical_validation_status": "NOT_RUN",
    "business_approval_status": "NOT_APPLICABLE_DEMO",
    "approved_template_sha256": null,
    "approved_mapping_sha256": null,
    "approved_golden_sha256": null,
    "approved_engine_image_digest": null,
    "approved_by": null,
    "approved_at": null
  }
}
```

摘要不得自引用：`approved_mapping_sha256` 计算对象是上述 `schema_version/applicability/workbook.allowed_structure/inputs/outputs/probe/cost_constraints/numeric_policy` 的规范 JSON（UTF-8、键排序、紧凑格式、无 NaN），不包含 approval、文件哈希、engine 验证结果。`approved_golden_sha256` 针对 golden_cases 的规范 JSON。注册记录的完整文件另存文件摘要。模板摘要针对原 `.xlsx` 完整字节。Pydantic 统一 `extra="forbid"`；DRAFT 允许尚未生成的摘要为 null，readiness=READY且status=APPROVED的运行校验器必须拒绝 null 和不匹配值。

`cost_constraints` 的生产语义须由业务给出；首期实现确定性 `MUTUALLY_EXCLUSIVE`、`REQUIRED_COMPONENTS` 等已定义类型，不开放任意 Python/SQL/Excel 表达式执行。合成模型中的空列表不意味着业务模型默认无成本检查。

### 8.3 runner request/result 最小示例

以下带 `REPLACE_...` 的摘要是文档占位，实际严格 SHA-256 校验必须拒绝，示例不表示任何计算已经执行。所有相对文件名为协议固定值，request 不携带任意路径；工作目录由已校验 UUID 派生。runner 构建信息由镜像生成，不信任 request 指定的版本。

`request.json`：

```json
{
  "schema_version": 1,
  "execution_id": "00000000-0000-4000-8000-000000000201",
  "job_id": "00000000-0000-4000-8000-000000000202",
  "lease_token": 4,
  "model_version_id": "00000000-0000-4000-8000-000000000101",
  "model_sha256": "REPLACE_WITH_ACTUAL_TEMPLATE_SHA256",
  "mapping_sha256": "REPLACE_WITH_ACTUAL_APPROVED_MAPPING_SHA256",
  "input_sha256": "REPLACE_WITH_ACTUAL_FILLED_INPUT_SHA256",
  "probe_nonce": "100003",
  "timeout_seconds": 120
}
```

`result.json` 成功形状：

```json
{
  "schema_version": 1,
  "execution_id": "00000000-0000-4000-8000-000000000201",
  "job_id": "00000000-0000-4000-8000-000000000202",
  "lease_token": 4,
  "status": "SUCCEEDED",
  "input_sha256": "REPLACE_WITH_ACTUAL_FILLED_INPUT_SHA256",
  "output_sha256": "REPLACE_WITH_ACTUAL_OUTPUT_SHA256",
  "mapping_sha256": "REPLACE_WITH_ACTUAL_APPROVED_MAPPING_SHA256",
  "engine": {
    "profile": "libreoffice-uno-v1",
    "image_digest": "REPLACE_WITH_ACTUAL_IMAGE_DIGEST",
    "runner_build_id": "REPLACE_WITH_ACTUAL_BUILD_ID"
  },
  "probe": {"expected": "200007", "actual": "200007", "passed": true},
  "started_at": "2026-10-09T03:00:00Z",
  "finished_at": "2026-10-09T03:00:02Z",
  "error": null
}
```

失败结果仍回显身份、token、输入摘要和时间；`status="FAILED"`、`output_sha256=null`、`probe=null`，`error={"code":"MODEL_REGISTRY_NOT_READY","message":"缺少已验证执行器与模型摘要","retryable":false}`。同一结构采用Pydantic判别联合约束成功和失败必填项；SCHEMA或SHA占位不合法时请求在执行前被拒绝，不生成成功result；不存在 `FAILED` 同时可采纳输出的形态。worker 重新解析输出 workbook，而不是信任结果清单声称的数值及 probe。计算数值另写业务 `calculation_outputs`；runner 不直接写项目事实、测算复核或文档发布记录。

### 8.4 人工参数确认与异步运行的命令序列

下列状态是 **输入确认状态**，不得与 execution/review/freshness 混用：`DRAFT/CONFIRMED/SUPERSEDED`。原执行状态中的 `PENDING` 可以涵盖等待确认；此时 `job_id=null`，尚未入队。

1. `create_calculation_run(project, scenario, model_version_id, baseline_run_id)`：生成输入建议和相对基线差异，`input_status=DRAFT`、`execution=PENDING`、`review=PENDING`；暂不创建 job。已采纳项目事实直接引用其 `fact_revision_id` 及确认记录，不重新创建候选事实或重复事实审核。
2. `set_calculation_inputs(run_id, expected_version, selections)`：只在 DRAFT 允许改动，按参数选择已有事实版本、正式标准版本或批准假设版本。前端展示值、依据、口径、继承来源、缺失与冲突。人工直接更正“项目事实”应调用现有事实审核命令，再重新选择该版本；情景假设走独立的假设批准，不把假设暗写成项目事实。
3. `confirm_and_enqueue_calculation(run_id, expected_version, input_digest, idempotency_key)`：领域专家一次确认完整参数表，表达“这些已采纳事实及其他依据适用于本轮测算”。事务内按统一锁序锁项目、任务（如已有）、成员/运行及字段头，再共享锁模型，核对授权、输入摘要、各依赖版本及生产执行前门槛；冻结输入，追加 `PARAMETERS_CONFIRMED` 审核事件，置 input_status=CONFIRMED，并在同一事务创建唯一 job、审计和 Outbox。响应 `202`，返回 run_id/job_id/status_url。事务失败不能留下“确认成功但未排队”。重复同请求返回同一运行和 job；相同幂等键不同输入返回 409。
4. worker 领取后再次校验权限和依赖，执行状态转 RUNNING，生成本租约 execution_id 及 probe，投递 spool，等待结果时持续心跳。成功登记文件和结果后 execution=SUCCEEDED、review=PENDING；失败保存错误，不能自动批准。计算只更新该运行，不覆盖原正式结果。
5. `review_calculation(run_id, expected_result_digest, decision, reason)`：指定专业人员查看真正执行后的结果及附件，再 APPROVE/REJECT；审批时再次核对依赖和 mode。APPROVE 才移动正式当前指针并关闭对应待办。参数确认记录不能当作结果复核记录，同一人可连续完成两项职责但不能提前复核尚未产生的结果。
6. 参数变更不能更新已 CONFIRMED 运行：`revise_calculation_run(source_run_id)` 创建新 DRAFT 并展示变化，历史确认和运行保持可追溯。临时失败在原冻结输入下恢复；输入已过期时转 BLOCKED/INPUT_STALE，需要重新建运行及确认，不自动换用最新数据重试。

避免重复审核的边界：事实审核回答“项目当前事实是什么”；参数确认回答“本轮使用哪些已确认事实、标准与假设”；结果复核回答“该次真实计算结果能否作为正式成果”。UI 可合并展示来源审核记录，但三种决定的对象、时间和版本不能混淆。

### 8.5 注册落地与批准状态

demo-period须由种子创建为有明确起止日期的SCHOOL_YEAR登记（2026-09-01至2027-08-31），scope_key=project；demo.quantity/unit_price为DEMO专用字段定义，禁止被生产模型引用。上述mode、basis及数据类型与第3章使用同一枚举。

`readiness`是注册状态、文件摘要、诊断/功能及golden验证结果计算出的派生字段，不再增加可手改的独立真源。`calculation_models.status`统一DRAFT/APPROVED/RETIRED：DEMO通过技术验收并由本地演示责任人显式确认后可APPROVED；PRODUCTION还必须有真实专业批准。批准绑定确切模板、mapping、golden与执行器摘要，任一正文变化新建版本。

实现 `uv run presales model validate --manifest <path> --mode DEMO` 生成验证报告；随后通过 `uv run presales model approve --model-version <id> --evidence <report>` 在已认证管理员/专业责任人权限下登记批准。PRODUCTION不得使用默认演示责任人或测试报告。注册发布将已批准模板和mapping原子写入版本化只读registry；runner仅接受已登记版本，worker仍在开始与登记结果时复核数据库批准状态。

模型审批/配置CLI必须使用与HTTP相同的服务、认证与审计，不能直接UPDATE表。生产首次组织/管理员由受控部署初始化命令配置，不提供公开自注册管理员接口。runner镜像的build_id在构建时写入；image_digest由部署对实际镜像验证后只读注入，不把镜像摘要嵌入自身造成自引用。

读取原始缓存全部小数位用于误差检查，量化后再验证输出有效位；不要因为常见浮点缓存出现17位文本就直接拒绝。输入15位限制并不保证任意业务公式误差合格，仍须验证整个业务golden集合。DEMO的简单乘法及诊断公式仅用于技术样例，不能推断为供餐利润公式。

## 9. 分期实施与Codex任务包

外部依赖统一编号：`DEP-01` 真实模型、参数映射及标准答案；`DEP-02` 飞书真实身份与入口；`DEP-03` 审批模板、责任角色及条件变化规则；`DEP-04` 业务模板、字段、知识与脱敏材料；`DEP-05` 生产环境及大模型数据使用政策。`DEP-01` 缺失不阻断独立 DEMO 真引擎链；M2 上线必须有受控数据的真实 LLM 及真实身份验证，离线 fixture 只证明契约。

每包同时交付所需迁移、服务、API、必要页面、自动化验证和实施记录；不以仅建目录、空接口或页面占位结束。下表“正例/反例”是必须覆盖的代表行为，完整要求与 AC 清单共同生效。

| 任务包 / 阶段 / 依赖 | 交付物 | 正例 / 反例 | 结束条件 |
| --- | --- | --- | --- |
| P01 最小可见事实闭环；M0→M1；无 | 环境预检；基本PG worker、原子事务/幂等/审计；项目及成员；独立拜访；上传 UTF-8 文本；以 fixture 提取候选；人工确认；事实及历史；浏览器显示带来源的参考草稿；首版迁移与运行命令 | 上传“学生 1200 人”，确认后显示来源；未确认不得成为事实，非成员读取被拒绝 | AC-01/02/03/04 本包子集通过；`make demo` 可重放，并明确“本地模拟提取、参考草稿” |
| P02 材料与事实可靠性；M1；P01 | PDF/DOCX/XLSX 支持范围与拒绝规则；来源定位；AI 适配器；未知/零/不适用/标准/假设；字段级并发及幂等/原子审计完整验证；共享快照与最小标准/模板版本登记 | 两个不同字段可独立确认；同字段过期确认、单位错误、跨年覆盖和同键不同载荷被拒绝 | AC-03/04/05/06/15 通过；扫描件无法 OCR 时给出可处理阻塞 |
| P03 文档可审核可追溯；M1；P02 | 版本化 DOCX 草稿；模板登记；质量检查；审核发布；依赖快照；文件哈希；受控下载 | 测试批准模板生成文件可打开；关键字段变化阻止旧稿发布，无关变化不阻止 | AC-07/08 通过；测试发布与真实业务模板批准分开记录 |
| P04 独立真实 DEMO 计算；M1；P03 | 明确公式和 Decimal 精度的演示模型；冻结输入；实际运行；两次结果及差异；Excel 附件与日志 | 标准算例与手算一致；缺值不补零、零基准百分比不计算、演示结果禁止正式发布 | AC-09/10 本地部分通过；M1 本地验收报告形成，真实模型依赖单列 |
| P05 可恢复的持续任务；M2；P03/P04 | 在P01基础上完善PG worker、租约恢复、稳定任务编号、outbox可靠投递；规则待办与建议；修改影响和任务自动关闭 | 杀死 worker 后恢复且不重复产生结果；成员移除后重试暂停；不确定外部结果先查询 | AC-11/12 通过；业务事务提交不受索引同步失败影响 |
| P06 ST1/ST2 业务在线；M2；P05 | 商机初判、合作方向确认、多次拜访、知识标签检索与版本；项目工作台；真实 LLM 适配与受控数据评测；专业审核 | 同一项目第二次拜访保留首次历史；过期资料不能作为有效标准，模型不能批准阶段 | AC-02/13/15 通过；本地 UI 全流程可验，真实 LLM 评测与试用另验，M2 上线必验 |
| P07 身份、通知与索引；M2；P06 | 飞书身份及令牌生命周期、回调验证、单向索引同步、通知适配器；连接器运行手册 | 有权限的真实身份可访问；回调伪造、授权撤回、旧事件覆盖被拒绝；模拟器覆盖故障 | AC-14 契约部分通过；DEP-02/05 缺失则 M2 上线 BLOCKED，继续 P08 独立实现 |
| P08 ST3 审批与交接；M3；P05，生产执行另依赖 P07 真实验证 | 尽调记录、双席位、外部状态与依据有效性分离；材料审核、交接、现场门槛与报告 | 完整条件允许现场开始；重复批准不能凑双席位，依据实质变化关闭门槛但保留历史 | AC-16 本地部分通过；没有真实审批证据不得计为 ST3 生产通过 |
| P09 ST4 与生产门；M3；P04/P08 | 批准模型接入、V0/V1/V2 基线、参数审核、结果复核、比较与正式成果；备份恢复、生产配置和上线报告 | 标准答案一致且复核后可发布；公式错误、未复核、DEMO 或口径不同比较均阻断 | AC-09/10/17/18 通过相应范围；全部适用真实门通过后才能声明上线完成 |

## 10. 统一验收编号和证据

同一 AC 可有多个案例，用 `AC-05-01` 等后缀标识。每项记录 `id、package、scope(local/contract/external)、status、command、commit、evidence、blocker、owner`。状态仅为 `PASS/FAIL/BLOCKED`；测试未运行、依赖缺失均不能计作 PASS。没有条件实现的任务记录为 BLOCKED 并说明下一步，无须停止其余独立任务。

| ID | 必验行为 |
| --- | --- |
| AC-01 | 干净环境启动、迁移到 head、就绪检查、专用测试库隔离、可重复演示 |
| AC-02 | 项目权限、角色动作、跨项目文件访问、开发身份生产禁用；越权不产生写入 |
| AC-03 | 来源位置、对象/期间/范围/单位；未知与零区分；候选不能直接生效 |
| AC-04 | 人工采纳、拒绝及理由；每次拜访独立；历史和确认依据不覆盖 |
| AC-05 | 同字段竞争失败、不同字段成功、同请求重放、同键不同请求拒绝 |
| AC-06 | 故障注入时事实、历史、审核、审计、outbox 同时提交或同时回滚 |
| AC-07 | DOCX 可打开；章节、占位符、关键数字、来源、项目隔离检查 |
| AC-08 | 文件失败不发布；相关依赖变化使审核过期；正式历史保留；无关变化不误阻断 |
| AC-09 | 真运行、批准精度、独立预期输出、执行日志及产物；缓存或写入单元格不冒充重算 |
| AC-10 | 缺失/错误输入、DEMO、未复核、错误公式、不可比较口径、零基准处理 |
| AC-11 | worker 中止和租约回收、重复领取、超时、达到重试上限、恢复前权限重检 |
| AC-12 | 规则待办去重、拒绝建议不重复打扰、业务完成关闭待办、失败外送不回滚业务 |
| AC-13 | ST1/ST2 完整流程、知识适用期及授权范围、人工改变阶段 |
| AC-14 | 外部身份、回调验签/防重放、令牌失效、结果不确定时查证、索引乱序保护 |
| AC-15 | 附件提示注入、模型越权输出、结构错误及无来源数字不能绕过审核；模型升级固定样本回归 |
| AC-16 | 双领导席位、材料审核、交接、依据有效性联合门槛与实质变化处理 |
| AC-17 | 真实业务模型标准答案与人工复核；真实身份、审批、模板、存储和模型接入证据 |
| AC-18 | 生产严格配置、最小权限、备份恢复演练、日志不含秘密、恢复后文件引用完整 |

本地测试不得依赖真实 API 密钥。集成测试必须使用 PostgreSQL 16；外部 HTTP 使用录制脱敏样例或可控模拟器，仅证明适配器契约。真实审批、真实用户授权及业务模型标准答案单独列入 external，保存脱敏实例编号、时间和核验结果，不保存访问令牌。

AI评测固定保留集至少覆盖50条字段候选（含未知、冲突、跨年、近似数、注入及错误引用），由人工标注后冻结；合成与真实脱敏子集分开报告。统计字段precision/recall、引用定位正确率、关键字段错误率、审核修改率和耗时；不以模型自评分代替标注。严重错误（越权写入、伪造审批、未知补0进入正式测算）任一例失败即阻断。一般质量阈值和允许退化幅度由业务负责人基于首轮基线登记在evals/release-policy.yaml；阈值未确定时本地可报告，但M2真实AI发布门为BLOCKED。模型/提示词/解析器改变均重跑保留集，不能只更新预期答案以消除退化。

required清单必须提交 `acceptance/cases.yaml`：每条含id、package、min_phase、scope、test_path、dependency_ids、expected_evidence。P01只执行明确命名的AC子案例；M1必须包含P01–P04全部required local/contract项，M2包含P01–P07，M3包含全部。真实上线另要求本阶段external项。子案例SKIP/XFAIL不能映射PASS；禁止改清单躲过失败。

## 11. 运行、验证和报告命令契约

以下命令是交给 Codex 实现的目标接口，当前文档交付不表示已经执行成功。

| 命令 | 必须实现的行为 |
| --- | --- |
| `make bootstrap` | 检查 Python/uv/Docker Compose，`uv sync --frozen --all-groups`，启动本地 PG16，执行 Alembic；首次生成本地安全配置；不覆盖已有 `.env` |
| `make demo` | 单命令构建并启动本地 Compose，迁移、幂等演示数据、API 与 worker；默认绑定 `127.0.0.1:8000`，从P04起自动启动calc-runner，输出入口、演示身份和产物路径；不要求第三方密钥 |
| `make check` | Ruff 检查与格式校验、mypy、unit/integration 测试、迁移一致性检查；错误时非零退出 |
| `make acceptance PHASE=M1` | 在独立测试库运行指定阶段及其前置阶段所必需的 local/contract AC；支持 M0/M1/M2/M3；输出 JSON、Markdown 与 JUnit；未实现必需案例标 BLOCKED，不因 skip 改为 PASS |
| `make production-check PHASE=M3` | 按指定阶段只读验证 external 依赖及真实验收证据；未具备真实证据则 BLOCKED；不自动发通知或发起审批；需真实业务动作的检查另需显式测试项目、责任人和执行记录 |
| `docker compose up -d --build` | 标准启动入口；服务包括db、migrate、api、worker，从P04起还包括calc-runner，迁移完成后启动应用 |
| `uv run alembic upgrade head` | 从空数据库及上一发布版本升级；升级失败停止启动 |
| `uv run pytest tests/unit tests/integration` | 在专用测试库执行，不连接生产，失败非零退出 |

`artifacts/acceptance/<UTC时间>-<commit>/` 保存 `report.json、report.md、junit.xml、dependency-report.json` 及必要脱敏截图/文档；`artifacts/demo/` 保存演示输入、计算输出、DOCX/XLSX 和清单哈希。运行报告记录 `uv.lock` 哈希、迁移版本、适配器模式与命令退出码。每个阶段维护 required case 清单，汇总只计算本次所选阶段和前置阶段的必需案例；未选择的未来阶段仅列为未纳入范围，不写成 PASS。required 清单不得以“实现了哪些”动态缩减。汇总工具不得仅依据退出码把未执行 AC 标记为 PASS；存在 FAIL 返回 1，无 FAIL 但有 BLOCKED 返回 2，所有 required 案例通过返回 0。

证据文件不因存在就可信：production-check核对生成它的适配器模式、目标租户/测试项目、配置和代码版本、模型/模板摘要、责任人、时间及有效期；过期或配置变化后证据需要重新验证。该命令不调用会产生副作用的端点，不通过实际客户操作完成自测。M2真实身份与LLM验证、M3真实审批及业务模型验证分别对应external案例；缺少任何required证据返回2。

P01即实现reporter和cases.yaml的最小版本；后续任务包扩展case列表而非人工填PASS。首次执行命令的真实stdout/stderr及退出码保存在报告目录，秘密过滤后归档。本文编写阶段没有运行未来业务代码、没有执行真实审批或生产财务模型，不能把本文件的样例值当作验收证据。

## 12. 可直接发给Codex的执行指令

### 12.1 可直接复制给 Codex 的总任务

```text
请阅读 docs/logistics-presales-ai-system-design-v3.1.zh-CN.md 和 docs/logistics-presales-ai-technical-spec-v1.0.zh-CN.md，按两份文档实现本仓库。
先读取仓库 AGENTS.md、现有代码和依赖清单；业务边界以设计为准，具体实现以技术方案为准，发现矛盾按技术规范第0章的优先级处理，不扩大业务范围；无法解决的业务规则记录为依赖阻塞。
技术基线：Python 3.12 / FastAPI / Pydantic 2 / SQLAlchemy 2 / psycopg 3 / Alembic / PostgreSQL 16 / uv；Jinja2 加本地 JavaScript；模块化单体与 PG worker。
按 P01–P09 及依赖顺序执行。每包必须形成可运行的竖切闭环，并完成所对应 AC 的正反案例、迁移、页面、命令与证据。通过后自行进入下一包，无须逐包等待确认。
先实现本地可运行与可重复测试；外部缺失记录 BLOCKED，并继续不依赖该条件的工作。模拟提取/审批仅证明本地或适配器契约，不能作为真实生产成功证据；不要编造客户数据、领导审批、生产公式、接口权限或配置。
不扩大至 ST5 自动化、合同投标、自动客户发送、微服务、Redis/Celery、SQLite、向量库或插件平台。不要用 TODO/固定返回值代替应实现的业务路径。
保护用户已有改动，不覆盖秘密，不提交运行数据和凭据。实现及测试属于本次范围；生产部署、发送真实业务通知、发起真实审批和远端推送须依据会话已有授权执行，未授权时保留为清晰的待办，不妨碍本地工作。
每包更新 docs/implementation-status.md：实现范围、AC 状态、命令、证据路径、阻塞与下一步。最终提供 make demo、make check、make acceptance PHASE=<实际完成阶段> 结果和真实上线依赖，不以模拟通过声称生产完成。
```


### 12.2 首包启动任务

```text
现在执行 P01。先做环境预检并保存依赖报告，然后交付一个真实 PostgreSQL 支撑的浏览器闭环：开发身份登录→创建项目→创建独立拜访→上传 UTF-8 文本→fixture 生成明确标记的候选→人工确认→查看当前事实、历史和来源→生成可读参考草稿。
用“学生 1200 人”的固定脱敏材料验证正例；至少验证未经确认不成为事实、非成员不能读取项目、重复提交不重复采纳。fixture 仅模拟模型提取，项目、文件、审核、事务和事实历史必须真实实现。
先提供基础PG worker、事务内审计与幂等，再提供首版 Alembic 迁移、锁定依赖、.env.example、Compose、make demo 和对应 AC-01/02/03/04 子集；没有 LLM 或飞书密钥也必须能运行。只在开发模式开放演示身份；生产模式拒绝该身份设置。
运行验证，保存报告和产物；不要停在脚手架或空页面。P01 通过后按总任务继续 P02，无须等待确认；基础设施阻塞时保留证据并完成不受影响的实现。
```

## 13. V3.1需求与实现定位

| V3.1业务要求 | 本规范主要实现位置 | 任务包 / 验收 |
| --- | --- | --- |
| 单项目长期上下文、来源和历史 | 第3–5章事实/候选/快照/拜访 | P01–P03；AC-03–08 |
| 人机分工、关键审核、主动待办 | 第4、6、7章角色/命令/任务 | P01/P05/P06；AC-02/11–13 |
| ST1/ST2与合作方向 | 第4–5章接口/模板/分析能力 | P06；AC-13/15 |
| 真实双领导审批、材料审核、交接 | 第6章尽调与实例历史 | P08；AC-14/16/17 |
| 真实测算、情景继承、结果复核 | 第6、8章执行及模型协议 | P04/P09；AC-09/10/17 |
| 文档版本、依赖失效、正式发布 | 第3、5、6章快照/质量/发布 | P03/P09；AC-07/08/10 |
| 共享权限与任务隔离 | 第3、4、7章权限和认证 | P01/P07；AC-02/14/18 |
| 知识治理与受控AI | 第5章检索/候选/评测 | P02/P06；AC-03/13/15 |
| 单向项目索引、后台持续运行 | 第6章Outbox/队列/核对 | P05/P07；AC-11/12/14 |
| 分期实施、真实依赖与可验收成果 | 第9–12章任务包/命令/提示词 | P01–P09；AC-01–18 |

把完整API字段、类型和迁移放入代码真源；本文件是行为契约。若实施发现必须调整本文，先记录具体矛盾、最终决定及受影响验收，保持业务基线和技术规范同步，不在代码里静默改变关键门槛。
