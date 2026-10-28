# 1.11 设计模式在测试中的应用

> 运行环境：Python 3.12+
>
> 对应模块验收：模块①·设计模式

**Validates: Requirements 22.5**

## 目录
- [1. 场景引入](#1-场景引入)
- [2. 可运行代码示例](#2-可运行代码示例)
- [3. 设计思考](#3-设计思考)
- [4. 常见坑](#4-常见坑)
- [5. 最佳实践](#5-最佳实践)
- [6. 练习题](#6-练习题)
- [7. 小结与自检](#7-小结与自检)

## 1. 场景引入

一个接口测试框架最初可能只有状态码校验：`actual == expected`。随着项目增长，团队又加入必填字段、响应头、JSON Schema、业务错误码和响应耗时规则。若所有规则继续堆在一个 `if/elif` 中，就会出现三种变化混在一起：

1. **对象创建变化**：JSON 配置怎样校验、怎样变成可执行规则；
2. **校验算法变化**：每种规则怎样判断响应并生成问题；
3. **横切行为变化**：每次执行前后都要计时、记录日志或采集指标。

重构前示意保存在 [`examples/conditional_before.txt`](code/11-design-patterns-in-testing/examples/conditional_before.txt)：

```python
# 重构前示意：创建、算法和附加行为散落在同一流程中。
def validate(response, configs):
    started = perf_counter()
    issues = []
    for config in configs:
        if config["type"] == "status_code":
            if response.status_code != config["expected"]:
                issues.append("状态码不匹配")
        elif config["type"] == "required_fields":
            for field in config["fields"]:
                if field not in response.body:
                    issues.append(f"缺少字段: {field}")
        else:
            raise ValueError("未知规则")
    print(f"elapsed={perf_counter() - started}")
    return issues
```

这里的 `if/elif` 本身不是错误。问题是它同时解释外部配置、实现所有算法、控制错误格式并负责计时输出。新增规则必须修改这个中心函数；若登录校验、订单校验、批量回归入口都需要计时，又会各自复制 `started = perf_counter()` 和输出代码。横切代码一旦有的用秒、有的用毫秒、有的遇到异常不记录，测试报告就会逐渐不一致。

**一句话直觉：工厂集中回答“创建哪个对象”，策略把“使用哪个算法”变成可替换对象，装饰器在不改核心算法的情况下统一附加计时等横切行为。**

本章的目标不是消灭所有条件分支，而是让每种变化只影响与它同一职责的边界。

## 2. 可运行代码示例

完整离线工程位于 [`code/11-design-patterns-in-testing/`](code/11-design-patterns-in-testing/)：

```text
11-design-patterns-in-testing/
├── .gitignore
├── README.md
├── pyproject.toml
├── examples/
│   ├── conditional_before.txt
│   └── rules.json
├── src/test_validation_patterns/
│   ├── __init__.py
│   ├── __main__.py
│   ├── decorators.py
│   ├── factory.py
│   ├── models.py
│   ├── runner.py
│   └── strategies.py
└── tests/
    ├── test_decorators.py
    ├── test_factory.py
    ├── test_main.py
    ├── test_runner.py
    └── test_strategies.py
```

示例只处理内存中的 `ApiResponse`，不访问真实接口。规则配置来自本地 JSON；时钟由入口注入固定值，因此输出可重复。

### 2.1 稳定模型：让模式围绕同一种输入输出协作

`models.py` 定义进入校验核心的数据：

```python
from collections.abc import Mapping
from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class ApiResponse:
    """进入校验核心的稳定响应模型。"""

    status_code: int
    body: Mapping[str, object]


@dataclass(frozen=True, slots=True)
class ValidationIssue:
    """一条带规则来源的校验问题。"""

    rule: str
    message: str


@dataclass(frozen=True, slots=True)
class ValidationResult:
    """一次响应经过所有策略后的聚合结果。"""

    issues: tuple[ValidationIssue, ...]

    @property
    def passed(self) -> bool:
        return not self.issues
```

使用自己的 `ApiResponse`，而不是让策略直接依赖某个 HTTP 库的 response 类型，可以保持示例离线，也避免将第三方库细节传播到每个策略。

### 2.2 策略：每种规则独立实现同一个最小契约

`strategies.py` 的 `Protocol` 描述执行器真正需要的能力：策略有名称，并能把响应转换为零到多条问题。

```python
from dataclasses import dataclass, field
from typing import Protocol

from .models import ApiResponse, ValidationIssue


class ValidationStrategy(Protocol):
    """执行器需要的最小校验算法契约。"""

    @property
    def name(self) -> str:
        raise NotImplementedError

    def validate(self, response: ApiResponse) -> tuple[ValidationIssue, ...]:
        raise NotImplementedError


@dataclass(frozen=True, slots=True)
class StatusCodeStrategy:
    expected: int
    name: str = field(default="status_code", init=False)

    def validate(self, response: ApiResponse) -> tuple[ValidationIssue, ...]:
        if response.status_code == self.expected:
            return ()
        return (
            ValidationIssue(
                rule=self.name,
                message=f"expected {self.expected}, got {response.status_code}",
            ),
        )


@dataclass(frozen=True, slots=True)
class RequiredFieldsStrategy:
    fields: tuple[str, ...]
    name: str = field(default="required_fields", init=False)

    def validate(self, response: ApiResponse) -> tuple[ValidationIssue, ...]:
        missing = tuple(field for field in self.fields if field not in response.body)
        if not missing:
            return ()
        return (
            ValidationIssue(
                rule=self.name,
                message=f"missing fields: {', '.join(missing)}",
            ),
        )
```

两个策略没有继承共同基类。Python 的结构化 `Protocol` 允许新对象只要提供匹配的 `name` 和 `validate()` 就能接入，测试中的 `HeaderStrategy` 正是这样证明扩展能力的。

`runner.py` 只负责按注入顺序执行与聚合，不判断具体策略类型：

```python
from collections.abc import Iterable

from .models import ApiResponse, ValidationResult
from .strategies import ValidationStrategy


class ValidationRunner:
    """按注入顺序执行策略并聚合所有问题。"""

    def __init__(self, strategies: Iterable[ValidationStrategy]) -> None:
        self._strategies = tuple(strategies)

    @property
    def strategy_names(self) -> tuple[str, ...]:
        return tuple(strategy.name for strategy in self._strategies)

    def run(self, response: ApiResponse) -> ValidationResult:
        issues = tuple(
            issue
            for strategy in self._strategies
            for issue in strategy.validate(response)
        )
        return ValidationResult(issues=issues)
```

这里仍然存在循环，因为“依次执行多条规则”是稳定流程；被移除的是执行器对 `status_code`、`required_fields` 等具体类型的分支依赖。

### 2.3 工厂：把不可信配置转换为合法策略

`factory.py` 使用函数工厂。它集中处理类型选择与构造参数校验，让策略只接收已经合法的数据：

```python
import json
from collections.abc import Iterable, Mapping
from json import JSONDecodeError
from pathlib import Path

from .strategies import (
    RequiredFieldsStrategy,
    StatusCodeStrategy,
    ValidationStrategy,
)


class StrategyConfigError(ValueError):
    """规则配置无法构造合法策略。"""


def _reject_extra_keys(config: Mapping[str, object], allowed: set[str]) -> None:
    extra = set(config) - allowed
    if extra:
        raise StrategyConfigError(f"unexpected config keys: {', '.join(sorted(extra))}")


def create_strategy(config: Mapping[str, object]) -> ValidationStrategy:
    """函数工厂集中选择具体策略并校验其构造参数。"""
    rule_type = config.get("type")
    if rule_type == "status_code":
        _reject_extra_keys(config, {"type", "expected"})
        expected = config.get("expected")
        if isinstance(expected, bool) or not isinstance(expected, int):
            raise StrategyConfigError("status_code.expected must be an integer")
        if not 100 <= expected <= 599:
            raise StrategyConfigError(
                "status_code.expected must be between 100 and 599"
            )
        return StatusCodeStrategy(expected=expected)

    if rule_type == "required_fields":
        _reject_extra_keys(config, {"type", "fields"})
        raw_fields = config.get("fields")
        if not isinstance(raw_fields, list) or not raw_fields:
            raise StrategyConfigError("required_fields.fields must be a non-empty list")
        if any(not isinstance(item, str) or not item for item in raw_fields):
            raise StrategyConfigError(
                "required_fields.fields must contain non-empty strings"
            )
        fields = tuple(raw_fields)
        if len(set(fields)) != len(fields):
            raise StrategyConfigError(
                "required_fields.fields must not contain duplicates"
            )
        return RequiredFieldsStrategy(fields=fields)

    raise StrategyConfigError(f"unknown rule type: {rule_type!r}")


def create_strategies(
    configs: Iterable[Mapping[str, object]],
) -> tuple[ValidationStrategy, ...]:
    strategies = tuple(create_strategy(config) for config in configs)
    if not strategies:
        raise StrategyConfigError("at least one rule config is required")
    return strategies


def load_rule_configs(path: Path) -> tuple[dict[str, object], ...]:
    """从 JSON 边界读取配置并转换为工厂可接收的稳定结构。"""
    try:
        raw_data: object = json.loads(path.read_text(encoding="utf-8"))
    except (OSError, JSONDecodeError) as error:
        raise StrategyConfigError(f"cannot load rule config: {path}") from error

    if not isinstance(raw_data, list):
        raise StrategyConfigError("rule config root must be a list")

    configs: list[dict[str, object]] = []
    for index, raw_config in enumerate(raw_data):
        if not isinstance(raw_config, dict):
            raise StrategyConfigError(f"rule config at index {index} must be an object")
        normalized: dict[str, object] = {}
        for key, value in raw_config.items():
            if not isinstance(key, str):
                raise StrategyConfigError(
                    f"rule config at index {index} has a non-string key"
                )
            normalized[key] = value
        configs.append(normalized)
    return tuple(configs)
```

工厂内部仍使用两个清楚的 `if`。设计模式并不要求项目中完全没有条件分支；它把“创建哪个策略”的分支集中在创建边界，使 `ValidationRunner` 不再因算法增加而修改。

### 2.4 装饰器：统一计时，同时保留函数元数据

`decorators.py` 把时钟和事件接收器都作为参数注入：

```python
from collections.abc import Callable
from dataclasses import dataclass
from functools import wraps
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")
Clock = Callable[[], float]


@dataclass(frozen=True, slots=True)
class TimingEvent:
    function_name: str
    elapsed_seconds: float


TimingSink = Callable[[TimingEvent], None]


def timed(
    *, clock: Clock, sink: TimingSink
) -> Callable[[Callable[P, R]], Callable[P, R]]:
    """为函数附加可注入、可确定测试的计时行为。"""

    def decorate(function: Callable[P, R]) -> Callable[P, R]:
        @wraps(function)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            started_at = clock()
            try:
                return function(*args, **kwargs)
            finally:
                sink(
                    TimingEvent(
                        function_name=function.__name__,
                        elapsed_seconds=clock() - started_at,
                    )
                )

        return wrapper

    return decorate
```

关键点有四个：

- `ParamSpec` 和 `TypeVar` 让包装后函数尽量保持原参数与返回类型；
- `@wraps(function)` 复制 `__name__`、`__doc__` 等元数据，pytest 报告、日志和调试信息不会只看到 `wrapper`；
- `finally` 保证被装饰函数成功或抛异常时都会产生计时事件；
- `clock` 与 `sink` 不在装饰器内部写死，所以测试可注入固定时钟和列表的 `append`，不依赖真实时间和终端输出。

### 2.5 三种模式如何在入口协作

`__main__.py` 是完整组装入口：工厂读取并创建策略，执行器运行策略，装饰器在外层附加计时。

```python
from collections.abc import Iterator
from pathlib import Path

from .decorators import TimingEvent, timed
from .factory import create_strategies, load_rule_configs
from .models import ApiResponse, ValidationResult
from .runner import ValidationRunner


def _demo_clock(values: Iterator[float]) -> float:
    return next(values)


def main() -> int:
    project_root = Path(__file__).resolve().parents[2]
    configs = load_rule_configs(project_root / "examples" / "rules.json")
    runner = ValidationRunner(create_strategies(configs))

    timings: list[TimingEvent] = []
    clock_values = iter((10.0, 10.025))

    @timed(clock=lambda: _demo_clock(clock_values), sink=timings.append)
    def run_validation(response: ApiResponse) -> ValidationResult:
        return runner.run(response)

    response = ApiResponse(status_code=200, body={"user_id": "u-1001"})
    result = run_validation(response)

    print(
        f"RULES count={len(runner.strategy_names)} "
        f"types={','.join(runner.strategy_names)}"
    )
    print(f"RESULT passed={result.passed} issues={len(result.issues)}")
    for issue in result.issues:
        print(f"ISSUE [{issue.rule}] {issue.message}")
    event = timings[0]
    print(
        f"DECORATOR function={event.function_name} "
        f"elapsed_ms={event.elapsed_seconds * 1000:.1f}"
    )
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

配置要求状态码为 201，并要求 `user_id` 与 `token`；示例响应实际为 200 且缺少 `token`，因此两条策略各产生一个问题。固定时钟从 `10.0` 前进到 `10.025`，确定性耗时是 25.0 毫秒。

### 2.6 先预测，再运行

先预测三个问题：工厂会创建几个策略？问题顺序由什么决定？被装饰函数的名称会显示 `run_validation` 还是 `wrapper`？

运行完整示例与质量门禁：

```bash
cd codelearn/01-python-basics/code/11-design-patterns-in-testing
/opt/homebrew/bin/python3.12 -m venv .venv
.venv/bin/python -m pip install "setuptools==75.8.0"
.venv/bin/python -m pip install --no-build-isolation -e ".[dev]"
.venv/bin/python -m test_validation_patterns
.venv/bin/python -m pytest --color=no
.venv/bin/python -m mypy src tests
.venv/bin/python -m ruff check .
.venv/bin/python -m ruff format --check .
```

示例预期输出与 `README.md`、入口集成测试保持一致：

```text
RULES count=2 types=status_code,required_fields
RESULT passed=False issues=2
ISSUE [status_code] expected 201, got 200
ISSUE [required_fields] missing fields: token
DECORATOR function=run_validation elapsed_ms=25.0
```

测试分别覆盖两个策略的成功与失败、问题聚合顺序、工厂合法与非法配置、新策略接入、`wraps` 元数据、固定时钟、被装饰函数异常传播、实例状态隔离和真实模块入口。

## 3. 设计思考

### 3.1 三种模式各自只解决一个变化维度

| 模式 | 本例职责 | 它不负责什么 | 主要收益 |
|---|---|---|---|
| 工厂 | 校验配置并创建具体策略 | 执行全部策略、记录耗时 | 对象创建规则集中，外部配置不直接污染算法 |
| 策略 | 封装一种可替换校验算法 | 解析 JSON、选择日志格式 | 新算法独立测试，执行器不依赖具体类型 |
| 装饰器 | 在函数调用外统一附加计时 | 决定校验是否通过 | 横切行为复用，核心函数保持单一职责 |

若一个类同时解析配置、实现算法又记录指标，只是给旧大函数换了面向对象外壳，并没有真正分离变化原因。

### 3.2 函数工厂还是工厂类

| 方案 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|
| 直接构造具体策略 | 最简单、跳转最少 | 调用方必须知道具体类型与参数校验 | 规则在代码中固定，调用点很少 |
| 函数工厂，本例选择 | 代码短、无实例状态、配置分支集中 | 规则很多时函数会变长；动态注册能力有限 | 少量已知规则，创建过程同步且无共享依赖 |
| 工厂类 | 可注入注册表、schema 校验器、依赖或缓存；适合插件化 | 增加生命周期、状态和测试成本 | 规则由插件动态注册，创建依赖环境或需要缓存 |

本例只有两种已知规则，没有工厂状态，因此函数比工厂类更诚实。仅仅因为名称中有“工厂”，就创建 `StrategyFactory` 类，是把模式形式放在问题之前。

### 3.3 简单 if、映射表还是策略

| 方案 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|
| 单个 `if/elif` | 阅读路径直接，修改成本低 | 创建、算法和编排容易继续堆在一起 | 两三个稳定判断，逻辑短且不会独立替换 |
| 类型到函数的映射表 | 分派简洁，适合无状态函数 | 参数结构差异大时仍需额外适配和校验 | 每种规则签名一致，只需按名称选择函数 |
| 策略对象，本例选择 | 可携带 `expected`、`fields` 等配置；可组合、可独立测试 | 类型和文件更多，有理解成本 | 算法会增长、规则需按配置组合或运行时替换 |

策略模式没有消除所有 `if`：工厂仍需识别配置类型。它消除的是执行阶段不断扩大的类型判断。新增 `HeaderStrategy` 后，执行器仍只调用 `strategy.validate(response)`，这才是本例的扩展证据。

### 3.4 装饰器的适用边界

适合装饰器的行为通常满足三个条件：横跨多个函数、位于调用前后、不会改变核心业务含义。计时、指标、统一 tracing 和权限检查经常符合；只在一个函数中使用的一行计算通常不值得装饰。

装饰器也不是所有横切行为的默认答案：

- 行为需要调用方明确选择时，普通高阶函数或上下文管理器更直观；
- 行为围绕一段代码块而非完整函数时，上下文管理器更合适；
- 行为需要大量业务参数并改变返回模型时，应用服务或显式包装对象更容易理解；
- 测试若必须层层穿透五六个装饰器才能找到失败来源，抽象已经超过收益。

### 3.5 `wraps`、顺序、异常传播与副作用

多个装饰器从下往上应用：

```python
@outer
@inner
def run() -> str:
    return "done"
```

它等价于 `run = outer(inner(run))`。调用时先进入 `outer`，再进入 `inner`，退出顺序相反。计时、重试与异常转换叠加时，顺序会改变语义：把计时放在重试外层，得到所有尝试的总耗时；放在重试内层，则每次尝试各记录一次。

`@wraps` 不只是让名称好看。测试发现函数、错误堆栈、日志字段、文档工具和某些依赖注入框架都会读取函数元数据。忘记它后，多个函数看起来都叫 `wrapper`，定位问题会明显变难。

本例用 `try/finally`，所以被装饰函数的 `RuntimeError` 不会被吞掉，同时异常路径也记录耗时。但有一个重要边界：`clock()` 和 `sink()` 本身也是副作用；如果它们抛异常，可能覆盖被装饰函数原本的异常。本例选择可控的可靠 sink，并用列表 `append` 测试。生产指标若允许“尽力而为”，应明确决定是捕获 sink 异常并记录备用日志，还是让采集失败阻断业务，不能无意识地吞掉异常。

### 3.6 什么时候不使用模式

以下情况保持普通代码通常更好：

- 只有一个稳定算法，而且没有运行时替换或独立配置；
- 条件分支只有两三行，新增频率低，集中阅读反而更清楚；
- 横切行为只出现一次，没有重复，也不影响核心测试；
- 引入模式后，新增一条规则要跨接口、抽象基类、注册表、工厂类和配置类五个位置；
- 团队必须先学习模式术语，才能理解原本很直接的业务判断。

设计模式是对重复设计问题的命名，不是项目成熟度徽章。先观察变化和重复，再选择最小结构。

## 4. 常见坑

| 坑 | 后果 | 根本原因 |
|---|---|---|
| 为两行稳定判断建立策略体系 | 文件和概念比业务逻辑更多 | 只看到模式名称，没有评估变化频率 |
| 工厂、策略、执行器都判断规则类型 | 新增规则仍要修改多处 | 分支没有真正集中，职责边界重复 |
| 用工厂类但类中没有状态或依赖 | 多一次实例化与跳转，无实际收益 | 误以为工厂模式必须是类 |
| 策略直接读取原始 JSON | 每个算法重复做类型与缺字段校验 | 配置边界没有集中到工厂 |
| 执行器返回第一条失败就停止 | 后续问题不可见，排障需反复运行 | 没先定义 fail-fast 还是问题聚合语义 |
| 把规则顺序放进 `set` | 问题输出顺序不稳定 | 忽略测试报告需要确定性 |
| 装饰器忘记 `functools.wraps` | 函数名、文档和调试信息变成 `wrapper` | 只关注调用结果，忽略元数据契约 |
| 装饰器捕获 `Exception` 后返回默认值 | 真正程序错误被伪装成校验失败 | 横切行为越权改变业务语义 |
| 计时测试使用真实 `perf_counter()` 并断言精确值 | 测试受机器调度影响而波动 | 时间依赖未注入 |
| 多个装饰器顺序随意 | 总耗时、单次耗时、重试和异常语义改变 | 没把嵌套调用顺序写成契约 |
| sink 写网络或文件却没有失败策略 | 指标采集失败可能阻断校验或覆盖原异常 | 副作用被装饰器语法隐藏 |
| 用全局列表保存计时事件 | 测试间状态串扰，重复运行结果变化 | 可变状态没有交给调用方管理 |

## 5. 最佳实践

1. **先写变化清单，再选模式**：把对象创建、算法变化和横切行为分开；没有独立变化原因时，不增加抽象。
2. **让工厂只负责合法创建**：在配置边界拒绝未知类型、额外字段、布尔状态码、越界状态码、空字段和重复字段，策略只处理合法构造参数。
3. **让策略契约保持最小**：本例只要求 `name` 与 `validate()`；不把文件路径、日志器和 JSON 配置塞入每个策略。
4. **让执行器依赖能力而非类名**：执行器循环调用协议，不用 `isinstance()` 区分内置策略；用测试专用新策略证明替换能力。
5. **明确聚合与顺序语义**：本例执行全部策略，并按配置顺序、字段配置顺序输出问题，便于一次看到完整失败信息并保持报告稳定。
6. **装饰器默认使用 `wraps`**：除非有非常明确的理由改变函数身份，否则保留名称、文档与可检查签名。
7. **注入不可控副作用**：时间、事件 sink、文件和网络都由外部提供；单元测试使用确定性实现，真实适配器另做边界测试。
8. **异常策略写进测试**：至少验证正常返回不变、原异常继续传播、异常路径是否记录事件，以及不同装饰器实例不共享状态。
9. **限制装饰器层数**：两三层尚可读时仍需记录顺序语义；层数增加到难以追踪时，改用显式应用服务或包装对象。
10. **允许局部使用简单分支**：创建边界中的少量 `if` 比过早引入动态插件注册表更容易 review；达到真实扩展规模后再重构。

## 6. 练习题

### 练习 1：新增响应头策略（基础）

**对应验收：模块①·设计模式；Req 22.5**

需求：新增 `HeaderStrategy`，检查响应 body 中是否存在 `trace_id`；先直接把对象注入 `ValidationRunner`，不得修改 `runner.py`。为成功和失败各写一个测试。

验收重点：如果执行器需要新增 `elif rule_type == "header"`，说明策略边界没有真正建立。思考为什么“直接注入对象”暂时不要求修改工厂。

### 练习 2：选择函数工厂、工厂类或不使用工厂（进阶）

**对应验收：模块①·设计模式；Req 22.5**

分别为三个场景选择方案，并写出至少一项优点、一项缺点：

1. 两种规则全部在代码中直接构造，半年没有变化；
2. 五种规则来自 JSON，参数结构不同，但类型集合固定；
3. 第三方团队能在运行时注册规则，创建策略还需要共享 schema 仓库。

参考判断方向：场景 1 可不使用工厂，调用路径最直接；场景 2 适合本章的函数工厂，集中校验且成本低；场景 3 才可能值得带注册表和依赖的工厂类。答案不是看模式数量，而是说明变化规模与维护成本。

### 练习 3：设计重试与计时顺序（挑战）

**对应验收：模块①·设计模式；Req 22.5**

假设已有 `@retry` 和本章 `@timed`：

- 方案 A：`@timed` 在外、`@retry` 在内；
- 方案 B：`@retry` 在外、`@timed` 在内。

画出两次失败、第三次成功时的调用顺序，说明每种方案会产生几个计时事件、每个事件表示总耗时还是单次耗时，并选择适合“报告整个校验动作耗时”的方案。再补充 sink 抛异常时应阻断还是降级的决定及理由。

方案取舍：A 只产生一次总耗时，报告简洁但看不到单次波动；B 产生三次单次耗时，诊断更细但指标数量更多。若目标是用户看到的整体耗时，优先 A；若目标是分析每次重试，选择 B 或同时采集两层不同名称的指标。

## 7. 小结与自检

- 工厂把不可信配置转换为合法策略；少量固定规则使用函数工厂即可，不必机械创建工厂类。
- 策略把独立算法封装成满足最小协议的对象，使执行器按统一方式组合规则，而不是不断增加类型分支。
- 装饰器适合计时等调用前后横切行为；`wraps` 保留元数据，注入 clock/sink 让副作用可控、测试确定。
- 装饰器的嵌套顺序会改变重试、计时和异常语义；被装饰函数异常与 sink 自身异常必须分别设计。
- 简单 `if`、映射表和“不使用模式”都是合法选项；只有模式带来的扩展与测试收益超过理解成本时才使用。

自检问题（可选择跳过）：

1. 为什么本例工厂里仍有 `if`，却仍然可以说策略模式消除了扩展困难？被消除的是哪一层的分支？
2. 如果计时 sink 改成写远程监控服务，你会让监控失败阻断接口校验吗？怎样测试你选择的异常策略？
