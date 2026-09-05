---
name: "alembic-smooth-migrations"
description: "Applies battle-tested Alembic conventions: ordered revision chains, verb_object naming, motive docstrings, server_default on NOT NULL columns, shared env.py, self-healing init_db(). Invoke when creating migrations or setting up Alembic."
---

# alembic-smooth-migrations Alembic 迁移规范与自愈

多环境长期演进项目的 Alembic 实战约定：迁移链人肉可读、新列不炸存量数据、部署无需人工迁移。

## 迁移文件规范

- **文件名 = `<revision>_<动词>_<对象>.py`**：`add_user_avatar_column` / `drop_file_tags` / `env_add_success_codes`，一文件一变更，小步快跑
- **revision 用手工编排的 12 位顺序 hex 链**：`a3b4c5d6e7f8 → b4c5d6e7f8a9 → c5d6e7f8a9b0`（首字母顺延），既满足 alembic 的 hex 格式又让链序肉眼可读。生成时用 `--rev-id` 指定：

```bash
alembic revision --rev-id b4c5d6e7f8a9 -m "add_wait_after_ms"
```

- **docstring 写变更动机**（为什么加、行为预期），不写模板废话：

```python
"""add wait_after_ms to case_node_configs

新增 wait_after_ms 字段：
- 当前节点执行完成后，到下一节点请求前的等待毫秒数
- 默认 0（立即执行），用于给后端事务落库留时间，避免下游读到未提交数据
"""


def upgrade() -> None:
    op.add_column(
        "case_node_configs",
        sa.Column("wait_after_ms", sa.Integer(), nullable=False,
                  server_default="0",
                  comment="节点执行完后到下一请求的等待毫秒数，默认0"),
    )


def downgrade() -> None:
    op.drop_column("case_node_configs", "wait_after_ms")
```

## upgrade 写法铁律

- **新增 NOT NULL 列必须带 `server_default`**——否则存量行在 `ALTER TABLE` 时直接失败
- Column 必写 `comment`（业务含义），查表结构即文档
- `downgrade` 必须写（回滚是迁移的一部分），逐列/逐表 drop 或 create 的逆操作
- 手写优先：`alembic revision -m "verb_object"`；autogenerate 仅作参考，生成后必须逐行人工检查

## env.py 三条约定

```python
from logging.config import fileConfig

from alembic import context

config = context.config

# 1. disable_existing_loggers=False：否则 fileConfig 会禁用 uvicorn 等已创建的 logger，
#    应用 startup 调 init_db() 后 uvicorn access log 静默消失（高频踩坑）
if config.config_file_name is not None:
    fileConfig(config.config_file_name, disable_existing_loggers=False)

import os, sys
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

# 2. 复用应用的 engine 与 Base.metadata（连接配置单一事实源，不读 alembic.ini 的 url）
from app.database import engine, Base
from app import models  # noqa: F401  触发所有模型注册到 Base.metadata

target_metadata = Base.metadata
config.set_main_option("sqlalchemy.url", str(engine.url))


def run_migrations_online() -> None:
    with engine.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,   # 3. 列类型变更可被 autogenerate 检出
        )
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    # offline 模式同样带 compare_type=True
    ...
else:
    run_migrations_online()
```

## init_db() 启动自愈（三分支）

应用 startup 时调用，达到"git pull 后重启即迁移"，无需人工干预：

```python
def init_db():
    """智能 Alembic 迁移：
    - 旧库（有表无 alembic_version 表）：stamp head，标记 schema 已到位，不执行 DDL
    - 全新库 / 已迁移库：upgrade head，建表或应用增量迁移
    """
    from pathlib import Path

    from alembic import command
    from alembic.config import Config
    from sqlalchemy import inspect

    from . import models  # noqa: F401  触发模型注册

    alembic_cfg = Config(str(Path(__file__).parent.parent / "alembic.ini"))
    existing_tables = set(inspect(engine).get_table_names())

    if existing_tables and "alembic_version" not in existing_tables:
        command.stamp(alembic_cfg, "head")      # 旧库收编：只打标记不动表
    else:
        command.upgrade(alembic_cfg, "head")
```

关键点：`stamp` 分支让"已有历史表但从未纳入 alembic 管理"的库安全接入，不会重复建表。

## 提交前验证清单

1. 干净库上 `alembic upgrade head` 跑通，再 `alembic downgrade -1` / `upgrade head` 验证回滚
2. `alembic history` 链条连续、revision 命名符合顺序 hex 约定
3. CI/部署：deploy 阶段执行 `alembic upgrade head`；若应用有 init_db() 自愈，重启即可

## 常见坑

- 忘了 `server_default` 的 NOT NULL 新列 → 存量数据迁移失败
- env.py 用 alembic.ini 的 url 而非应用 engine → 测试/生产连接配置漂移
- `disable_existing_loggers` 默认 True → 集成后日志静默消失，极难排查
- 一个迁移文件塞多个不相关变更 → 回滚粒度失控，永远一文件一变更
