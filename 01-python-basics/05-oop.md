# 1.5 面向对象

> 运行环境：Python 3.12+
>
> 对应模块验收：模块①·面向对象

**Validates: 模块①·面向对象**

## 目录
- [1. 场景引入](#1-场景引入)
- [2. 可运行代码示例](#2-可运行代码示例)
- [3. 设计思考](#3-设计思考)
- [4. 常见坑](#4-常见坑)
- [5. 最佳实践](#5-最佳实践)
- [6. 练习题](#6-练习题)
- [7. 小结与自检](#7-小结与自检)

## 1. 场景引入

一次接口回归测试会持续经历“添加用例 → 记录结果 → 查看汇总 → 输出报告”。如果只用几个全局容器：

```python
cases: list[dict[str, object]] = []
results: list[dict[str, object]] = []
passed_count = 0
```

任何函数都能直接修改它们。结果可能记录到不存在的用例上，同一用例可能记录两次，`passed_count` 也可能忘记同步。问题不在于函数不能实现需求，而在于**同一轮测试运行的状态分散了，修改状态的规则没有统一入口**。

**一句话直觉：类把“持续存在的相关状态”和“操作这些状态的行为”放在同一边界内。**

本章用 `TestRun` 表示“一次测试运行”。对象内部保存本次运行的用例和结果，外部必须通过 `add_case()`、`record_result()` 操作，因此“未知用例不能记录结果”“同一用例不能重复记录”可由一个边界统一保证。

先区分两个词：

- **类（class）**是对象的设计说明，例如 `TestRun` 规定一次运行有哪些状态和操作；
- **对象/实例（object/instance）**是根据类创建的具体事物，例如“冒烟测试”和“全量回归”是两个独立 `TestRun` 实例。

类不是为了把函数换一种写法。只有状态需要跨多次调用持续存在，而且行为必须维护这些状态的一致性时，类才真正提供价值。

## 2. 可运行代码示例

完整工程位于 [`code/05-oop/`](code/05-oop/)。运行前先预测：示例有三条用例，实际状态码依次为 `200`、`201`、`500`，最终通过、失败和待执行各有多少条？

### 2.1 工程结构

```text
05-oop/
├── .gitignore
├── README.md
├── pyproject.toml
├── src/
│   └── test_run_manager/
│       ├── __init__.py
│       ├── __main__.py
│       ├── models.py
│       ├── reporter.py
│       └── run.py
└── tests/
    ├── test_entrypoint.py
    ├── test_models.py
    ├── test_package.py
    ├── test_reporter.py
    └── test_run.py
```

1.4 已学习按职责拆包，本章继续保持边界：`models.py` 放数据对象，`run.py` 管理有状态流程，`reporter.py` 负责展示，`__main__.py` 只组装和运行。

### 2.2 用 `dataclass` 表达简单数据对象

`src/test_run_manager/models.py`：

```python
"""测试运行领域中的简单数据对象。"""

from dataclasses import dataclass


@dataclass(frozen=True)
class TestCase:
    """描述一条通用测试用例；冻结后不能在运行过程中被意外改写。"""

    case_id: str
    title: str
    expected_status: int

    def __post_init__(self) -> None:
        if not self.case_id.strip():
            raise ValueError("case_id 不能为空")
        if not self.title.strip():
            raise ValueError("title 不能为空")
        if (
            not isinstance(self.expected_status, int)
            or isinstance(self.expected_status, bool)
            or not 100 <= self.expected_status <= 599
        ):
            raise ValueError("expected_status 必须是 100~599 的整数")

    @property
    def description(self) -> str:
        """返回适合报告展示的用例描述。"""
        return self.title


@dataclass(frozen=True)
class ApiTestCase(TestCase):
    """一种带 HTTP 信息的测试用例：ApiTestCase is-a TestCase。"""

    method: str
    endpoint: str

    def __post_init__(self) -> None:
        super().__post_init__()
        if not self.method.strip():
            raise ValueError("method 不能为空")
        if not self.endpoint.startswith("/"):
            raise ValueError("endpoint 必须以 / 开头")

    @property
    def description(self) -> str:
        """在通用标题前补充 API 请求信息。"""
        return f"{self.method.upper()} {self.endpoint} - {self.title}"


@dataclass(frozen=True)
class TestResult:
    """记录一条用例的一次执行结果。"""

    case_id: str
    description: str
    expected_status: int
    actual_status: int
    duration_ms: int

    def __post_init__(self) -> None:
        if self.duration_ms < 0:
            raise ValueError("duration_ms 不能为负数")

    @property
    def passed(self) -> bool:
        """实际状态码等于预期状态码时通过。"""
        return self.actual_status == self.expected_status


@dataclass(frozen=True)
class RunSummary:
    """一次测试运行的只读汇总快照。"""

    total: int
    executed: int
    passed: int
    failed: int
    pending: int
```

`@dataclass` 自动生成初始化、比较和易读的字符串表示，适合“主要承载数据”的对象。`frozen=True` 表示创建后不可重新赋值，避免某条用例执行完后预期状态码又被外部改掉。

`ApiTestCase(TestCase)` 是本章唯一的继承关系：API 用例确实是测试用例，因此能放进任何接收 `TestCase` 的位置；它只增加 HTTP 数据，并替换 `description` 的展示行为。继承不是为了少写几行字段，而是表达稳定的 **is-a** 关系。

### 2.3 报告器：被组合的协作对象

`src/test_run_manager/reporter.py`：

```python
"""测试结果报告器。"""

from collections.abc import Sequence
from typing import Protocol

from .models import RunSummary, TestResult


class ResultReporter(Protocol):
    """TestRun 所需的最小报告协作接口。"""

    def build_report(
        self,
        *,
        run_name: str,
        results: Sequence[TestResult],
        summary: RunSummary,
    ) -> str:
        """根据结果与汇总生成报告。"""


class TextReporter:
    """把测试运行结果格式化为纯文本。"""

    def build_report(
        self,
        *,
        run_name: str,
        results: Sequence[TestResult],
        summary: RunSummary,
    ) -> str:
        lines = [
            f"测试运行：{run_name}",
            (
                f"总用例: {summary.total} | 已执行: {summary.executed} | "
                f"通过: {summary.passed} | 失败: {summary.failed} | "
                f"待执行: {summary.pending}"
            ),
        ]

        for result in results:
            status = "PASS" if result.passed else "FAIL"
            lines.append(
                f"[{status}] {result.case_id} {result.description} "
                f"(expected={result.expected_status}, actual={result.actual_status}, "
                f"{result.duration_ms}ms)"
            )

        return "\n".join(lines)
```

`ResultReporter` 描述 `TestRun` 真正需要的能力：有一个 `build_report()` 方法即可。`TestRun` 不继承报告器，而是持有一个报告器对象，这叫**组合**：`TestRun` **has-a** reporter。

### 2.4 有状态类：把状态与不变量放在一起

`src/test_run_manager/run.py`：

```python
"""一次测试运行的有状态领域对象。"""

from .models import RunSummary, TestCase, TestResult
from .reporter import ResultReporter, TextReporter


class TestRun:
    """管理同一次测试运行中的用例、结果和汇总不变量。"""

    def __init__(self, name: str, reporter: ResultReporter | None = None) -> None:
        if not name.strip():
            raise ValueError("测试运行名称不能为空")

        self._name = name
        # 实例属性：每个 TestRun 都有自己的容器，不会共享状态。
        self._cases: dict[str, TestCase] = {}
        self._results: dict[str, TestResult] = {}
        # 组合：TestRun has-a reporter。
        self._reporter = reporter if reporter is not None else TextReporter()

    @property
    def name(self) -> str:
        """只读运行名称。"""
        return self._name

    @property
    def cases(self) -> tuple[TestCase, ...]:
        """返回不可变视图，避免外部绕过 add_case 修改状态。"""
        return tuple(self._cases.values())

    @property
    def results(self) -> tuple[TestResult, ...]:
        """按记录顺序返回不可变结果视图。"""
        return tuple(self._results.values())

    @property
    def is_complete(self) -> bool:
        """所有已添加用例都有结果时，运行才算完成。"""
        return len(self._cases) == len(self._results)

    def add_case(self, test_case: TestCase) -> None:
        """添加用例，并阻止重复 case_id。"""
        if test_case.case_id in self._cases:
            raise ValueError(f"用例已存在: {test_case.case_id}")
        self._cases[test_case.case_id] = test_case

    def record_result(
        self,
        case_id: str,
        *,
        actual_status: int,
        duration_ms: int,
    ) -> TestResult:
        """为已知用例记录唯一结果。"""
        if case_id not in self._cases:
            raise ValueError(f"未知用例: {case_id}")
        if case_id in self._results:
            raise ValueError(f"结果已记录: {case_id}")

        test_case = self._cases[case_id]
        result = TestResult(
            case_id=test_case.case_id,
            description=test_case.description,
            expected_status=test_case.expected_status,
            actual_status=actual_status,
            duration_ms=duration_ms,
        )
        self._results[case_id] = result
        return result

    def summary(self) -> RunSummary:
        """根据当前状态计算汇总，不维护易失真的独立计数器。"""
        executed = len(self._results)
        passed = sum(result.passed for result in self._results.values())
        total = len(self._cases)
        return RunSummary(
            total=total,
            executed=executed,
            passed=passed,
            failed=executed - passed,
            pending=total - executed,
        )

    def build_report(self) -> str:
        """把当前只读快照交给组合的报告器。"""
        return self._reporter.build_report(
            run_name=self.name,
            results=self.results,
            summary=self.summary(),
        )
```

`__init__` 是初始化方法：调用 `TestRun("冒烟测试")` 时，Python 先创建对象，再调用它设置初始状态。`self` 指当前实例，因此两个 `TestRun` 的 `self._cases` 是两个不同字典。

`_cases` 前的单下划线表示“内部实现，请不要从类外直接使用”。Python 不把它变成绝对私有，必要时仍可访问；封装的重点不是藏住数据，而是提供一个可信边界。这里外部通过只读属性得到 `tuple`，修改只能经过维护规则的方法。

### 2.5 入口：创建对象并发送消息

`src/test_run_manager/__main__.py`：

```python
"""通过 python -m test_run_manager 运行面向对象示例。"""

from test_run_manager import ApiTestCase, TestCase, TestRun, TextReporter


def main() -> None:
    """创建一次测试运行，添加用例、记录结果并输出报告。"""
    test_run = TestRun("订单 API 回归", reporter=TextReporter())
    test_run.add_case(TestCase("login-001", "登录成功", 200))
    test_run.add_case(
        ApiTestCase(
            "order-001", "创建订单", 201,
            method="POST", endpoint="/api/orders",
        )
    )
    test_run.add_case(
        ApiTestCase(
            "refund-001", "退款参数校验", 400,
            method="POST", endpoint="/api/refunds",
        )
    )

    test_run.record_result("login-001", actual_status=200, duration_ms=86)
    test_run.record_result("order-001", actual_status=201, duration_ms=143)
    test_run.record_result("refund-001", actual_status=500, duration_ms=97)

    print(test_run.build_report())


if __name__ == "__main__":
    main()
```

入口不直接操作 `_cases` 或 `_results`，而是调用对象公开的方法。可以把方法调用理解为“向对象发送请求”：运行对象收到“添加用例”请求后，自己决定怎样保存并维护约束。

### 2.6 用测试验证对象边界

`tests/test_run.py` 中的核心边界测试：

```python
import pytest

from test_run_manager import TestCase as Case
from test_run_manager import TestRun as RunManager


def test_unknown_or_duplicate_result_is_rejected() -> None:
    test_run = RunManager("冒烟测试")
    test_run.add_case(Case("login-001", "登录成功", 200))

    with pytest.raises(ValueError, match="未知用例: missing-001"):
        test_run.record_result("missing-001", actual_status=404, duration_ms=10)

    test_run.record_result("login-001", actual_status=200, duration_ms=20)
    with pytest.raises(ValueError, match="结果已记录: login-001"):
        test_run.record_result("login-001", actual_status=500, duration_ms=30)


def test_different_runs_do_not_share_state() -> None:
    first_run = RunManager("第一次运行")
    second_run = RunManager("第二次运行")

    first_run.add_case(Case("login-001", "登录成功", 200))

    assert len(first_run.cases) == 1
    assert second_run.cases == ()
    assert second_run.results == ()
```

测试不只验证“能算对”，还验证对象承诺的不变量和封装边界。特别是实例隔离测试，可及时发现把可变容器误写成类属性造成的串扰。

### 2.7 安装、运行与测试

`pyproject.toml` 固定构建与测试依赖：

```toml
[build-system]
requires = ["setuptools==75.8.0"]
build-backend = "setuptools.build_meta"

[project]
name = "test-run-manager"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = []

[project.optional-dependencies]
test = [
    "pytest==8.4.2",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
addopts = "-q"
```

执行：

```bash
cd codelearn/01-python-basics/code/05-oop
/opt/homebrew/bin/python3.12 -m venv .venv
.venv/bin/python -m pip install "setuptools==75.8.0"
.venv/bin/python -m pip install --no-build-isolation -e ".[test]"
.venv/bin/python -m test_run_manager
.venv/bin/python -m pytest --color=no
```

入口预期输出：

```text
测试运行：订单 API 回归
总用例: 3 | 已执行: 3 | 通过: 2 | 失败: 1 | 待执行: 0
[PASS] login-001 登录成功 (expected=200, actual=200, 86ms)
[PASS] order-001 POST /api/orders - 创建订单 (expected=201, actual=201, 143ms)
[FAIL] refund-001 POST /api/refunds - 退款参数校验 (expected=400, actual=500, 97ms)
```

## 3. 设计思考

### 3.1 三种方案怎么选？

| 方案 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|
| 函数 + 普通数据 | 简单直接，调用链清楚，最少抽象 | 状态跨多次调用时容易散落；调用者可绕过规则修改字典 | 一次输入得到一次输出的转换、无持续状态的小脚本 |
| `dataclass` / 简单数据对象 | 字段明确，自动生成初始化和比较，适合类型注解；可冻结 | 若塞入大量流程方法，会让“数据”承担过多职责 | `TestCase`、配置、请求/响应、不可变结果快照 |
| 有状态类 | 状态与操作规则同处一处，可维护跨调用不变量 | 需要设计边界；滥用会制造只有一个方法的空壳类 | 一次测试运行、会话、任务队列等有生命周期的对象 |

本例没有把所有东西都做成有状态类。`TestCase` 和 `TestResult` 主要表达数据，所以用冻结的 dataclass；`TextReporter` 的工作是一次输入生成一次文本，没有内部状态；只有 `TestRun` 需要在多次调用之间持续管理状态并维护规则。

如果函数签名清晰，而且数据无需跨调用维护一致性，继续用函数通常更好。**“项目要专业”不是使用 OOP 的理由。**

### 3.2 封装不是绝对私有

Python 的单下划线是团队约定：`_cases` 表示它不是稳定公共 API。双下划线会触发名称改写，但仍不是安全边界，也不应拿来保护秘密。

本例采用三层边界：

1. `_cases`、`_results` 标记为内部状态；
2. `cases`、`results` 属性只返回 tuple，不暴露可变字典；
3. 所有写操作经过方法，在写入前检查不变量。

`name` 只有 getter，没有 setter，因此 `run.name = "新名字"` 会失败。这是只读属性。只读的目的不是阻止恶意调用者，而是告诉正常调用者：运行创建后，名称不应被随意改变。

### 3.3 为什么汇总要现算？

如果同时保存 `_results`、`_passed_count` 和 `_failed_count`，一次写入要更新三个状态；任何一步遗漏都会不一致。本例只保存事实数据 `_results`，调用 `summary()` 时再派生计数。

这是“减少状态源”的取舍：当结果数量是常规测试规模时，遍历计算成本很低，却能换来更可靠的一致性。只有数据量巨大且汇总成为明确性能瓶颈时，才值得引入缓存，同时承担缓存失效的复杂度。

### 3.4 组合为什么通常优先于继承？

`TestRun` 需要报告能力，但它**不是一种报告器**，因此 `class TestRun(TextReporter)` 在语义上错误。组合写成 `TestRun(..., reporter=TextReporter())`，运行管理和文本格式可独立变化。

组合的依赖通常较松：替换报告器只需满足 `build_report()` 协议；继承会把子类绑定到父类字段、初始化和可覆盖方法，父类变化更容易影响子类。

本例只在 `ApiTestCase is-a TestCase` 这种稳定关系中使用一层继承，并验证 `ApiTestCase` 可以替代 `TestCase`。如果你只是想复用格式化函数，或两个类型只是“碰巧字段相似”，不要继承。

### 3.5 什么时候需要重构？

当前 `TestRun` 只管理状态码结果，结构足够简单。若以后增加重试历史、并发执行、持久化和多种断言，不要立即创建多层基类。先观察变化方向：

- 只是增加结果字段：扩展数据对象；
- 报告格式变化：新增报告器并组合；
- 结果判定策略变化：先考虑注入函数或独立协作对象；
- 运行生命周期明显复杂：再拆分执行、存储等职责。

抽象应来自已出现的变化，而不是猜测未来。

## 4. 常见坑

| 坑 | 后果 | 根本原因 |
|---|---|---|
| 把 `_cases = {}` 写成类属性 | 多个 `TestRun` 实例共享同一字典，测试互相污染 | 没区分类本身的数据与每个实例的数据 |
| 直接返回内部 dict/list | 调用者可绕过校验添加或删除数据 | 只写了 getter，却没有真正保护写入边界 |
| 为每组函数都创建一个类 | 出现大量无状态、只有一个方法的壳 | 把 OOP 当成语法升级，而不是状态建模工具 |
| 在 `__init__` 中读取文件或发网络请求 | 对象难创建、难测试，初始化失败原因混杂 | 初始化承担了外部流程，而不只是建立合法状态 |
| 用继承只为复用几行代码 | 子类被父类实现细节绑住，关系难以解释 | 没先判断是否存在稳定 is-a 关系 |
| 多层继承并到处覆盖方法 | 实际调用哪个实现难追踪，修改影响范围扩大 | 过早设计扩展体系 |
| 同时存结果和多个汇总计数器 | 数据更新后计数忘记同步 | 保存了可从事实数据计算出的重复状态 |
| 误以为 `_name` 或 `__name` 绝对私有 | 把命名约定当安全措施 | 混淆封装接口与权限控制 |
| dataclass 默认放可变 list | 多个实例可能共享或意外修改列表 | 没有为可变字段使用正确的实例工厂或不可变结构 |

## 5. 最佳实践

1. **先写对象要维护的不变量**：本例先明确“用例 ID 唯一、结果只能属于已知用例、每条用例只记录一次”，再决定方法。
2. **实例状态放在 `__init__`**：每次创建都得到独立容器；不要把可变业务状态放到类属性。
3. **公开行为，隐藏可变表示**：外部调用 `add_case()`，而不是依赖 `_cases` 是 dict；未来更换内部结构不会影响调用方。
4. **数据对象保持简单**：用 dataclass 表达字段、基础校验和紧贴数据的派生属性，不把整个运行流程塞进去。
5. **减少重复状态**：能从结果计算的汇总就现算，先换取正确性，再用性能数据决定是否缓存。
6. **优先组合**：某对象“拥有/使用”另一种能力时，通过构造参数组合；只有可以自然说出“子类是一种父类”时才考虑继承。
7. **继承保持浅且可替换**：`ApiTestCase` 仍满足 `TestCase` 的约定，调用者无需为子类写特殊分支。
8. **测试对象的生命周期**：不仅测单个方法，还测初始 → 添加 → 记录 → 汇总，以及非法状态转换和实例隔离。

这些实践适用于状态会持续变化的小工具和业务对象。对于纯数据转换，函数依然是更轻、更透明的选择。

## 6. 练习题

### 练习 1：函数还是类？（基础）

**对应验收：模块①·面向对象**

判断下面三个需求应优先使用“函数 + 普通数据”“dataclass”还是“有状态类”，并为每项写出一条理由：

1. 输入状态码列表，返回其中所有 5xx 状态码；
2. 表示一份包含环境、接口地址、超时值的运行配置；
3. 管理一次批量接口测试的开始时间、用例结果、完成状态和汇总。

验收标准：理由必须提到是否有持续状态、是否要维护跨调用不变量，而不是只说“类更规范”。

### 练习 2：增加跳过结果（进阶）

**对应验收：模块①·面向对象**

复制本章工程，为结果增加 `SKIPPED` 状态：

- 先写清楚通过、失败、跳过、待执行四者的关系；
- 不新增一个需要手工同步的 `_skipped_count`；
- 仍禁止未知用例和重复结果；
- 给汇总、文本报告和非法转换补测试。

提示：先决定“状态”属于 `TestResult` 的事实字段，还是由其他字段推导，再动代码。

### 练习 3：组合一个 JSON 报告器（挑战）

**对应验收：模块①·面向对象**

新增 `JsonReporter`，使它满足与 `TextReporter` 相同的 `build_report()` 协议，并注入现有 `TestRun`。要求不修改 `add_case()`、`record_result()` 和 `summary()`。

验收标准：解释 `TestRun has-a reporter` 为什么比 `JsonTestRun(TestRun)` 更适合这个变化；用测试证明同一运行逻辑可以组合不同报告器。

## 7. 小结与自检

- 类适合把持续存在的相关状态与维护这些状态的行为放进同一边界；对象是类创建出的独立实例。
- `__init__` 建立实例初始状态，实例属性属于各自对象；方法通过 `self` 操作当前实例。
- Python 的单下划线是内部约定，不是绝对私有；只读属性、不可变快照和受控写方法共同形成实用封装。
- dataclass 适合简单数据对象，有状态类适合维护生命周期和不变量；纯转换优先函数，不要为 OOP 而 OOP。
- 组合表达 has-a，通常比继承更灵活；继承只用于稳定 is-a 关系，并确保子类可替代父类。
- 本章的 `TestRun` 管理用例、结果和汇总，通过测试实证“能用类建模并说明为何用类而非函数”。

自检问题（可选择跳过）：

1. 如果 `summary()` 只是一次计算，为什么它仍适合作为 `TestRun` 的方法，而不是随便放在一个工具模块里？
2. `TextReporter` 和 `ApiTestCase` 分别使用组合与继承，你能用 has-a / is-a 各解释一句吗？
