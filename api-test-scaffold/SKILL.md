---
name: "api-test-scaffold"
description: "Generates a layered pytest API automation framework (api/step/flow/testcase, YAML data-driven, env switching, auto-relogin, DB assert). Invoke when asked to set up API tests for any HTTP system."
---

# api-test-scaffold 分层 API 自动化测试框架生成器

为任意 HTTP 接口系统生成一套**分层 pytest 自动化测试框架**。核心卖点：

- 分层架构：`api → steps → flows → testcases`，业务用例只需 3 行
- YAML 模板覆写式数据驱动，按环境隔离（`config/env_{env}.yaml` + `data/{env}/`）
- Token 自动刷新回调 + 鉴权失效（HTTP 401 / 业务码）自动重登重试一次
- 可选 DB 落库断言（PyMySQL ping 保活重连）
- 幂等 Flow 编排：用例只声明终态，前置链路自动补齐
- pytest-html 自包含报告 + 业务链路阶段 marker 过滤

## 第零步：向用户确认（缺一项就先问，不要猜）

1. **base_url** 与鉴权方式：登录接口 URL、token 放哪个请求头（如 `Authorization: Bearer xxx`）、token 在登录响应中的位置
2. **业务成功码**：成功时 `resp["code"]` 的值（200？0？"0"？）→ 写入 `success_codes`
3. **鉴权失效码**：token 过期/异地登录时 HTTP 状态码与业务码 → 写入 `auth_failure_codes`
4. **是否需要 DB 落库校验**：不需要则跳过 `db/` 层与 pymysql 依赖
5. **首个业务模块**：模块名、接口清单、链路先后依赖、动态字段（哪个单号需要唯一生成）

## 生成的目录结构

```
project/
├── api/
│   ├── base_api.py            # API 顶层父类
│   ├── auth_api.py            # 登录接口（供 conftest 获取 token）
│   └── {module}/{module}_api.py
├── db/                        # 可选（无落库校验不生成）
│   ├── db_client.py           # PyMySQL 封装：ping 保活、DictCursor
│   ├── base_db.py             # DB 业务顶层父类
│   └── biz/{module}_db.py
├── steps/{module}/{module}_step.py
├── flows/{module}/{module}_flow.py
├── testcases/{module}/test_{module}.py
├── data/{env_name}/{module}/*.yaml
├── config/env_{env_name}.yaml
├── utils/
│   ├── http_client.py         # 核心：会话复用+自动重登+成功码校验+脱敏日志
│   ├── api_factory.py         # API 实例懒加载缓存 + 全局 header 同步
│   ├── db_factory.py          # DB 业务实例懒加载缓存
│   ├── assert_util.py         # 中文消息断言函数
│   ├── yaml_util.py
│   ├── generator_util.py      # 唯一单号/UUID 生成器
│   ├── exceptions.py          # 领域异常分类
│   ├── log_util.py
│   └── common_util.py         # 项目根定位
├── conftest.py                # session 级 fixtures：环境/工厂/登录
├── pytest.ini
└── requirements.txt
```

Python 3.10+。

## 基建代码（原样生成，标注【适配点】处按被测系统改）

### requirements.txt

```
pytest>=7.4
requests
PyYAML
pytest-html
pymysql   # 无 DB 校验需求时删除
```

### config/env_test.yaml

```yaml
env_name: test
base_url: https://api.example.com
common_headers:
  Content-Type: application/json
success_codes: [200]        # 【适配点】业务成功码集合，字符串码加引号如 ["0"]
auth_failure_codes: [401]   # 【适配点】业务鉴权失效码（token过期/异地登录专用码）
account:                    # 登录账号，conftest 用
  username: demo
  password: "******"
mysql:                      # 【适配点】无 DB 校验需求整段删除
  host: 127.0.0.1
  port: 3306
  user: root
  password: "******"
  database: demo
```

### utils/exceptions.py

```python
class HttpStatusError(Exception):
    """HTTP 状态码非 200"""


class HttpTimeoutError(Exception):
    """请求超时"""


class AuthError(Exception):
    """鉴权失效且无法自动恢复"""


class JsonParseError(Exception):
    """响应体不是合法 JSON"""


class BusinessError(Exception):
    """业务 code 不在成功码集合内"""

    def __init__(self, message: str, resp_json: dict | None = None):
        super().__init__(message)
        self.resp_json = resp_json  # 携带完整响应便于排查


class DBQueryError(Exception):
    """数据库查询异常"""
```

### utils/http_client.py（框架核心）

```python
import json
import logging

import requests

from utils.exceptions import (
    AuthError, BusinessError, HttpStatusError, HttpTimeoutError, JsonParseError,
)

logger = logging.getLogger("api_auto")


class HttpClient:
    """基于 requests.Session 的 HTTP 客户端。

    特性：连接复用 / 成功码集合校验 / 鉴权失效自动刷新 Token 并重试一次 / 日志脱敏。
    """

    def __init__(self, base_url: str, headers: dict | None = None,
                 success_codes=(200,), auth_failure_codes=(401,),
                 token_refresh_callback=None, timeout=10):
        self.base_url = base_url.rstrip("/")
        self.session = requests.Session()
        self.headers = headers or {}
        self.success_codes = set(success_codes)
        self.auth_failure_codes = set(auth_failure_codes)
        self.token_refresh_callback = token_refresh_callback
        self.timeout = timeout

    def update_header(self, key: str, value: str):
        self.headers[key] = value

    def get(self, path: str, params: dict | None = None) -> dict:
        return self._request("GET", path, params=params)

    def post(self, path: str, body: dict | None = None) -> dict:
        return self._request("POST", path, body=body)

    def post_form(self, path: str, data: dict | None = None) -> dict:
        return self._request("POST", path, data=data)

    def _request(self, method: str, path: str, retry_401: bool = True, **kwargs) -> dict:
        url = f"{self.base_url}{path}"
        self._log_request(method, url, kwargs)
        try:
            resp = self.session.request(
                method, url, headers=self.headers, timeout=self.timeout, **kwargs
            )
        except requests.Timeout as e:
            raise HttpTimeoutError(f"请求超时：{url}") from e

        # 鉴权失效判定一：HTTP 401
        if resp.status_code == 401 and retry_401:
            return self._relogin_and_retry(method, path, **kwargs)

        if resp.status_code != 200:
            raise HttpStatusError(f"HTTP状态码异常：{resp.status_code} url={url}")

        try:
            resp_json = resp.json()
        except ValueError as e:
            raise JsonParseError(f"响应非JSON：{resp.text[:500]}") from e

        code = resp_json.get("code")
        # 鉴权失效判定二：业务码（如异地登录/token过期的专用码）
        if code in self.auth_failure_codes and retry_401:
            return self._relogin_and_retry(method, path, **kwargs)
        if code not in self.success_codes:
            raise BusinessError(f"业务code异常：{code}", resp_json=resp_json)

        return resp_json

    def _relogin_and_retry(self, method: str, path: str, **kwargs) -> dict:
        """刷新 Token 后重试原请求一次；retry_401=False 防死循环"""
        if self.token_refresh_callback is None:
            raise AuthError("鉴权失效且未配置 token_refresh_callback")
        self.token_refresh_callback()
        return self._request(method, path, retry_401=False, **kwargs)

    def _log_request(self, method: str, url: str, kwargs: dict):
        safe_headers = {
            k: ("******" if k.lower() in ("authorization", "token", "cookie") else v)
            for k, v in self.headers.items()
        }
        body = kwargs.get("body") or kwargs.get("data")
        logger.info("HTTP %s %s headers=%s body=%s", method, url, safe_headers,
                    json.dumps(body, ensure_ascii=False)[:2000] if body else None)
```

### utils/api_factory.py

```python
from utils.http_client import HttpClient


class ApiFactory:
    """API 实例工厂：按类懒加载缓存；token 回调统一注册；全局 header 同步刷新。"""

    def __init__(self, base_url: str, common_headers: dict,
                 success_codes=(200,), auth_failure_codes=(401,)):
        self._base_url = base_url
        self._common_headers = dict(common_headers)
        self._success_codes = set(success_codes)
        self._auth_failure_codes = set(auth_failure_codes)
        self._token_refresh_callback = None
        self._apis: dict[type, object] = {}

    def get_api(self, api_cls: type):
        if api_cls not in self._apis:
            client = HttpClient(
                base_url=self._base_url,
                headers=dict(self._common_headers),
                success_codes=self._success_codes,
                auth_failure_codes=self._auth_failure_codes,
                token_refresh_callback=self._token_refresh_callback,
            )
            self._apis[api_cls] = api_cls(client)
        return self._apis[api_cls]

    def set_token_refresh_callback(self, callback):
        """登录 fixture 中调用；同步绑定到所有已缓存实例"""
        self._token_refresh_callback = callback
        for api in self._apis.values():
            api.http.token_refresh_callback = callback

    def update_global_header(self, key: str, value: str):
        """登录/换 Token 后同步刷新工厂与所有已缓存实例"""
        self._common_headers[key] = value
        for api in self._apis.values():
            api.http.update_header(key, value)
```

### db/db_client.py（可选，需要落库校验时生成）

```python
import pymysql


class DBClient:
    """PyMySQL 封装：每次操作前 ping 保活重连，DictCursor，autocommit。"""

    def __init__(self, conf: dict):
        self._conf = conf
        self._conn = self._connect()

    def _connect(self):
        return pymysql.connect(
            host=self._conf["host"], port=self._conf.get("port", 3306),
            user=self._conf["user"], password=self._conf["password"],
            database=self._conf["database"], charset="utf8mb4",
            cursorclass=pymysql.cursors.DictCursor, autocommit=True,
        )

    def _cursor(self):
        self._conn.ping(reconnect=True)
        return self._conn.cursor()

    def query(self, sql: str, args=None) -> list[dict]:
        with self._cursor() as cur:
            cur.execute(sql, args)
            return cur.fetchall()

    def query_one(self, sql: str, args=None) -> dict | None:
        with self._cursor() as cur:
            cur.execute(sql, args)
            return cur.fetchone()

    def execute(self, sql: str, args=None) -> int:
        with self._cursor() as cur:
            return cur.execute(sql, args)

    def close(self):
        self._conn.close()
```

### utils/db_factory.py（可选）

```python
from db.db_client import DBClient


class DbFactory:
    """DB 业务实例工厂：按类懒加载缓存，会话结束统一关闭连接。"""

    def __init__(self, mysql_conf: dict):
        self._client = DBClient(mysql_conf)
        self._dbs: dict[type, object] = {}

    def get_db(self, db_cls: type):
        if db_cls not in self._dbs:
            self._dbs[db_cls] = db_cls(self._client)
        return self._dbs[db_cls]

    def close_all(self):
        self._client.close()
```

### api/base_api.py 与 db/base_db.py

```python
# api/base_api.py
from utils.http_client import HttpClient


class BaseApi:
    def __init__(self, http_client: HttpClient):
        self.http = http_client
```

```python
# db/base_db.py
from db.db_client import DBClient


class BaseDB:
    def __init__(self, db_client: DBClient):
        self.db = db_client
```

### api/auth_api.py

```python
from api.base_api import BaseApi


class AuthApi(BaseApi):
    def login(self, account: dict) -> dict:
        """【适配点】登录接口 URL 与请求体结构按实际系统调整"""
        return self.http.post("/api/login", body=account)
```

### utils/assert_util.py

```python
def equal(actual, expected, message=""):
    assert actual == expected, f"{message} 【期望值】{expected} 【实际值】{actual}"


def not_equal(actual, expected, message=""):
    assert actual != expected, f"{message}失败 【实际值】{actual}（不应等于{expected}）"


def is_not_empty(value, message=""):
    assert value not in (None, "", [], {}), f"{message}失败 【实际值】{value}"


def contains(container, member, message=""):
    assert member in container, f"{message}失败 【期望包含】{member}"


def key_exists(data: dict, key, message=""):
    assert key in data, f"{message}失败 【缺少字段】{key}"
```

按需同风格扩展 in_list / is_empty / greater 等。

### utils/yaml_util.py

```python
from pathlib import Path

import yaml


def read_yaml(path: str | Path) -> dict:
    with open(path, encoding="utf-8") as f:
        return yaml.safe_load(f)


def write_yaml(path: str | Path, data: dict):
    path = Path(path)
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        yaml.dump(data, f, allow_unicode=True, sort_keys=False)
```

### utils/generator_util.py

```python
import time
import uuid
from random import Random

_r = Random()


def generate_no(prefix: str = "auto") -> str:
    """业务唯一单号：前缀+时间戳+4位随机码，长度不超过32"""
    suffix = "".join(_r.choices("ABCDEFGHJKMNPQRSTUVWXYZ23456789", k=4))
    return f"{prefix}{time.strftime('%Y%m%d%H%M%S')}{suffix}"


def generate_unique_id() -> str:
    """通用唯一ID（如费用行配对关联）"""
    return uuid.uuid4().hex
```

### utils/common_util.py

```python
from pathlib import Path


def get_project_root() -> Path:
    """以含 pytest.ini 的最近父目录作为项目根"""
    for parent in Path(__file__).resolve().parents:
        if (parent / "pytest.ini").exists():
            return parent
    return Path(__file__).resolve().parent.parent
```

### utils/log_util.py

```python
import logging
import sys
from pathlib import Path


def setup_logger(name: str = "api_auto", log_dir: str = "logs") -> logging.Logger:
    logger = logging.getLogger(name)
    if logger.handlers:
        return logger
    logger.setLevel(logging.INFO)
    fmt = logging.Formatter("%(asctime)s [%(levelname)s] %(message)s")
    sh = logging.StreamHandler(sys.stdout)
    sh.setFormatter(fmt)
    logger.addHandler(sh)
    Path(log_dir).mkdir(exist_ok=True)
    fh = logging.FileHandler(Path(log_dir) / "run.log", encoding="utf-8")
    fh.setFormatter(fmt)
    logger.addHandler(fh)
    return logger
```

### conftest.py（token 自动刷新的装配点）

```python
import os
import threading

import pytest

from api.auth_api import AuthApi
from utils.api_factory import ApiFactory
from utils.common_util import get_project_root
from utils.db_factory import DbFactory
from utils.log_util import setup_logger
from utils.yaml_util import read_yaml


@pytest.fixture(scope="session", autouse=True)
def _logger():
    setup_logger()


@pytest.fixture(scope="session")
def env_config() -> dict:
    env = os.getenv("TEST_ENV", "test")
    config = read_yaml(get_project_root() / f"config/env_{env}.yaml")
    config["env_name"] = env
    return config


@pytest.fixture(scope="session")
def api_factory(env_config) -> ApiFactory:
    return ApiFactory(
        base_url=env_config["base_url"],
        common_headers=env_config.get("common_headers", {}),
        success_codes=env_config.get("success_codes", [200]),
        auth_failure_codes=env_config.get("auth_failure_codes", [401]),
    )


@pytest.fixture(scope="session")
def login_token(api_factory, env_config):
    """登录并注册 Token 自动刷新回调（加锁防并发重复登录）"""
    lock = threading.Lock()

    def refresh_token():
        with lock:
            auth_api = api_factory.get_api(AuthApi)
            resp = auth_api.login(env_config["account"])
            # 【适配点】token 在响应中的位置按实际系统调整
            token = resp["data"]["token"]
            assert token, "登录成功但未获取到token"
            # 【适配点】请求头名称按实际系统调整
            api_factory.update_global_header("Authorization", f"Bearer {token}")

    refresh_token()
    api_factory.set_token_refresh_callback(refresh_token)
    return refresh_token


@pytest.fixture(scope="session", autouse=True)
def auto_login(login_token):
    """整个会话开始即完成全局登录"""


@pytest.fixture(scope="session")
def db_factory(env_config):
    mysql_conf = env_config.get("mysql")
    if not mysql_conf:
        pytest.skip("未配置mysql，跳过DB相关fixture")
    factory = DbFactory(mysql_conf)
    yield factory
    factory.close_all()
```

## 业务模块五层模板

模块英文名 `{module}`（snake_case），类名 PascalCase `{Module}`。以两个阶段为例：

### api/{module}/{module}_api.py（禁止断言）

```python
from api.base_api import BaseApi


class {Module}Api(BaseApi):
    def {action1}(self, req_body: dict) -> dict:
        """
        {动作1中文}接口
        注意：与其他接口共用同一 URL 时，通过请求体区分业务语义
        :return: 接口原始响应dict
        """
        return self.http.post("{url1}", body=req_body)
```

### db/biz/{module}_db.py（可选；SQL 统一放这里，禁止在 Flow 裸写）

```python
from db.base_db import BaseDB


class {Module}DB(BaseDB):
    def query_by_no(self, no: str) -> dict | None:
        """根据业务单号查单条记录"""
        sql = """
            SELECT *
            FROM {table} WHERE {no_col} = %s
        """
        return self.db.query_one(sql, args=[no])
```

### steps/{module}/{module}_step.py（禁止读 yaml；固定"调API+断言code"）

```python
from api.{module}.{module}_api import {Module}Api
from utils.assert_util import equal


class {Module}Step:
    def __init__(self, {module}_api: {Module}Api):
        self.{module}_api = {module}_api

    def {action1}(self, req_body: dict) -> dict:
        """原子步骤：{动作1中文}，内置硬性断言，失败终止链路"""
        resp = self.{module}_api.{action1}(req_body)
        equal(resp["code"], 200, "{动作1中文}：业务code不等于200")  # 与 success_codes 一致
        return resp
```

### flows/{module}/{module}_flow.py（幂等推进，框架精髓）

```python
import copy
import logging
from typing import Optional

from api.{module}.{module}_api import {Module}Api
from db.biz.{module}_db import {Module}DB
from steps.{module}.{module}_step import {Module}Step
from utils.assert_util import equal, is_not_empty
from utils.common_util import get_project_root
from utils.generator_util import generate_no
from utils.yaml_util import read_yaml


class {Module}Flow:
    """{模块中文名}链路编排器：每个阶段自动执行前置阶段（幂等），用例只声明终态。"""

    def __init__(self, api_factory, db_factory, env_config: dict):
        self.{module}_api: {Module}Api = api_factory.get_api({Module}Api)
        self.{module}_db: {Module}DB = db_factory.get_db({Module}DB)
        self.{module}_step = {Module}Step(self.{module}_api)
        self.env_name: str = env_config.get("env_name", "test")

        # 链路状态：随阶段推进填充，供后续阶段与用例复用
        self.biz_no: Optional[str] = None
        self.biz_id: Optional[int] = None

        # 阶段执行标记（幂等控制）
        self._{stage1}_done = False
        self._{stage2}_done = False

    def _load_yaml(self, filename: str) -> dict:
        """读取当前环境 yaml，必须深拷贝返回，防止模板被污染"""
        data_path = get_project_root() / f"data/{self.env_name}/{module}/{filename}"
        return copy.deepcopy(read_yaml(data_path))

    def {stage1}(self) -> dict:
        """{阶段1中文}。已执行过则跳过。"""
        if self._{stage1}_done:
            return {}

        body = self._load_yaml("{stage1}.yaml")
        # 动态字段注入：模板置 null 的字段在此生成或查库回填
        if body["biz_no"] is None:
            body["biz_no"] = generate_no(prefix="auto")
        self.biz_no = body["biz_no"]

        resp = self.{module}_step.{action1}(body)

        record = self.{module}_db.query_by_no(self.biz_no)
        is_not_empty(record, f"数据库未查询到：{self.biz_no}")
        self.biz_id = record["id"]
        equal(record["status"], 1, "{阶段1中文}：状态不一致")
        logging.info("{阶段1中文}成功")

        self._{stage1}_done = True
        return resp

    def {stage2}(self) -> dict:
        """{阶段2中文}，自动执行前置。已执行过则跳过。"""
        self.{stage1}()
        if self._{stage2}_done:
            return {}

        body = self._load_yaml("{stage2}.yaml")
        body["biz_id"] = self.biz_id  # 前置阶段查库回填的状态注入模板

        resp = self.{module}_step.{action2}(body)

        record = self.{module}_db.query_by_no(self.biz_no)
        equal(record["status"], 2, "{阶段2中文}：状态不一致")
        logging.info("{阶段2中文}成功")

        self._{stage2}_done = True
        return resp
```

阶段方法结构固定：**调前置 → 幂等检查 → 读yaml深拷贝 → 注入动态字段 → 调Step → DB断言 → 置done标记**。

### testcases/{module}/test_{module}.py（每用例 3 行）

```python
import pytest

from flows.{module}.{module}_flow import {Module}Flow


@pytest.mark.{stage1}
def test_{stage1}(api_factory, db_factory, env_config):
    """{阶段1中文}"""
    flow = {Module}Flow(api_factory, db_factory, env_config)
    flow.{stage1}()


@pytest.mark.{stage2}
def test_{stage2}(api_factory, db_factory, env_config):
    """{阶段1中文} → {阶段2中文}"""
    flow = {Module}Flow(api_factory, db_factory, env_config)
    flow.{stage2}()
```

无 DB 层时 fixture 签名去掉 `db_factory`，Flow 中删除 DB 相关行。

### data/{env_name}/{module}/{stage}.yaml

内容 = 该接口完整请求体模板。动态字段置 null 并写注释：

```yaml
biz_no: # 不配置时由 Flow 调用 generator_util 动态生成；配置时长度不可超过32位
biz_id: '' # 无需配置，由 Flow 查库回填
```

敏感数据不要提交：`config/env_test.yaml` 加入 .gitignore，仓库只提交 `env_demo.yaml` 模板。

### pytest.ini

```ini
[pytest]
testpaths = testcases
pythonpath = .
addopts = -vs --html=report/report.html --self-contained-html
python_files = test_*.py
python_functions = test_*
markers =
    {stage1}: {阶段1中文}
    {stage2}: {阶段2中文}
log_cli = True
log_cli_level = INFO
```

## 分层铁律

- API 层禁止断言、禁止读配置；Step 层禁止读 yaml；Flow 层禁止直接 import requests；TestCase 层禁止写业务逻辑
- 断言/日志/yaml/生成器等只从 `utils/` 引用，禁止新建平行实现
- conftest.py 的 fixtures 全部 session 级，新业务模块不需要注册任何 fixture
- 请求/响应日志由 HttpClient 统一记录（Authorization 等凭证头自动脱敏），业务层不要重复打印

## 生成后验证

1. `pip install -r requirements.txt`
2. `pytest --collect-only` —— 用例可收集、无导入错误
3. `pytest -m {stage1}` —— 首阶段跑通，检查 `report/report.html` 生成
4. 提示用户：`TEST_ENV=其他环境` 切换环境；新增环境 = 复制 config/env_*.yaml + data/{env}/ 目录
