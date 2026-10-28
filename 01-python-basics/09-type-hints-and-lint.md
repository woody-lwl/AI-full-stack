# 1.9 类型注解与代码规范

> 运行环境：Python 3.12+
>
> 对应模块验收：模块①·类型注解；Req 1.2

**Validates: Requirements 1.2**

## 目录
- [1. 场景引入](#1-场景引入)
- [2. 可运行代码示例](#2-可运行代码示例)
- [3. 设计思考](#3-设计思考)
- [4. 常见坑](#4-常见坑)
- [5. 最佳实践](#5-最佳实践)
- [6. 练习题](#6-练习题)
- [7. 小结与自检](#7-小结与自检)

## 1. 场景引入

回归测试开始前，脚本需要从 JSON 读取接口用例，再按运行环境、优先级、标签和启用状态筛出待执行集合。下面这种代码“暂时能跑”，但 review 者必须猜 `Data` 的结构、`ENV=None` 的含义和返回内容：

```python
# 故意展示的整改前代码

def GET_CASES(Data, ENV=None):
    result=[]
    for x in Data:
        if x['enabled'] == True and ENV in x['environments']:
            result.append(x)
    return result
```

它还把不可信的外部字典传遍整个程序：字段拼错只能运行到该分支才暴露，字符串 `"true"` 甚至可能被误当成布尔配置。

**一句话直觉：类型注解把函数的数据契约写给人和工具，格式化与 lint 把团队重复争论变成自动检查。**

但要先记住边界：**类型注解通常不会在运行时替你拦截错误数据。** `case_id: str` 主要服务阅读、IDE 和静态检查；JSON 是否真的给了字符串，仍要靠运行时校验和测试确认。

先预测：示例有 5 条用例，在 `staging` 中选择 P0/P1、带 `smoke` 标签且启用的用例，最终会留下几条？

## 2. 可运行代码示例

完整工程位于 [`code/09-type-hints-and-lint/`](code/09-type-hints-and-lint/)：

```text
09-type-hints-and-lint/
├── pyproject.toml
├── README.md
├── examples/
│   ├── cases.json
│   └── problem-code-before.txt
├── src/test_case_selector/
│   ├── __init__.py
│   ├── __main__.py
│   ├── loader.py
│   ├── models.py
│   └── selector.py
└── tests/
    ├── test_loader.py
    ├── test_main.py
    └── test_selector.py
```

### 2.1 用精确类型表达领域契约

`models.py` 的核心模型：

```python
from dataclasses import dataclass
from typing import Literal

Environment = Literal["test", "staging", "prod"]
Priority = Literal["P0", "P1", "P2"]


@dataclass(frozen=True, slots=True)
class TestCase:
    """一条通过运行时校验的接口测试用例。"""

    case_id: str
    priority: Priority
    environments: frozenset[Environment]
    tags: frozenset[str]
    is_enabled: bool
    owner: str | None


@dataclass(frozen=True, slots=True)
class SelectionConfig:
    """选择测试用例时使用的筛选条件。"""

    environment: Environment
    allowed_priorities: frozenset[Priority]
    required_tags: frozenset[str]
    include_disabled: bool = False
```

这里展示了几类常用契约：

- `str | None` 明确 owner 可能缺失；它不等于“可以传任何东西”。
- `frozenset[str]` 写出容器元素类型，并表达去重、不可变和集合运算需求。
- `Literal` 把合法字符串限制在少数稳定值，比宽泛的 `str` 更早发现拼写错误。
- dataclass 适合在校验后承载稳定领域数据。若数据只需要保持字典形状并与现有字典 API 交互，可考虑 `TypedDict`；它同样不做运行时校验。
- `is_enabled`、`include_disabled` 直接表达布尔语义，比 `status`、`flag` 更少歧义。

### 2.2 外部数据先用 `object` 接住，再收窄

`loader.py` 的公共边界都有完整参数与返回值注解：

```python
import json
from collections.abc import Mapping
from pathlib import Path

from .models import Environment, Priority, TestCase


class CaseDataError(ValueError):
    """外部测试用例数据不符合约定。"""


def load_test_cases(path: Path) -> tuple[TestCase, ...]:
    """从 UTF-8 JSON 文件加载并校验测试用例。"""
    try:
        raw: object = json.loads(path.read_text(encoding="utf-8"))
    except (OSError, json.JSONDecodeError) as error:
        raise CaseDataError(f"无法读取测试用例: {path}") from error
    return parse_test_cases(raw)


def parse_test_cases(raw: object) -> tuple[TestCase, ...]:
    """收窄未知外部数据的类型，并转换为不可变 TestCase。"""
    if not isinstance(raw, list):
        raise CaseDataError("测试用例 JSON 顶层必须是数组")

    cases: list[TestCase] = []
    seen_case_ids: set[str] = set()
    for index, item in enumerate(raw):
        context = f"第 {index + 1} 条用例"
        values = _require_mapping(item, context)
        case = TestCase(
            case_id=_require_non_empty_str(values.get("case_id"), f"{context}.case_id"),
            priority=_parse_priority(values.get("priority"), context),
            environments=_parse_environments(values.get("environments"), context),
            tags=_parse_tags(values.get("tags"), context),
            is_enabled=_require_bool(values.get("enabled"), f"{context}.enabled"),
            owner=_parse_optional_owner(values.get("owner"), context),
        )
        if case.case_id in seen_case_ids:
            raise CaseDataError(f"case_id 重复: {case.case_id}")
        seen_case_ids.add(case.case_id)
        cases.append(case)
    return tuple(cases)
```

正式源码中的私有辅助函数继续用 `isinstance` 和 `match` 检查并收窄字符串、布尔值、环境和优先级。例如：

```python
def _parse_priority(value: object, context: str) -> Priority:
    match value:
        case "P0" | "P1" | "P2":
            return value  # 经过分支后，mypy 知道它属于 Priority
        case _:
            raise CaseDataError(f"{context}.priority 必须是 P0、P1 或 P2")
```

为什么入口使用 `object` 而不是 `Any`？`object` 表示“值存在，但使用前必须证明它是什么”；`Any` 会跳过许多检查，并沿调用链扩散。`cast(TestCase, raw)` 也不是校验：它只让静态检查器相信你，不会检查运行时字段。因此本例只在实际判断后构造 dataclass。

### 2.3 按“需要的能力”选择接口类型

`selector.py`：

```python
from collections.abc import Iterable, Sequence

from .models import Priority, SelectionConfig, SelectionSummary, TestCase

_PRIORITY_ORDER: dict[Priority, int] = {"P0": 0, "P1": 1, "P2": 2}


def _priority_rank(priority: Priority) -> int:
    """返回优先级用于排序的显式序号。"""
    return _PRIORITY_ORDER[priority]


def select_cases(
    cases: Iterable[TestCase], config: SelectionConfig
) -> tuple[TestCase, ...]:
    """选择满足环境、优先级、标签和启用状态的用例。"""
    selected = (case for case in cases if _matches(case, config))
    return tuple(
        sorted(
            selected,
            key=lambda case: (_priority_rank(case.priority), case.case_id),
        )
    )


def summarize_selection(
    all_cases: Sequence[TestCase],
    selected_cases: Sequence[TestCase],
    config: SelectionConfig,
) -> SelectionSummary:
    """使用可计数序列生成选择摘要。"""
    return SelectionSummary(
        config=config,
        input_count=len(all_cases),
        selected_case_ids=tuple(case.case_id for case in selected_cases),
    )
```

显式的 `_priority_rank(priority: Priority) -> int` 把排序规则和类型边界集中在一起。`format_summary()` 先将排序结果绑定为 `list[Priority]`，再执行 `",".join(...)`：这能阻止外层 `join` 的字符串上下文把 `sorted()` 元素类型反向拓宽为普通 `str`，随后可安全使用 `key=_priority_rank`。对应实现如下：

```python
ordered_priorities: list[Priority] = sorted(
    config.allowed_priorities, key=_priority_rank
)
priorities = ",".join(ordered_priorities)
```

`select_cases()` 使用 `_priority_rank(case.priority)`，让业务排序与摘要排序共享同一规则。这里的重点是参数能力：`select_cases()` 只遍历一次，因此接收更宽的 `Iterable[TestCase]`，生成器也能传入；摘要需要 `len()`，所以接收 `Sequence[TestCase]`。不要为了省事把所有参数都写成 `list`。

公开 API 的“完整注解”包括每个参数和返回值；无返回值函数也应写 `-> None`。私有函数同样建议注解，本项目由 mypy 配置强制所有函数完成注解。

### 2.4 最小自动化质量门禁

`pyproject.toml` 固定 Python 3.12 和精确工具版本，并把规则留在仓库：

```toml
[project.optional-dependencies]
dev = [
    "mypy==1.16.1",
    "pytest==8.4.2",
    "ruff==0.11.13",
]

[tool.mypy]
python_version = "3.12"
disallow_untyped_defs = true
disallow_incomplete_defs = true
no_implicit_optional = true
warn_return_any = true
warn_unused_ignores = true

[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "B", "N", "UP", "SIM", "RUF"]
```

安装、运行和检查：

```bash
cd codelearn/01-python-basics/code/09-type-hints-and-lint
/opt/homebrew/bin/python3.12 -m venv .venv
.venv/bin/python -m pip install "setuptools==75.8.0"
.venv/bin/python -m pip install --no-build-isolation -e ".[dev]"
.venv/bin/python -m test_case_selector
.venv/bin/python -m pytest --color=no
.venv/bin/python -m mypy src tests
.venv/bin/python -m ruff check .
.venv/bin/python -m ruff format --check .
```

示例预期输出：

```text
environment=staging priorities=P0,P1 required_tags=smoke include_disabled=False
input=5 selected=2 skipped=3
selected_cases=login-smoke,payment-smoke
```

各工具不是同一件事的不同写法：pytest 执行代码验证行为；mypy 不运行代码，检查类型在调用链上是否一致；Ruff lint 找未使用导入、可疑写法、命名等问题；Ruff formatter 统一空格、换行和引号，`--check` 只验证、不改文件。先用 `ruff format .` 自动整改格式，再用四条门禁命令验收，是常见本地流程。

## 3. 设计思考

### 3.1 五道防线各自负责什么？

| 手段 | 擅长发现 | 发现不了或不擅长 |
|---|---|---|
| 只靠 review | 领域规则、设计取舍、命名是否表达业务 | 容易漏掉机械问题，结果受时间和经验影响 |
| 自动格式化 | 空格、换行、引号等纯格式差异 | 业务错误、类型错误、糟糕命名 |
| lint | 未使用导入、可疑表达式、部分命名和缺陷模式 | 跨函数业务行为是否正确 |
| 静态类型检查 | 参数/返回值不匹配、遗漏 `None` 分支、错误容器元素 | JSON 真实内容、复杂运行时业务规则 |
| 测试 | 给定输入下的真实行为、异常和边界 | 没执行到的路径，且不能替代可读契约 |

因此目标不是选择“最强工具”，而是让机械问题自动化，把 review 时间留给设计和业务。工具全绿也不证明程序绝对正确，但能持续降低低级缺陷和理解成本。

### 3.2 dataclass、TypedDict 与普通 dict 的取舍

| 方案 | 优点 | 适用场景 |
|---|---|---|
| dataclass | 属性访问清晰，可冻结，可承载领域含义 | 校验后在业务层长期流动的数据 |
| `TypedDict` | 保留字典形状，适合逐步给现有 dict 加类型 | API 载荷适配层、仍需字典操作的代码 |
| `dict[str, object]` | 适合未知外部数据的临时容器 | 运行时校验边界，不应扩散到核心逻辑 |
| `dict[str, Any]` | 迁移旧代码方便 | 只能作为短期过渡，并应限制传播范围 |

本例不把 JSON 直接声明为 `TypedDict`，因为声明不会让外部数据自动可信；先校验，再构造 `TestCase`，业务层就不必反复检查字段。

### 3.3 渐进式类型注解，而不是类型炫技

旧项目不必一次注解每个局部变量。建议顺序是：

1. 先注解公共函数、文件/API 边界和核心数据模型，让调用者先获得稳定契约；
2. 再处理核心数据流中 mypy 暴露的 `None`、宽泛容器和错误返回值；
3. 最后按收益补充内部辅助函数，不为表达一个简单规则堆叠复杂泛型。

本项目规模小，所以直接要求全部函数有注解；大型遗留项目可按模块逐步扩大检查范围。严格度应可执行、可持续，而不是第一天开启所有规则后用大量 ignore 压过去。

### 3.4 命名和注释也是契约

模块、函数和变量用 `snake_case`，类用 `PascalCase`，稳定常量用 `UPPER_CASE`。名称优先表达业务：`selected_cases` 比 `result` 清楚，`is_enabled` 比 `flag` 清楚。注释应解释“为什么注入时钟”“为什么返回 tuple”等决策，不要写 `count += 1  # count 加一` 这种复述代码的注释。

## 4. 常见坑

| 坑 | 后果 | 根本原因 |
|---|---|---|
| 以为注解会校验 JSON | 错误字段进入业务层后才爆炸 | 混淆静态契约与运行时校验 |
| 到处使用 `Any` | 类型错误沿调用链静默传播 | 用“关闭检查”代替类型建模 |
| 用 `cast()` 包住外部数据 | 静态检查绿了，运行时仍不安全 | `cast` 不会执行验证 |
| 写 `owner: str = None` | 契约自相矛盾，mypy 报错 | 没显式建模缺失值 |
| 参数统一写 `list` | 生成器、tuple 等合法输入被无故拒绝 | 按具体容器而不是所需能力建模 |
| 类型越复杂越专业 | review 成本比收益高 | 忘记类型首先服务读者 |
| 用 `Data`、`flag`、`tmp` 表达核心业务 | 类型正确但语义仍难懂 | 把命名规范当纯格式问题 |
| 注释逐行翻译代码 | 修改后注释容易过期 | 没记录原因和约束 |
| 只运行 formatter | 代码很整齐，但类型和行为仍可能错误 | 把不同质量工具混为一谈 |
| 大面积 `# type: ignore` | 真问题被永久隐藏 | 为了“全绿”绕过门禁 |

## 5. 最佳实践

1. **公共边界先完整注解**：参数与返回值都写清楚，`None` 明确进入联合类型。
2. **外部输入先验证再建模**：用 `object` 接未知值，经过 `isinstance`/`match` 收窄，再构造 dataclass 或 TypedDict 约定的数据。
3. **按能力选抽象类型**：只遍历用 `Iterable`，需要长度和下标时用 `Sequence`，避免无意义绑定到 `list`。
4. **限制 `Any` 的半径**：无法避免时只停留在适配边界，尽快转换；不要用 `cast` 假装完成校验。
5. **命名表达领域意图**：遵循 snake_case、PascalCase、UPPER_CASE，并给布尔名称使用 `is_`、`has_`、`include_` 等语义前缀。
6. **注释解释原因**：代码说明“做什么”，注释补充“为什么这样做”和重要边界。
7. **共享一份工具配置**：把 Python 版本、规则与精确依赖写进 `pyproject.toml`，避免每个人本地标准不同。
8. **四项门禁都运行**：格式、lint、类型和测试互补；不要以其中一项通过替代其他检查。
9. **渐进提高严格度**：从公共 API 和核心数据流开始，以少量、可修复的反馈推动迁移。

## 6. 练习题

### 练习 1：整改问题函数（基础）

**对应验收：模块①·类型注解；Req 1.2**

打开 `examples/problem-code-before.txt`，先预测 Ruff 与 mypy 会关注哪些问题，再将它复制到临时 `.py` 文件整改：改名为 `select_enabled_cases`，为参数和返回值补全类型，明确环境是否允许 `None`，最后运行 formatter、lint 和 mypy。不要直接给它加 `Any`。

### 练习 2：增加 owner 筛选（进阶）

**对应验收：模块①·类型注解；Req 1.2**

为 `SelectionConfig` 增加 `owner: str | None`：`None` 表示不限制 owner，否则只选择对应负责人。覆盖 owner 缺失、配置不限制和精确匹配三种测试。思考：这个条件应在 JSON loader 还是 selector 中实现？为什么？

### 练习 3：给旧字典边界增加 TypedDict（挑战）

**对应验收：模块①·类型注解；Req 1.2**

假设下游报告库必须接收字典。定义只用于“校验后输出”的 `TypedDict`，写 `to_report_row(case: TestCase) -> ReportRow`。不要用它直接声明未经校验的 `json.loads()` 结果；用测试证明 owner 为 `None` 时输出策略符合你的约定。

## 7. 小结与自检

- 类型注解是给读者、IDE 和静态检查器的数据契约，通常不是运行时验证器。
- `str | None`、容器泛型、`Literal`、dataclass 和 `collections.abc` 接口类型用于表达不同约束，选择够用且易读的类型即可。
- 未知外部数据适合先作为 `object`，通过真实判断完成类型收窄；`Any` 与 `cast` 不能替代验证。
- 命名、必要注释、formatter、lint、mypy 和 pytest 共同提高可维护性，各自不能互相替代。
- 渐进式注解优先公共边界和核心数据流，不以复杂类型或大量 ignore 追求表面严格。

自检问题（可选择跳过）：

1. 为什么 `TypedDict` 能帮助 mypy，却不能证明 JSON 中的 `enabled` 一定是布尔值？
2. `select_cases()` 为什么接收 `Iterable[TestCase]`，而 `summarize_selection()` 接收 `Sequence[TestCase]`？
