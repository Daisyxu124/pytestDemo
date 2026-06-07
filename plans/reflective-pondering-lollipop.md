# 项目整体架构分析 & 用户登录接口测试完整调用链路

## 一、项目目录结构

```
pytestDemo/
├── pytest.ini                    # pytest 主配置（测试路径、markers、allure 报告）
├── requirements.txt
├── config/
│   └── setting.ini               # 环境配置（api_root_url、MySQL 连接信息）
├── core/                          # ★ 底层：HTTP 请求封装
│   ├── rest_client.py            #    封装 requests 库，提供 get/post/put/delete/patch
│   └── result_base.py            #    关键字返回结果的空壳基类
├── api/                           # ★ 第二层：接口封装层
│   └── user.py                   #    继承 RestClient，一个方法对应一个 HTTP 接口
├── operation/                     # ★ 第三层：关键字/业务逻辑层
│   └── user.py                   #    组装接口调用，构造请求体，统一检查返回码
├── testcases/                     # ★ 第四层：测试用例层
│   ├── conftest.py               #    顶层 conftest：数据加载、session级fixture、DB前后置
│   ├── api_test/                  #    单接口测试
│   │   ├── conftest.py           #    提供 testcase_data fixture
│   │   ├── test_01_get_user_info.py
│   │   ├── test_02_register.py
│   │   ├── test_03_login.py      #    ★ 登录单接口测试
│   │   ├── test_04_update_user.py
│   │   └── test_05_delete_user.py
│   └── scenario_test/             #    场景测试（多接口串联）
│       ├── conftest.py
│       ├── test_01_register_login_list.py    # 注册→登录→查看
│       ├── test_02_register_login_update.py  # 注册→登录→修改
│       ├── test_03_register_login_delete.py  # 注册→登录→删除
│       └── test_04_repeat_register.py
├── common/                        # 公共模块
│   ├── logger.py                 #   日志（按日期滚动，同时输出文件+控制台）
│   ├── read_data.py              #   数据读取（YAML/JSON/INI），含大小写保留的 ConfigParser
│   └── mysql_operate.py          #   数据库操作（pymysql 连接池，模块加载即实例化）
└── data/                          # 测试数据（YAML 格式）
    ├── base_data.yml             #   基础数据（管理员账号、SQL语句）
    ├── api_test_data.yml         #   单接口测试数据
    └── scenario_test_data.yml    #   场景测试数据
```

## 二、分层架构（四层模型）

```
┌────────────────────────────────────────────────────────┐
│  4. testcases/  测试用例层                              │
│     - 定义测试类和方法                                    │
│     - @pytest.mark.parametrize 数据驱动                  │
│     - 调用 operation 层关键字，执行 assert 断言            │
│     - Allure 装饰器（@allure.step/feature/story 等）      │
├────────────────────────────────────────────────────────┤
│  3. operation/  关键字/业务逻辑层                         │
│     - 组装多个接口调用为有业务语义的关键字                   │
│     - 构造 JSON 请求体、设置 headers                      │
│     - 统一检查 res.json()["code"] == 0 判断成功/失败       │
│     - 返回 ResultBase 对象（含 success/error/msg/response）│
├────────────────────────────────────────────────────────┤
│  2. api/  接口封装层                                     │
│     - 每个 HTTP 接口封装为一个 Python 方法                 │
│     - 继承 RestClient                                    │
│     - 仅负责 URL 路径和 HTTP 方法，不做业务判断            │
│     - 模块级实例化全局单例 `user = User(api_root_url)`     │
├────────────────────────────────────────────────────────┤
│  1. core/  HTTP 请求封装层                               │
│     - RestClient: 封装 requests 库                       │
│     - 自动拼接 api_root_url 前缀                         │
│     - 每次请求自动记录完整日志（地址/方式/头/参数/体）       │
│     - ResultBase: 空壳基类，业务层动态附加属性             │
└────────────────────────────────────────────────────────┘
```

## 三、用户登录接口测试完整调用链路

以 `testcases/api_test/test_03_login.py::TestUserLogin::test_login_user` 为例：

### 3.1 测试启动阶段

```
pytest 启动
  ↓
读取 pytest.ini → 配置 testpaths/testcases, markers, --alluredir ./report
  ↓
加载 testcases/conftest.py（顶层 conftest）
  ├── get_data() 加载 3 个 YAML → base_data, api_data, scenario_data 全局变量
  │     data/api_test_data.yml → api_data["test_login_user"] =
  │       [ ["wintest","123456",True,0,"登录成功"],
  │         ["测试test","123456",False,1003,"用户名不存在"] ]
  ├── login_fixture (session级) — 本用例未使用
  ├── insert_delete_user (function级) — 本用例未使用
  ├── delete_register_user (function级) — 本用例未使用
  └── update_user_telephone (function级) — 本用例未使用
  ↓
加载 testcases/api_test/conftest.py（子目录 conftest）
  └── testcase_data fixture: 根据 request.function.__name__ 匹配 api_data 中的同名 key
  ↓
pytest 收集 test_03_login.py 中的测试
  └── @pytest.mark.parametrize 将 api_data["test_login_user"] 展开为 2 组参数
        [wintest, 123456, True, 0, "登录成功"]
        [测试test, 123456, False, 1003, "用户名不存在"]
```

### 3.2 单条用例执行流程（以第1组参数为例）

```
输入参数:
  username="wintest", password="123456"
  except_result=True, except_code=0, except_msg="登录成功"

① 测试用例层 — test_login_user()
  │  logger.info("开始执行用例")
  │  调用 operation.user.login_user("wintest", "123456")
  ↓
② 关键字层 — operation/user.py:login_user()
  │  创建 result = ResultBase()
  │  构造 payload = {"username": "wintest", "password": "123456"}
  │  构造 header = {"Content-Type": "application/x-www-form-urlencoded"}
  │  调用 api 层: user.login(data=payload, headers=header)
  ↓
③ 接口层 — api/user.py:User.login()
  │  调用 self.post("/login", data=payload, headers=header)
  ↓
④ HTTP 封装层 — core/rest_client.py:RestClient.post()
  │  调用 self.request("/login", "POST", data=payload, headers=header)
  ↓
⑤ core/rest_client.py:RestClient.request()
  │  url = api_root_url + "/login"
  │      = "http://192.168.89.128:9999" + "/login"
  │      = "http://192.168.89.128:9999/login"
  │  request_log(): 记录完整请求日志
  │  执行 requests.post(url, data=payload, headers=header)
  │  ★ 发出 HTTP POST 请求到被测 Flask 服务 ★
  ↓
⑥ HTTP 响应返回（requests.Response 对象）
  │  响应 JSON 示例: {"code": 0, "msg": "登录成功", "login_info": {"token": "xxx"}}
  ↓
⑦ 回到关键字层 — operation/user.py:login_user()
  │  检查 res.json()["code"] == 0 → True
  │  result.success = True
  │  result.token = res.json()["login_info"]["token"]  ← 提取 token
  │  result.msg = "登录成功"
  │  result.response = res   (原始 Response 对象)
  │  logger.info("登录用户返回结果")
  │  return result
  ↓
⑧ 回到测试用例层 — test_login_user()
  │  result = login_user("wintest", "123456")
  │  step_1("wintest")  ← Allure step 标记
  │
  │  ★ 断言链 ★
  │  assert result.success == True         → 通过（success 为 True）
  │  assert result.response.status_code == 200  → 通过
  │  assert result.response.json()["code"] == 0  → 通过
  │  assert "登录成功" in result.msg        → 通过
  │
  │  logger.info("结束执行用例")
```

### 3.3 场景测试中的登录调用链（多接口串联）

`testcases/scenario_test/test_01_register_login_list.py`:

```
注册→登录→查看 完整链路:
  fixture: delete_register_user (前置清理同名用户)
    ↓
  ① register_user(testcase_data) → POST /register → 注册新用户
    ↓
  ② login_user(testcase_data)    → POST /login    → 登录刚注册的用户
    ↓
  ③ get_one_user_info(testcase_data) → GET /users/{username} → 查看用户信息
```

## 四、关键设计模式总结

| 层次 | 职责 | 关键模式 |
|------|------|----------|
| data/ | 测试数据与代码分离 | YAML 数据驱动，按函数名组织 |
| conftest.py | 前后置处理 | pytest fixture（session/function scope）|
| core/ | HTTP 通信基础 | 封装 requests，统一 URL 拼接和日志 |
| api/ | 接口 1:1 映射 | 全局单例 `user`，方法名对应 API 路径 |
| operation/ | 业务语义组装 | ResultBase 统一返回格式，code==0 判断成败 |
| testcases/ | 断言与报告 | parametrize 数据展开 + Allure 装饰器 |

## 五、数据流转全景图

```
data/*.yml                     config/setting.ini
    │                                  │
    ▼                                  ▼
testcases/conftest.py           api/user.py (模块级)
  get_data() 加载YAML            读取 api_root_url
    │                                  │
    ▼                                  ▼
api_data 全局变量               user = User(api_root_url)
    │                                  │
    ▼                                  ▼
@pytest.mark.parametrize        api/user.py 方法
    │                                  │
    ▼                                  ▼
test_login_user()  ──调用──→  operation/user.py  ──调用──→  core/rest_client.py
       │                           login_user()                request()
       │                               │                          │
       │                               ▼                          ▼
       │                          ResultBase              requests.post(url)
       │                          (success/error/              │
       │                           msg/token/response)         ▼
       │                               │                 HTTP Response
       └──── 断言 ←────────────────────┘
```

## 六、验证方法

```bash
# 运行登录单接口测试
pytest testcases/api_test/test_03_login.py -v -s

# 查看 allure 报告
allure serve ./report
```

预期：2 组参数化用例执行（1 成功登录 + 1 用户名不存在），断言全部通过。
