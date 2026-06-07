# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

基于 **Python + Requests + Pytest + YAML + Allure** 的接口自动化测试框架。被测服务为 Flask REST API（用户管理相关接口，详见 [flaskDemo](https://github.com/wintests/flaskDemo)）。

## 常用命令

```bash
# 运行全部测试（同时生成 allure 报告原始数据到 ./report）
pytest

# 运行单个测试文件
pytest testcases/api_test/test_01_get_user_info.py

# 按标记运行
pytest -m single        # 单接口测试
pytest -m multiple      # 场景测试
pytest -m negative      # 异常用例

# 查看 allure 报告（需先安装 allure 命令行工具）
allure serve ./report

# 安装依赖
pip install -r requirements.txt
```

## 架构分层（由底向上）

### 1. core/ — HTTP 请求封装
- `rest_client.py`：封装 `requests` 库，提供 `get/post/put/delete/patch` 方法，包含请求日志记录（通过 `common.logger`）。所有请求自动拼接 `api_root_url` 前缀。
- `result_base.py`：关键字返回结果的基类（空壳），业务层动态附加 `success`、`error`、`msg`、`response` 等属性。

### 2. api/ — 接口封装层
- 将 HTTP 接口封装为 Python 方法。例如 `api/user.py` 继承 `RestClient`，提供 `list_all_users()`、`register()`、`login()` 等。
- 模块级自动从 `config/setting.ini` 读取 `api_root_url`，并实例化一个全局单例（如 `user = User(api_root_url)`）。
- **新增接口时**：在此层添加对应方法。

### 3. operation/ — 关键字封装层（业务逻辑层）
- 将多个接口调用组合为有业务语义的关键字，返回 `ResultBase` 对象。统一检查 `res.json()["code"] == 0` 来判断成功/失败，设置 `result.success`。
- 例如 `register_user()` 内部调用 `user.register()`，封装了 JSON 构造、header 设置和结果检查。
- **新增业务关键字时**：在此层组装。

### 4. testcases/ — 测试用例层
- `api_test/`：单接口测试（每个接口一个文件），使用 `@pytest.mark.single`
- `scenario_test/`：多步骤业务场景测试（如 注册→登录→查看），使用 `@pytest.mark.multiple`
- 测试类按接口/场景分组，方法内调用 operation 层的关键字，做断言。

## 数据驱动测试

- 测试数据存放在 `data/` 目录的 YAML 文件中，按测试函数名组织。
- `testcases/conftest.py` 顶层 conftest 通过 `get_data()` 加载所有 YAML 文件为全局变量（`base_data`、`api_data`、`scenario_data`）。
- 各子目录 conftest 提供 `testcase_data` fixture，根据 `request.function.__name__` 自动匹配当前测试对应的数据。
- 单接口测试通常使用 `@pytest.mark.parametrize` 配合 `api_data["test_xxx"]` 直接展开参数；场景测试使用 `testcase_data` fixture 以字典形式获取。

## Configuration 和 Fixtures

- `config/setting.ini`：存放 `api_root_url`（被测服务地址）和 MySQL 数据库连接信息。**此文件包含环境相关配置，不要提交敏感信息。**
- `testcases/conftest.py` 中定义了 session 级别的 `login_fixture`（管理员登录获取 token）和 function 级别的数据库前后置 fixture（`insert_delete_user`、`delete_register_user`、`update_user_telephone`），用于测试前后的数据清理/准备。
- 日志模块 `common/logger.py` 自动输出到 `log/` 目录，按日期滚动。

## 关键约定

- 所有 fixture 依赖通过 `pytest.mark.usefixtures` 显式声明，不依赖隐式 autouse。
- `common/read_data.py` 中 `MyConfigParser` 重写了 `optionxform`，使 `.ini` 键名保留原大小写。
- `common/mysql_operate.py` 在模块加载时即创建数据库连接池实例，需确保目标数据库可访问，否则导入就会失败。
- Allure step 函数（如 `step_1()`）仅用于在报告中标记步骤，实际日志通过 `common.logger` 输出。
