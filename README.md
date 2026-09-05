# trae-api-test-skills

面向 TRAE 的 API 自动化测试 / FastAPI 工程技能包（Skills）。源自一套生产级金融 API 测试平台（pytest 命令行框架 + FastAPI/Vue3 可视化平台）的实战沉淀。

## Skills

### api-test-scaffold

让 AI 为**任意 HTTP 接口系统**一键生成分层 pytest 自动化测试框架。

- 分层架构 `api → steps → flows → testcases`，业务用例只需 3 行
- Token 自动刷新回调 + 鉴权失效（HTTP 401 / 业务码）自动重登重试
- YAML 模板覆写式数据驱动，按环境隔离（`config/env_{env}.yaml` + `data/{env}/`）
- 幂等 Flow 编排：用例只声明终态，前置链路自动补齐
- 可选 DB 落库断言（PyMySQL ping 保活重连）
- pytest-html 自包含报告 + 业务链路阶段 marker 过滤

### fastapi-fake-db-tests

让 AI 用**手写最小 FakeDb**（队列剧本编排查询结果）为 FastAPI/SQLAlchemy 写单测——不需要真实数据库、不启动应用、不用 TestClient。

- 域函数显式收 db + 手写替身 + 事件接缝（Sink Protocol）三件套
- `first_results` 按查询调用顺序排剧本，测试即文档
- 断言副作用计数与对象状态，拒绝 MagicMock
- 实战背书：37 个测试文件同风格、700+ 用例全绿

### arch-guard-tests

把架构规则写成 **pytest 守卫**：规则不再只活在 README 里，每次跑测试都被强制执行。

- 分层守卫：`inspect.getsource` 断言禁止反向 import（如 engine 不得依赖 routers）
- 防平行实现守卫：否定断言（禁止内联实现标识）+ 肯定断言（必须委托共享实现）
- 附 AST 升级版，不受注释/字符串误触发
- AI 辅助开发时代尤其有价值：约束 AI 不越层、不复制平行实现

### alembic-smooth-migrations

多环境长期演进项目的 **Alembic 实战约定**，全是踩坑沉淀。

- 顺序 hex revision 链 + `<动词>_<对象>` 命名，迁移史人肉可读
- NOT NULL 新列必带 `server_default`、Column 必写业务 comment、downgrade 必写
- env.py 三约定：复用应用 engine、`compare_type=True`、`disable_existing_loggers=False`（防日志静默消失）
- `init_db()` 启动自愈三分支（旧库 stamp / 全新库 upgrade），git pull 重启即迁移

## 安装

把想要的 skill 目录整个复制到你的项目下：

```
你的项目/.trae/skills/<skill-name>/
```

可按需安装任意一个，互不依赖。

## 使用

在 TRAE 中对 AI 直接描述任务，对应技能会自动触发：

- "帮我给 XX 系统搭一套接口自动化测试框架，base_url 是 https://xxx……" → `api-test-scaffold`
- "给这个 crud 模块写单元测试，不要连数据库" → `fastapi-fake-db-tests`
- "把我们项目的分层规则固化成守卫测试，防止 AI 越层 import" → `arch-guard-tests`
- "新增一个迁移：订单表加 xx 字段" → `alembic-smooth-migrations`

## License

MIT
