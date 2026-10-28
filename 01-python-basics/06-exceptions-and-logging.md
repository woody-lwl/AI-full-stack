# 1.6 异常与日志

> 运行环境：Python 3.12+
>
> 对应模块验收：模块①·异常；Req 2.2

**Validates: Requirements 2.2**

## 目录
- [1. 场景引入](#1-场景引入)
- [2. 可运行代码示例](#2-可运行代码示例)
- [3. 设计思考](#3-设计思考)
- [4. 常见坑](#4-常见坑)
- [5. 最佳实践](#5-最佳实践)
- [6. 练习题](#6-练习题)
- [7. 小结与自检](#7-小结与自检)

## 1. 场景引入

假设你写了一个批量接口测试脚本：它从 JSON 读取任务，然后逐条执行。真实运行时可能遇到：

- 文件路径写错，整批任务根本无法启动；
- JSON 少了括号，解析器只抛出一串技术错误；
- 某条任务的 `case_id` 是空值；
- 一条接口超时，但后面的任务仍然应该执行；
- 接口返回 500，断言失败，但 Python 程序本身没有出错。

如果只写一个大范围的 `except Exception: pass`，脚本表面上“没有崩”，却不知道失败发生在哪个文件、哪条用例、哪个动作，也可能把真正的代码缺陷一起吞掉。如果每层都打印一次错误，同一故障又会产生多条重复日志。

**一句话直觉：异常负责把失败沿调用链传递，日志负责留下“何时、在哪、处理什么、为何失败”的诊断证据。**

本章用 `test_job_runner` 建立三层异常边界：

```text
入口层 __main__        记录最终无法处理的失败，决定退出码
        ↑
流程层 runner          决定单条失败是继续、重试还是终止
        ↑
底层 loader/executor   把 JSONDecodeError、TimeoutError 转成领域异常
```

先区分两类“失败”：

1. **预期业务失败**：实际状态码与预期不一致。程序正常完成了请求与比较，应返回 `CaseResult(passed=False)`，通常记录 `WARNING`，而不是抛异常或记录 `ERROR`。
2. **程序/运行异常**：文件不存在、JSON 无法解析、调用超时、代码缺陷等，正常流程无法继续，需要异常传递或恢复策略。

异常不是“失败结果”的同义词。把所有失败都抛异常、都打 `ERROR`，会让真正需要处理的错误淹没在噪声中。

## 2. 可运行代码示例

完整工程位于 [`code/06-exceptions-and-logging/`](code/06-exceptions-and-logging/)。它不发送真实网络请求，而是用本地数据稳定模拟成功、断言失败和超时，便于重复观察异常与日志。

运行前先预测：三条任务依次为成功、断言失败、超时。程序退出码是多少？最后一条超时后，汇总日志中的 `completed` 是多少？

### 2.1 工程结构

```text
06-exceptions-and-logging/
├── .gitignore
├── README.md
├── pyproject.toml
├── examples/
│   └── jobs.json
├── src/
│   └── test_job_runner/
│       ├── __init__.py
│       ├── __main__.py
│       ├── errors.py
│       ├── executor.py
│       ├── job.py
│       ├── loader.py
│       ├── models.py
│       └── runner.py
└── tests/
    ├── test_entrypoint.py
    ├── test_executor.py
    ├── test_loader.py
    ├── test_package.py
    └── test_runner.py
```

延续 1.4 的职责拆分：加载器不决定批处理是否继续，执行器不配置日志，入口不解析 JSON。异常边界建立在模块职责边界上。

### 2.2 自定义异常：给调用方稳定语义

`src/test_job_runner/errors.py`：

```python
"""测试任务运行器的领域异常。"""


class TestJobError(Exception):
    """测试任务运行器可识别错误的基类。"""


class TestDataError(TestJobError):
    """测试数据文件无法读取或内容不符合约定。"""


class CaseTimeoutError(TestJobError):
    """单条测试用例在规定时间内未完成。"""
```

自定义异常不一定需要复杂字段。它首先提供一个稳定分类：流程层可以捕获 `CaseTimeoutError`，而不必知道底层将来使用哪个 HTTP 库、会抛哪一种超时异常。

不要为每条错误消息创建一个异常类。只有调用方需要采取不同策略时，分类才有价值：数据文件整体错误要终止，单条超时可以跳过并继续。

### 2.3 底层转换：主动 `raise` 并保留异常链

`loader.py` 的核心代码：

```python
import json
from pathlib import Path

from .errors import TestDataError


def load_cases(source: Path) -> list[object]:
    try:
        text = source.read_text(encoding="utf-8")
    except FileNotFoundError as exc:
        raise TestDataError(
            f"测试数据文件不存在: source={source}"
        ) from exc

    try:
        raw_data = json.loads(text)
    except json.JSONDecodeError as exc:
        raise TestDataError(
            "JSON 格式错误: "
            f"source={source} line={exc.lineno} column={exc.colno}"
        ) from exc

    if not isinstance(raw_data, list):
        # 已识别出的领域规则失败，可以直接主动抛出。
        raise TestDataError(f"JSON 顶层必须是数组: source={source}")

    return raw_data
```

工程中的真实 `load_cases()` 还会逐条校验 `case_id`、动作、状态码、超时值和 token，并返回 `list[TestCase]`；上面只截取异常转换主线，完整实现见 [`loader.py`](code/06-exceptions-and-logging/src/test_job_runner/loader.py)。

这里有三个关键点：

- `except FileNotFoundError` 和 `except JSONDecodeError` 只捕获当前层能识别的具体异常，不误伤 `TypeError`、`MemoryError` 等其他问题；
- `raise TestDataError(...)` 把“文件系统/JSON 技术细节”翻译为“测试数据不可用”的领域语义；
- `raise ... from exc` 建立异常链。上层看到清晰的领域错误，排查时仍能沿 `__cause__` 找到原始异常和堆栈。

执行器使用相同方式转换超时：

```python
from .errors import CaseTimeoutError
from .models import CaseResult, TestCase


def _simulate_request(test_case: TestCase) -> int:
    if test_case.action == "timeout":
        raise TimeoutError(f"超过 {test_case.timeout_seconds:.1f}s")
    return test_case.actual_status


def execute_case(test_case: TestCase) -> CaseResult:
    try:
        actual_status = _simulate_request(test_case)
    except TimeoutError as exc:
        raise CaseTimeoutError(
            f"用例执行超时: case_id={test_case.case_id} "
            f"timeout={test_case.timeout_seconds:.1f}s"
        ) from exc

    passed = actual_status == test_case.expected_status
    detail = (
        "状态码符合预期"
        if passed
        else f"预期 {test_case.expected_status}，实际 {actual_status}"
    )
    return CaseResult(test_case.case_id, passed, detail)
```

注意断言失败走正常 `return`，只有超时走异常通道。

### 2.4 流程层：`try/except/else/finally` 各司其职

`src/test_job_runner/runner.py` 的主循环：

```python
import logging
from pathlib import Path

from .errors import CaseTimeoutError
from .executor import execute_case
from .models import RunSummary, TestCase

logger = logging.getLogger(__name__)


def run_cases(cases: list[TestCase], *, source: Path) -> RunSummary:
    passed = assertion_failed = timed_out = completed = 0

    for test_case in cases:
        logger.info(
            "开始执行 case_id=%s source=%s action=%s",
            test_case.case_id,
            source,
            test_case.action,
        )
        try:
            result = execute_case(test_case)
        except CaseTimeoutError as exc:
            # 已知可恢复异常：记录后继续循环。
            timed_out += 1
            logger.warning(
                "用例超时，已跳过 case_id=%s source=%s action=%s reason=%s",
                test_case.case_id,
                source,
                test_case.action,
                exc,
            )
        else:
            # 只有 try 没有抛异常时才进入 else。
            if result.passed:
                passed += 1
                logger.info("用例通过 case_id=%s source=%s", test_case.case_id, source)
            else:
                assertion_failed += 1
                logger.warning(
                    "断言未通过 case_id=%s source=%s detail=%s",
                    test_case.case_id,
                    source,
                    result.detail,
                )
        finally:
            # 无论通过、超时，甚至未知异常向上抛，都执行收尾动作。
            completed += 1
            logger.debug(
                "执行尝试结束 case_id=%s source=%s completed=%d",
                test_case.case_id,
                source,
                completed,
            )

    return RunSummary(len(cases), completed, passed, assertion_failed, timed_out)
```

四个关键字的职责是：

| 结构 | 何时执行 | 本例职责 |
|---|---|---|
| `try` | 尝试可能失败的最小代码范围 | 只调用单条执行器 |
| `except` | 匹配到指定异常 | 超时计数、记录并继续 |
| `else` | `try` 未抛异常 | 处理成功或断言失败结果 |
| `finally` | 无论是否抛异常 | 记录该次执行尝试已结束 |

`finally` 适合释放文件、连接、锁等必须清理的资源；许多资源应优先使用 `with` 上下文管理器。本例用它展示完成计数语义，而不是要求每个 `try` 都带 `finally`。

流程层只捕获它有恢复策略的 `CaseTimeoutError`。如果执行器因代码缺陷抛 `RuntimeError`，这里不会吞掉，它会立即向上传递；继续执行未知状态下的后续任务，反而可能制造更多错误。

### 2.5 入口统一配置，模块只获取 logger

`src/test_job_runner/__main__.py`：

```python
import logging
from pathlib import Path

from .errors import TestDataError
from .job import run_job

logger = logging.getLogger("test_job_runner")


def configure_logging() -> None:
    logging.basicConfig(
        level=logging.DEBUG,
        format="%(asctime)s %(levelname)s %(name)s %(message)s",
    )


def main(source: Path | None = None) -> int:
    configure_logging()
    job_source = source or Path(__file__).resolve().parents[2] / "examples" / "jobs.json"

    try:
        summary = run_job(job_source)
    except TestDataError:
        # logger.exception 只能在 except 中使用，会自动附带当前异常堆栈。
        logger.exception("测试任务启动失败 source=%s", job_source)
        return 1

    logger.info(
        "测试任务完成 source=%s total=%d completed=%d passed=%d "
        "assertion_failed=%d timed_out=%d",
        job_source,
        summary.total,
        summary.completed,
        summary.passed,
        summary.assertion_failed,
        summary.timed_out,
    )
    return 0
```

其他模块只写 `logging.getLogger(__name__)`，不调用 `basicConfig()`、不私自添加 handler。这样命令行、pytest、未来的 FastAPI 应用可以各自决定日志输出到终端、文件还是采集系统，避免重复日志。

这里使用**参数化日志**：

```python
logger.info("开始执行 case_id=%s", case_id)
```

而不是：

```python
logger.info(f"开始执行 case_id={case_id}")
```

前者把模板和参数交给 logging，若 INFO 被过滤，可以避免提前格式化字符串；结构也更统一。不是说日志中永远不能使用 f-string，而是高频、可能被过滤的日志优先参数化。

### 2.6 日志级别不是“严重语气”

| 级别 | 含义 | 本例/适用场景 |
|---|---|---|
| `DEBUG` | 开发排查所需的细节 | 单条尝试结束、内部计数 |
| `INFO` | 正常生命周期中的关键事件 | 开始执行、用例通过、整批完成 |
| `WARNING` | 可恢复、需关注，但程序仍按策略运行 | 断言未通过、单条超时后跳过 |
| `ERROR` | 当前操作失败，通常需要人工处理 | 数据文件使整批任务无法启动 |
| `CRITICAL` | 系统核心能力不可用，可能必须立即停止 | 配置中心和数据库同时不可用等极端故障 |

`logger.exception()` 的日志级别默认是 `ERROR`，并附带当前异常堆栈，因此应在 `except` 块中用于“这一层最终记录的异常”。对已经理解且可恢复的超时，如果每次都打印完整堆栈，会制造大量噪声；本例只记录一条带原因的 `WARNING`。

### 2.7 安装、运行与测试

`pyproject.toml` 固定 Python 与依赖：

```toml
[build-system]
requires = ["setuptools==75.8.0"]
build-backend = "setuptools.build_meta"

[project]
name = "test-job-runner"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = []

[project.optional-dependencies]
test = ["pytest==8.4.2"]
```

执行：

```bash
cd codelearn/01-python-basics/code/06-exceptions-and-logging
/opt/homebrew/bin/python3.12 -m venv .venv
.venv/bin/python -m pip install "setuptools==75.8.0"
.venv/bin/python -m pip install --no-build-isolation -e ".[test]"
.venv/bin/python -m test_job_runner
.venv/bin/python -m pytest --color=no
```

日志含运行时间和绝对路径，因此不要逐字符比对。应核对这些稳定信息：

```text
INFO    ... case_id=login-001 ... action=success
WARNING ... 断言未通过 case_id=order-002 ...
WARNING ... 用例超时，已跳过 case_id=refund-003 ...
INFO    ... total=3 completed=3 passed=1 assertion_failed=1 timed_out=1
```

测试还验证：文件不存在、非法 JSON、错误顶层结构、空 `case_id`、异常链、超时后继续、正确日志级别与上下文、token 不出现在日志中，以及未知 `RuntimeError` 不会被流程层吞掉。

## 3. 设计思考

### 3.1 应该在哪一层捕获？

判断标准不是“哪里写起来方便”，而是：**这一层是否拥有足够上下文，并且能做出有意义的恢复决定？**

| 边界 | 知道什么 | 应做什么 | 不应做什么 |
|---|---|---|---|
| 底层加载/执行 | 具体技术操作为何失败 | 捕获具体技术异常，转换为领域异常并保留链 | 决定整批继续还是退出 |
| 流程层 | 当前 `case_id`、来源、后续任务 | 对已知可恢复异常跳过/重试/汇总 | 捕获所有 `Exception` 后假装成功 |
| 入口层 | 应用如何向用户/系统结束 | 统一配置日志，记录最终失败，返回非零退出码 | 重复记录每个已经处理过的异常 |

异常可以经过多层，但日志不应每层都写。常用原则是：**要么处理，要么补充语义后继续抛；在最终决定失败的一层记录完整堆栈。**

### 3.2 自定义异常与原生异常如何取舍？

- 直接使用 `ValueError`：代码短，适合函数参数等局部调用约定；跨模块后，调用方难区分字段错误和内部代码错误。
- 自定义领域异常：调用方策略稳定，不依赖底层库；代价是要维护合理分类。
- 包装并使用 `raise ... from ...`：同时得到领域语义和原始原因，最适合跨层边界；不要把敏感原始数据拼进新消息。

本例在 `_parse_case()` 内将空字段导致的 `ValueError` 包装为 `TestDataError`，并添加 `source` 与 `index`。调用方知道“测试数据无效”，排查者还能找到“第 0 条的 `case_id` 为空”。

### 3.3 为什么日志上下文比长错误描述更重要？

“执行失败”几乎无法定位问题；`case_id=refund-003 source=examples/jobs.json action=timeout reason=...` 能回答：处理哪个对象、输入来自哪里、正在做什么、为什么失败。

但上下文不是越多越好。token、密码、Cookie、身份证号、完整请求体可能进入日志平台并长期保存。示例的 `TestCase.token` 使用 `repr=False` 降低误打印风险，真正的规则仍是：**日志调用只传允许记录的字段，不记录整个对象或原始 JSON。** 如必须关联用户，应使用脱敏值或内部追踪 ID。

### 3.4 何时重试？

本章选择“超时后跳过”，因为重点是异常边界。真实项目增加重试前要回答：

- 操作是否幂等，重复请求会不会创建两笔订单？
- 哪些异常值得重试，参数错误显然不该重试；
- 最大次数、退避时间和总超时是多少；
- 每次尝试如何记录，最终失败由哪层汇总。

不要把 `except Exception: 再执行一次` 当成重试策略。

## 4. 常见坑

| 坑 | 后果 | 根本原因 |
|---|---|---|
| `except Exception: pass` | 真正缺陷被隐藏，结果可能不完整却显示成功 | 没有定义恢复策略，只想让程序“不报错” |
| 捕获范围覆盖几十行 | 无法判断是哪一步失败，也可能误捕后续处理异常 | `try` 没缩小到真正可能失败的操作 |
| 只写 `raise TestDataError(...)`，不写 `from exc` | 原始异常关系不清晰，排查技术原因困难 | 转换语义时丢了因果链 |
| 底层、流程、入口都打印同一异常 | 一次故障产生多条 ERROR 和多份堆栈 | 没区分“传递异常”和“最终记录” |
| 所有断言失败都打 ERROR | 告警噪声过大，程序缺陷与测试结果混在一起 | 把预期业务失败等同于程序异常 |
| 在库模块调用 `basicConfig()` | 使用方无法控制格式，可能重复输出 | 模块越过职责边界配置整个应用 |
| `logger.exception()` 写在 `except` 外 | 日志出现无意义的 `NoneType: None` | 没理解它读取当前异常上下文 |
| 日志使用 `f"...{token}"` 或打印整个请求 | 凭据和隐私数据进入长期日志 | 为方便排查而无选择地记录输入 |
| 用 `finally` 返回值 | 覆盖 `try` 中的返回或异常 | 误把 finally 当普通收尾分支 |
| 对所有错误立即重试 | 重复写数据、放大下游压力 | 没判断异常类型与操作幂等性 |

## 5. 最佳实践

1. **先画异常边界，再写 `try`**：明确底层负责翻译、流程层负责恢复、入口层负责最终记录与退出码。
2. **只捕获可处理的具体异常**：捕获 `JSONDecodeError`、`CaseTimeoutError`；未知异常默认继续向上暴露。
3. **缩小 `try` 范围**：把正常结果处理放在 `else`，减少误捕自己处理代码中的错误。
4. **转换异常时保留因果链**：使用 `raise DomainError(...) from exc`，新消息补充 `source`、`index`、`case_id` 等安全上下文。
5. **异常消息面向调用方，日志面向诊断**：消息说明发生什么，日志再增加时间、模块、对象标识和动作。
6. **入口统一配置 logging**：模块使用 `logging.getLogger(__name__)`；应用入口选择级别、格式和输出位置。
7. **使用参数化日志**：`logger.info("case_id=%s", case_id)`；让 logging 延迟格式化并保持模板稳定。
8. **按运营含义选择级别**：正常过程用 INFO，可恢复异常或业务失败用 WARNING，当前操作失败用 ERROR，系统级灾难才用 CRITICAL。
9. **仅在需要堆栈时使用 `logger.exception()`**：放在 `except` 内；已知、频繁、可恢复失败通常不必每次打印堆栈。
10. **默认不记录敏感数据**：使用允许列表选择日志字段；token、密码、Cookie 和完整隐私数据必须省略或脱敏。
11. **测试异常和日志契约**：不仅断言异常类型，也断言 `__cause__`、继续/终止行为、日志级别、定位上下文和敏感值不存在。

这些做法适合测试脚本、后端任务和 AI 调用流程。很小的一次性脚本可以减少异常层级，但“不要吞未知异常”和“不要泄露凭据”没有规模例外。

## 6. 练习题

### 练习 1：增加“连接失败”策略（基础）

**对应验收：模块①·异常；Req 2.2**

新增本地动作 `connection_error`，底层模拟 `ConnectionError` 并转换为自定义 `CaseConnectionError`。流程层将它记录为 `WARNING` 后继续，不捕获其他未知异常。

验收：测试异常链的 `__cause__`；日志必须包含 `case_id`、`source`、`action`，不能包含 token。

### 练习 2：设计有限重试（进阶）

**对应验收：模块①·异常；Req 2.2**

为超时增加最多 2 次重试。先写出重试计数、最终超时数和 `completed` 的定义，再实现代码。

提示：把“执行一次”和“重试编排”分开；每次尝试记录 DEBUG，最终仍失败记录 WARNING。说明为什么创建订单类非幂等动作不能直接套用同一策略。

### 练习 3：检查日志安全（挑战）

**对应验收：模块①·异常；Req 2.2**

在任务中增加 `user_email` 和 `password`，设计安全日志字段：邮件仅保留掩码或内部 user_id，密码永不输出。用 `caplog` 写测试，证明正常、断言失败和超时三条路径都不会泄露原值。

验收：除了断言“秘密不存在”，还要断言 `case_id` 和安全追踪字段仍然存在，避免以“不写日志”假装安全。

## 7. 小结与自检

- 异常沿调用链传递失败；日志留下时间、模块、处理对象、动作和原因等诊断证据，两者职责不同。
- 捕获具体异常；底层用 `raise ... from ...` 转换为领域异常，流程层只处理有恢复策略的类型，入口记录最终失败。
- `try` 放最小风险操作，`except` 处理匹配失败，`else` 处理正常结果，`finally` 保证必要清理，但不必强行四者齐用。
- 断言未通过是可预期测试结果，不等于程序异常；日志级别应表达运营影响，而不是表达情绪。
- logger 由模块按 `__name__` 获取，配置集中在入口；优先参数化日志，`logger.exception()` 只在异常上下文中用于需要堆栈的最终记录。
- 日志必须包含 `case_id`、来源和动作等定位上下文，同时不得泄露 token、密码或完整隐私数据。

自检问题（可选择跳过）：

1. 为什么流程层只捕获 `CaseTimeoutError`，反而比捕获 `Exception` 更可靠？如果出现 `RuntimeError`，你希望程序继续还是立即暴露？
2. 非法 JSON 在 loader、runner 和入口之间传递时，哪一层转换异常、哪一层记录完整堆栈？为什么不应三层都记录 ERROR？
