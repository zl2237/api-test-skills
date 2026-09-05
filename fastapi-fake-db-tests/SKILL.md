---
name: "fastapi-fake-db-tests"
description: "Writes FastAPI/SQLAlchemy unit tests with a hand-rolled FakeDb (queue-scripted query results) — no real DB, no TestClient. Invoke when writing unit tests for crud/service/domain functions."
---

# fastapi-fake-db-tests 无数据库单测模式

不依赖真实数据库、不启动应用、不用 TestClient，直接测 FastAPI 的**业务不变量**（crud/service 域函数）。适用于任何 SQLAlchemy 项目。相比 SQLite 内存库 / dependency_overrides：无方言差异、无建表开销、测试即文档。

## 核心三件套（缺一不可）

1. **域函数显式收 db**：router 只留 HTTP 语义（状态码/鉴权），业务不变量（唯一性、最后管理员保护、审计字段）全部收敛到 crud/service 域函数，第一个参数为 `db`
2. **手写最小 FakeDb**：只实现被测函数用到的 Session 方法子集，`first()` 按队列消费预设结果
3. **事件接缝（有副作用的服务才需要）**：副作用（写执行记录、发通知）通过 Protocol 接缝注入，测试传内存实现

## FakeDb 模板

```python
class FakeDb:
    """SQLAlchemy Session 最小替身：first() 按队列消费；all() 返回空列表；count() 返回预设值"""

    def __init__(self, first_results=(), count_value=2):
        self._queue = list(first_results)   # 按 query().first() 调用顺序依次弹出
        self.count_value = count_value      # query().count() 的固定返回
        self.added = []                     # 记录 add() 的对象
        self.committed = 0                  # commit() 次数

    def query(self, *_a, **_kw):
        db = self

        class _Q:                            # 链式查询替身：吞掉任意 filter/filter_by
            def filter(self, *a, **kw):
                return self

            def filter_by(self, **kw):
                return self

            def first(self):
                return db._queue.pop(0) if db._queue else None

            def count(self):
                return db.count_value

            def all(self):
                return []

        return _Q()

    def add(self, obj):
        self.added.append(obj)

    def commit(self):
        self.committed += 1

    def refresh(self, _obj):
        pass

    def rollback(self):
        pass
```

## 测试编写规则

- **first_results 是剧本**：按 `query().first()` 的调用顺序排列结果，必须写注释说明每次查询的语义（如"第 1 次：username 查重 → None"）
- **断言副作用计数与对象状态**（`db.added` / `db.committed >= 1` / 字段被改写），不使用 mock 的 assert_called
- 请求对象用 `SimpleNamespace`，不必构造 Pydantic 模型
- 业务异常用 `pytest.raises(HTTPException)` 捕获，断言 `status_code` 与 `detail` 关键词

```python
from types import SimpleNamespace

import pytest
from fastapi import HTTPException

from app.crud import users as users_domain


def _admin():
    return SimpleNamespace(id=9, username="boss", role="admin")


class TestCreateUser:
    def test_duplicate_username_raises_400(self):
        # 剧本：第 1 次 query().first()（查重）返回已存在用户
        db = FakeDb(first_results=[_make_user(username="taken")])
        req = SimpleNamespace(username="taken", password="abc12345", role="member")
        with pytest.raises(HTTPException) as e:
            users_domain.create_user(db, req, operator=_admin())
        assert e.value.status_code == 400

    def test_create_hashes_password(self):
        # 剧本：第 1 次 query().first()（查重）返回 None
        db = FakeDb(first_results=[None])
        req = SimpleNamespace(username="newbie", password="abc12345", role="member")
        user = users_domain.create_user(db, req, operator=_admin())
        assert user.password_hash != "abc12345"      # 哈希而非明文
        assert db.added and db.committed >= 1        # 副作用断言
```

## 变体扩展：内嵌子类按需补方法

被测路径用到 FakeDb 没有的方法（`merge` 等）时，在测试内定义最小子类，不要把所有方法都塞进基类：

```python
def test_merge_path(self):
    class MergeableDb(FakeDb):
        def merge(self, obj):
            return obj

    db = MergeableDb(first_results=[None])
    ...
```

## 配套替身（与 FakeDb 同风格）

```python
class MemorySink:
    """事件接缝的内存实现：收集 StepResult，断言事件序列"""
    def __init__(self):
        self.events = []

    def record_step(self, result):
        self.events.append(result)


class StubHttpClient:
    """HTTP 客户端替身：记录最后请求体，返回固定响应"""
    def __init__(self, resp=None):
        self.last_json_body = None
        self._resp = resp or {"code": 200}

    def post(self, path, body=None, **kw):
        self.last_json_body = body
        return self._resp
```

事件接缝的生产侧设计（让执行主链路脱离 Session 可测）：

```python
from typing import Protocol


class StepSink(Protocol):
    def record_step(self, result: "StepResult") -> None: ...

# 生产用 DbSink（写库），测试/dry-run 用 MemorySink
```

## conftest 要点

模块加载期（`import app` 触发 database/auth 模块）就要的环境变量，必须在 conftest **顶层**设置（不是 fixture 里）：

```python
import os
import sys
from pathlib import Path

os.environ.setdefault("DB_PASSWORD", "test-only")
os.environ.setdefault("JWT_SECRET_KEY", "test-only")

sys.path.insert(0, str(Path(__file__).resolve().parent.parent))
```

## 铁律

- 不用 MagicMock 代替手写替身——剧本的可读性是本模式的核心价值
- 不为可测性在业务代码里加 `if TEST` / 环境判断；改不动时先造接缝（域函数抽离、Sink Protocol）
- HTTP 层行为（鉴权、路由状态码）留给少量 TestClient 集成测试，不混入本模式
- FakeDb 只实现被测路径用到的方法，宁可子类扩展也不预埋全家桶
