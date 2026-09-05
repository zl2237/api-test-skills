---
name: "arch-guard-tests"
description: "Turns architecture rules into pytest guards via source/AST assertions: no reverse layer imports, must delegate to shared implementations. Invoke when enforcing layering or reviewing AI-generated code."
---

# arch-guard-tests 架构守卫测试

把架构规则写成 pytest：规则不再只活在 README 里，每次跑测试都被强制执行。对 AI 辅助开发尤其重要——AI 最常犯的架构错误是**越层 import**（如 engine 反向依赖 routers）和**复制一份平行实现**（内联一段本应复用的逻辑），守卫测试让这两类错误在提交前暴露。

## 第一步：向用户收集规则

1. **分层规则**：谁不允许 import 谁（如 `engine` 不得 import `routers`；`routers` 不得直接写 ORM 查询）
2. **单实现点**：哪些能力只允许一份实现（如 HTTP 请求发送、multipart 组装、报告导出）
3. **委托要求**：哪些模块必须复用共享实现而非内联（如 debug 接口必须委托 `services.request_sender`）

## 守卫模板

### 1. 分层守卫（禁止反向依赖）

```python
import inspect

import app.engine.dag_executor as de


class TestEngineLayering:
    def test_engine_does_not_import_routers(self):
        """分层约束：engine 不得依赖 routers 层"""
        src = inspect.getsource(de)
        assert "from ..routers" not in src, "engine 反向依赖 routers，应改走 services 层"
```

### 2. 防平行实现守卫（否定断言 + 肯定断言）

否定断言找"内联实现的特征标识"（低层 API 名、被禁的私有函数名），肯定断言要求存在委托 import：

```python
import inspect

from app.routers import apis
from app.engine import dag_executor


class TestNoParallelImplementations:
    """架构守卫：请求发送只允许一份实现（services.request_sender）"""

    def test_debug_api_reuses_shared_sender(self):
        src = inspect.getsource(apis)
        assert "post_multipart" not in src, "debug 接口内联了 multipart 组装，应复用 services.request_sender"
        assert "from ..services.request_sender import send_request" in src

    def test_dag_executor_delegates_to_shared_sender(self):
        src = inspect.getsource(dag_executor)
        assert "from ..services.request_sender import send_request" in src
        assert "_build_multipart_files" not in src, "仍持有私有 multipart 组装，应删除并委托"
```

### 3. 整包批量守卫（遍历目录所有文件）

```python
import inspect
from pathlib import Path

import app.engine as engine_pkg


def test_engine_package_never_imports_routers():
    """分层约束：engine 包任何模块不得 import routers"""
    pkg_dir = Path(inspect.getfile(engine_pkg)).parent
    for py in pkg_dir.rglob("*.py"):
        src = py.read_text(encoding="utf-8")
        assert "from ..routers" not in src and "from app.routers" not in src, (
            f"{py.name} 反向依赖 routers，应改走 services 层"
        )
```

## 编写规则

- 守卫放在独立 `tests/test_arch_guards.py`，或挂在相关模块测试文件的 `TestXxxLayering` / `TestNoParallelImplementations` 类中
- **失败消息必须写修复指引**（"应改走 services 层" / "应删除并委托"），守卫失败即架构评审意见
- 每条守卫对应 README 分层规约的一条，文档与测试一一对应
- 规则新增时机：每次 code review 拦下一次越层/平行实现，就固化一条守卫

## 升级：AST 版（不受注释/字符串误触发）

子串断言会被注释、字符串字面量误触发（误报不漏报，通常可接受）；要求更严时用 AST：

```python
import ast
from pathlib import Path


def _imported_modules(tree: ast.AST) -> set[str]:
    mods = set()
    for node in ast.walk(tree):
        if isinstance(node, ast.ImportFrom) and node.module:
            mods.add(node.module)
        elif isinstance(node, ast.Import):
            mods.update(a.name for a in node.names)
    return mods


def test_engine_never_imports_routers_ast():
    pkg_dir = Path(__file__).resolve().parent.parent / "app" / "engine"
    for py in pkg_dir.rglob("*.py"):
        tree = ast.parse(py.read_text(encoding="utf-8"))
        bad = [m for m in _imported_modules(tree)
               if m.startswith("app.routers") or m.endswith(".routers")]
        assert not bad, f"{py.name} 反向依赖 routers：{bad}"
```

## 局限（如实告知用户）

- 只覆盖静态 import，跨文件动态引用（importlib、字符串导入）不覆盖
- 单文件源码断言看不出运行时行为，守卫是"架构约束"不是"行为测试"
- 守卫数量与规则一一对应即可，不要为不存在风险的层写守卫
