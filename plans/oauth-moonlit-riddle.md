# 新增接口测试 — 实现方案

## Context

需要在现有的 pytest 接口自动化测试框架中新增一个接口的测试用例。框架基于 **Python + Requests + Pytest + YAML + Allure**，分层架构为 `core/ → api/ → operation/ → testcases/`。目前只覆盖了用户管理相关接口（CRUD + 登录/注册）。

目标：以最小改动、最低风险的方式新增一个接口的测试。

## 待确认问题

在制定精确方案前，需要你确认以下几点：

1. **新接口是什么？** — 是现有被测服务 (Flask REST API) 的新增接口，还是另一个服务的接口？
2. **接口的 HTTP 方法和路径** — 例如 `GET /users`、`POST /order` 等
3. **是否需要新的数据库 fixture？** — 是否需要测试前后的数据准备/清理？

---

## 通用实现模板（适用于任何新接口）

以下改动按顺序执行，每一步都不破坏现有功能。

### Step 1: API 层 — 新增接口封装方法

**文件**: `api/user.py`（如果新接口属于同一服务）或新建 `api/<module>.py`

**模式**（在现有 `User` 类中添加方法，或新建类）:
```python
# 如果是同一服务（同一个 api_root_url），直接在 User 类中加方法：
def <method_name>(self, <params>, **kwargs):
    return self.<http_method>("/<endpoint>", **kwargs)

# 如果是另一个服务，新建类继承 RestClient：
class <ModuleName>(RestClient):
    def __init__(self, api_root_url, **kwargs):
        super().__init__(api_root_url, **kwargs)
    
    def <method_name>(self, <params>, **kwargs):
        return self.<http_method>("/<endpoint>", **kwargs)

<module> = <ModuleName>(api_root_url)
```

**改动量**: 在现有类中加方法：~3 行。新建类：~8 行。

### Step 2: Operation 层 — 新增业务关键字

**文件**: `operation/user.py`（同模块）或新建 `operation/<module>.py`

**模式**（复制现有函数结构）:
```python
from core.result_base import ResultBase
from api.<module> import <singleton>
from common.logger import logger

def <keyword_name>(<params>):
    """<docstring>"""
    result = ResultBase()
    header = {"Content-Type": "application/json"}  # 按需调整
    json_data = {<构造请求体>}
    res = <singleton>.<method>(json=json_data, headers=header)
    result.success = False
    if res.json()["code"] == 0:
        result.success = True
        # 如有 token 等额外字段，在此提取：result.token = res.json()["xxx"]["token"]
    else:
        result.error = "接口返回码是 【 {} 】, 返回信息：{} ".format(res.json()["code"], res.json()["msg"])
    result.msg = res.json()["msg"]
    result.response = res
    logger.info("<描述> ==>> 返回结果 ==>> {}".format(result.response.text))
    return result
```

**改动量**: ~20 行/函数。

### Step 3: 测试数据 — 在 YAML 中增加数据

**文件**: `data/api_test_data.yml`（单接口测试）或 `data/scenario_test_data.yml`（场景测试）

**模式**:
```yaml
test_<function_name>:
  # <参数注释>
  - [<param1>, <param2>, ..., <except_result>, <except_code>, <except_msg>]
  # 按需增加正向/异常多行用例
```

**注意**:
- `except_code` 需要和 API 实际返回的类型一致（根据现有测试数据混用 `0` (int) 和 `"1004"` (str)，建议确认后再写）
- 至少覆盖：正向用例 + 关键异常用例（参数非法、资源不存在等）

**改动量**: ~5-10 行/用例。

### Step 4: 测试用例文件 — 新建测试文件

**文件**: `testcases/api_test/test_0X_<name>.py`（单接口）或 `testcases/scenario_test/test_0X_<name>.py`（场景）

**单接口模式**（复制 `test_03_login.py` 为模板）:
```python
import pytest
import allure
from operation.<module> import <keyword>
from testcases.conftest import api_data
from common.logger import logger

@allure.step("步骤1 ==>> <描述>")
def step_1(<params>):
    logger.info("步骤1 ==>> ...")

@allure.severity(allure.severity_level.NORMAL)
@allure.epic("针对单个接口的测试")
@allure.feature("<模块名>模块")
class Test<ModuleName>():
    """<docstring>"""

    @allure.story("用例--<描述>")
    @allure.description("<描述>")
    @allure.issue("https://www.cnblogs.com/wintest", name="点击，跳转到对应BUG的链接地址")
    @allure.testcase("https://www.cnblogs.com/wintest", name="点击，跳转到对应用例的链接地址")
    @allure.title("测试数据：【 {param1}，...，{except_result}，{except_code}，{except_msg}】")
    @pytest.mark.single
    @pytest.mark.parametrize("<param_names>, except_result, except_code, except_msg",
                             api_data["test_<function_name>"])
    # @pytest.mark.usefixtures("<fixture_name>")  # 如果需要 DB 清理
    def test_<name>(self, <params>, except_result, except_code, except_msg):
        logger.info("*************** 开始执行用例 ***************")
        result = <keyword>(<args>)
        step_1(<args>)
        assert result.response.status_code == 200
        assert result.success == except_result, result.error
        logger.info("code ==>> 期望结果：{}， 实际结果：【 {} 】".format(except_code, result.response.json().get("code")))
        assert result.response.json().get("code") == except_code
        assert except_msg in result.msg
        logger.info("*************** 结束执行用例 ***************")

if __name__ == '__main__':
    pytest.main(["-q", "-s", "test_0X_<name>.py"])
```

**改动量**: ~55 行/文件。

### Step 5 (可选): 数据库 Fixture

如果新接口需要测试前后的数据准备/清理：

1. **`data/base_data.yml`** — 在 `init_sql` 下增加 SQL 语句
2. **`testcases/conftest.py`** — 新增 fixture 函数（复制 `insert_delete_user` 模式）

### Step 6 (可选): 新标记

如果新接口属于新类别，在 `pytest.ini` 的 `markers` 中新增标记。

---

## 改动文件清单（典型场景）

| 文件 | 操作 | 风险 |
|------|------|------|
| `api/user.py` | 修改（新增方法） | 极低，纯增量 |
| `operation/user.py` | 修改（新增函数） | 极低，纯增量 |
| `data/api_test_data.yml` | 修改（新增数据块） | 极低，纯增量 |
| `testcases/api_test/test_0X_<name>.py` | **新建** | 极低，独立文件 |
| `data/base_data.yml` | 可能修改 | 低，新增 SQL 不影响现有 |
| `testcases/conftest.py` | 可能修改 | 低，新增 fixture 不影响现有 |

---

## 验证方式

```bash
# 1. 单独运行新用例
pytest testcases/api_test/test_0X_<name>.py -v

# 2. 运行全部用例确认无回归
pytest -v

# 3. 查看 allure 报告
allure serve ./report
```

---

## 下一步

请告诉我新接口的具体信息（HTTP 方法、路径、参数），我会按照上述模板生成完整代码。
