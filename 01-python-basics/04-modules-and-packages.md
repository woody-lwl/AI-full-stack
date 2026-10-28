# 1.4 模块与包

> 运行环境：Python 3.12+
>
> 对应模块验收：模块①·模块与包

**Validates: 模块①·模块与包**

## 目录
- [1. 场景引入](#1-场景引入)
- [2. 可运行代码示例](#2-可运行代码示例)
- [3. 设计思考](#3-设计思考)
- [4. 常见坑](#4-常见坑)
- [5. 最佳实践](#5-最佳实践)
- [6. 练习题](#6-练习题)
- [7. 小结与自检](#7-小结与自检)

## 1. 场景引入

你写了一个接口测试辅助脚本：先读取 JSON 用例，再校验 `case_id`、接口路径和预期状态码，最后打印文本报告。最初只有几十行，放在 `tool.py` 中很方便；后来加载规则、字段规则和报告格式都在增长，任何改动都要在同一个文件里寻找位置，测试也只能从头跑完整流程。

超单文件的根本问题不是“文件太长”，而是**多种变化原因被放在了同一个边界里**：JSON 格式变化会影响加载，业务字段变化会影响校验，展示要求变化会影响报告。它们本应能独立修改和测试。

**一句话直觉：模块是一个可导入的 `.py` 文件，包是组织多个模块的目录；按职责拆分，就是让每种变化找到自己的家，再通过 import 组合起来。**

本章沿用 1.3 的职责单一函数，不引入类。我们把脚本拆成：

- `loader.py`：只读取并检查 JSON 容器结构；
- `validator.py`：只校验单条用例字段；
- `reporter.py`：只把校验结果变成文本；
- `__main__.py`：只编排完整流程；
- `__init__.py`：声明包希望使用者看到的公共 API。

## 2. 可运行代码示例

完整工程位于 [`code/04-modules-and-packages/`](code/04-modules-and-packages/)。运行前请先预测：三条示例用例中有几条通过？`order-002` 与 `refund-003` 分别会报告什么？

### 2.1 从单文件的数据流识别职责

重构前，所有步骤通常串在一个文件里：

```python
import json
from pathlib import Path

case_file = Path("examples/test_cases.json")
with case_file.open(encoding="utf-8") as file:
    test_cases = json.load(file)

results: list[tuple[str, list[str]]] = []
for test_case in test_cases:
    errors: list[str] = []
    for field in ("case_id", "endpoint", "expected_status"):
        if field not in test_case:
            errors.append(f"缺少必填字段: {field}")

    expected_status = test_case.get("expected_status")
    if expected_status is not None and (
        not isinstance(expected_status, int)
        or isinstance(expected_status, bool)
        or not 100 <= expected_status <= 599
    ):
        errors.append("expected_status 必须是 100~599 的整数")

    case_id = test_case.get("case_id", "unknown")
    results.append((str(case_id), errors))

passed = sum(not errors for _, errors in results)
print(f"总数: {len(results)} | 通过: {passed} | 失败: {len(results) - passed}")
for case_id, errors in results:
    print(f"[{'PASS' if not errors else 'FAIL'}] {case_id}")
    for error in errors:
        print(f"  - {error}")
```

这段代码能运行，但读取、规则、格式和流程没有边界。重构时不要按“每 30 行切一个文件”，而应先给每段逻辑写出职责名称；能独立命名、独立变化、独立测试的职责，才适合成为模块。

### 2.2 工程结构

```text
04-modules-and-packages/
├── .gitignore
├── README.md
├── pyproject.toml
├── examples/
│   └── test_cases.json
├── src/
│   └── api_test_toolkit/
│       ├── __init__.py
│       ├── __main__.py
│       ├── loader.py
│       ├── reporter.py
│       └── validator.py
└── tests/
    ├── test_entrypoint.py
    ├── test_loader.py
    ├── test_package.py
    ├── test_reporter.py
    └── test_validator.py
```

这里每个 `.py` 文件都是一个**模块**，`api_test_toolkit/` 是一个**包**。`src/` 布局要求我们先安装项目再运行，避免测试时意外导入项目根目录中的同名文件。

### 2.3 加载模块：只处理文件和 JSON 容器结构

`src/api_test_toolkit/loader.py`：

```python
"""负责从 JSON 文件读取接口测试用例。"""

import json
from pathlib import Path
from typing import TypeAlias

TestCase: TypeAlias = dict[str, object]


def load_test_cases(path: str | Path) -> list[TestCase]:
    """读取测试用例列表，并校验 JSON 顶层与记录的基本结构。"""
    file_path = Path(path)
    with file_path.open(encoding="utf-8") as file:
        data = json.load(file)

    if not isinstance(data, list):
        raise ValueError("测试用例文件的顶层必须是 JSON 数组")

    test_cases: list[TestCase] = []
    for index, item in enumerate(data, start=1):
        if not isinstance(item, dict):
            raise ValueError(f"第 {index} 条测试用例必须是 JSON 对象")
        test_cases.append(item)

    return test_cases
```

加载模块只关心“能否得到一组字典”。它不判断 `expected_status` 是否有效，因为那是业务字段规则，应该由校验模块负责。

### 2.4 校验模块：只处理字段规则

`src/api_test_toolkit/validator.py`：

```python
"""负责校验单条接口测试用例的字段规则。"""

from collections.abc import Mapping

REQUIRED_FIELDS: tuple[str, ...] = ("case_id", "endpoint", "expected_status")


def validate_test_case(test_case: Mapping[str, object]) -> list[str]:
    """返回单条用例的全部字段问题；空列表表示校验通过。"""
    errors = [
        f"缺少必填字段: {field}"
        for field in REQUIRED_FIELDS
        if field not in test_case
    ]

    if "case_id" in test_case:
        case_id = test_case["case_id"]
        if not isinstance(case_id, str) or not case_id.strip():
            errors.append("case_id 必须是非空字符串")

    if "endpoint" in test_case:
        endpoint = test_case["endpoint"]
        if not isinstance(endpoint, str) or not endpoint.startswith("/"):
            errors.append("endpoint 必须是以 / 开头的字符串")

    if "expected_status" in test_case:
        expected_status = test_case["expected_status"]
        is_valid_status = (
            isinstance(expected_status, int)
            and not isinstance(expected_status, bool)
            and 100 <= expected_status <= 599
        )
        if not is_valid_status:
            errors.append("expected_status 必须是 100~599 的整数")

    return errors
```

该模块不读取文件，也不打印。只要传入映射，就可以直接单元测试字段规则。这正是 1.3“函数职责单一”在跨文件组织中的延续。

### 2.5 报告模块：只负责格式

`src/api_test_toolkit/reporter.py`：

```python
"""负责把用例校验结果格式化为文本报告。"""

from collections.abc import Sequence
from typing import TypeAlias

ValidationResult: TypeAlias = tuple[str, Sequence[str]]


def build_text_report(results: Sequence[ValidationResult]) -> str:
    """生成包含汇总数据和逐条结果的可打印文本。"""
    total = len(results)
    passed = sum(not errors for _, errors in results)
    failed = total - passed

    lines = [
        "接口测试用例校验报告",
        "====================",
        f"总数: {total} | 通过: {passed} | 失败: {failed}",
    ]

    for case_id, errors in results:
        if not errors:
            lines.append(f"[PASS] {case_id}")
            continue

        lines.append(f"[FAIL] {case_id}")
        lines.extend(f"  - {error}" for error in errors)

    return "\n".join(lines)
```

它返回字符串而不是直接 `print`，因此测试可以精确比较结果，未来也能把同一文本写入文件或发送到其他系统。

### 2.6 用 `__init__.py` 声明公共 API

`src/api_test_toolkit/__init__.py`：

```python
"""接口测试用例工具包的稳定公共 API。"""

from .loader import load_test_cases
from .reporter import build_text_report
from .validator import validate_test_case

__all__ = ["build_text_report", "load_test_cases", "validate_test_case"]
```

这里的 `.loader` 是**显式相对导入**：“从当前包的 `loader` 模块导入”。包的使用者可以写：

```python
from api_test_toolkit import load_test_cases, validate_test_case
```

而不必知道函数实际存在哪个内部模块。`__all__` 记录受支持的公共名称；它不是安全机制，也不会阻止别人直接导入内部名称，但能表达维护者承诺的 API 边界。

### 2.7 用 `__main__.py` 组合流程

`src/api_test_toolkit/__main__.py`：

```python
"""支持通过 python -m api_test_toolkit 运行完整示例。"""

from pathlib import Path

from api_test_toolkit import build_text_report, load_test_cases, validate_test_case

EXAMPLE_FILE = Path(__file__).resolve().parents[2] / "examples" / "test_cases.json"


def main() -> None:
    """编排加载、校验和报告模块，保持入口本身只负责流程。"""
    test_cases = load_test_cases(EXAMPLE_FILE)
    results: list[tuple[str, list[str]]] = []

    for index, test_case in enumerate(test_cases, start=1):
        raw_case_id = test_case.get("case_id")
        case_id = raw_case_id if isinstance(raw_case_id, str) else f"record-{index}"
        results.append((case_id, validate_test_case(test_case)))

    print(build_text_report(results))


if __name__ == "__main__":
    main()
```

入口使用**绝对导入**，从包的公共 API 取函数。执行 `python -m api_test_toolkit` 时，Python 会找到这个包并运行其中的 `__main__.py`。入口只做流程编排，没有复制加载、校验或报告规则。

示例数据 `examples/test_cases.json`：

```json
[
  {
    "case_id": "login-001",
    "endpoint": "/api/login",
    "expected_status": 200
  },
  {
    "case_id": "order-002",
    "expected_status": 201
  },
  {
    "case_id": "refund-003",
    "endpoint": "/api/refunds/3",
    "expected_status": 700
  }
]
```

### 2.8 安装、运行与测试

`pyproject.toml` 固定构建和测试依赖：

```toml
[build-system]
requires = ["setuptools==75.8.0"]
build-backend = "setuptools.build_meta"

[project]
name = "api-test-toolkit"
version = "0.1.0"
description = "用于学习 Python 模块拆分与包导入的接口测试用例工具"
readme = "README.md"
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
cd codelearn/01-python-basics/code/04-modules-and-packages
/opt/homebrew/bin/python3.12 -m venv .venv
.venv/bin/python -m pip install "setuptools==75.8.0"
.venv/bin/python -m pip install --no-build-isolation -e ".[test]"
.venv/bin/python -m api_test_toolkit
.venv/bin/python -m pytest --color=no
```

入口预期输出：

```text
接口测试用例校验报告
====================
总数: 3 | 通过: 1 | 失败: 2
[PASS] login-001
[FAIL] order-002
  - 缺少必填字段: endpoint
[FAIL] refund-003
  - expected_status 必须是 100~599 的整数
```

测试分别验证加载、校验、报告模块，覆盖正常数据、缺字段、非法字段格式、非法 JSON、错误顶层结构与空报告；包测试验证公共导入，入口集成测试用真实子进程执行 `python -m api_test_toolkit`。

## 3. 设计思考

### 3.1 import 到底做了什么？

可以把 import 直觉化为三步：

1. Python 根据模块名，在导入搜索路径 `sys.path` 中寻找对应模块或包；
2. 第一次找到时执行该模块的顶层代码，创建模块对象；
3. 把模块缓存到 `sys.modules`，同一进程再次导入通常复用缓存，而不是重新执行。

`src/` 项目不能依赖“碰巧从某个目录运行”。执行可编辑安装后，解释器才能稳定找到 `api_test_toolkit`。这也是 README 先安装再运行的原因。

导入模块会执行顶层代码，所以模块顶层适合放常量和函数定义，不适合直接发网络请求、读取业务文件或启动流程。入口逻辑放在 `main()` 中，并由 `if __name__ == "__main__"` 控制。

### 3.2 绝对导入还是相对导入？

| 方案 | 示例 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|---|
| 绝对导入 | `from api_test_toolkit import load_test_cases` | 来源清晰，跨目录阅读容易 | 包名修改时要同步更新 | 入口、测试、包外调用 |
| 显式相对导入 | `from .loader import load_test_cases` | 简洁，明确来自当前包 | 相对层级过深时难读；文件不能随意直接执行 | 包内部相邻模块 |

本例在 `__init__.py` 用相对导入聚合相邻模块，在 `__main__.py` 用绝对导入消费公共 API。两者都正确，关键是团队保持一致并避免难读的多层相对路径。

### 3.3 `__init__.py` 的作用是什么？

在本项目中，它有两个作用：明确把目录作为常规包使用；把三个公开函数重新导出，形成稳定入口。现代 Python 支持没有 `__init__.py` 的命名空间包，但那主要服务于跨多个目录组合一个包名的场景。普通应用包显式保留它，更容易让初学者、编辑器和工具理解边界。

公共 API 还能降低重构成本：未来把 `build_text_report` 移到别的内部模块，只要 `api_test_toolkit.__init__` 继续导出它，包外调用方式就不必改变。不要把所有内部常量和辅助函数都导出，否则内部实现会变成难以修改的隐性承诺。

### 3.4 为什么按职责拆，而不是按文件大小拆？

文件行数只能提示“可能需要整理”，不能决定边界。`loader.py` 即使以后有 100 行，只要都围绕输入文件解析，仍可能保持单一职责；反过来，一个 30 行文件若同时读文件、发请求和格式化报告，也已经混合了多个变化原因。

| 拆分方式 | 优点 | 风险 | 选择建议 |
|---|---|---|---|
| 单文件 | 小脚本启动快，阅读无需跳转 | 增长后变化互相干扰 | 一次性、逻辑很少的脚本 |
| 按职责拆模块 | 边界清楚，可独立测试与复用 | 模块太碎会增加跳转 | 多种规则会独立变化时，本章采用 |
| 按技术层做更多子包 | 大项目可分层和控制依赖 | 小项目容易过度设计 | 模块数量和团队规模明显增长后 |

### 3.5 如何避免循环导入？

循环导入常见于 `loader` 导入 `validator`，同时 `validator` 又导入 `loader`。Python 第一次加载模块时，对方可能还没执行完，于是出现“部分初始化模块”错误。

本例采用单向数据流：

```text
__main__ → 公共 API → loader / validator / reporter
```

三个业务模块彼此不依赖，由入口拿到加载结果，再传给校验和报告。若两个模块必须共享简单类型或常量，可把共享定义放入更底层、且不反向依赖它们的模块；不要靠把 import 随意塞进函数来掩盖错误边界。

## 4. 常见坑

| 坑 | 后果 | 根本原因 |
|---|---|---|
| 在项目根目录直接运行包内文件 | 相对导入失败，或导入到错误的同名模块 | 绕过了包上下文与标准入口 |
| 依赖当前工作目录修改 `sys.path` | 换目录、CI 或同事电脑就无法导入 | 没有正确安装 `src/` 项目 |
| 在模块顶层读取文件或执行流程 | 仅仅 import 就产生副作用，测试难隔离 | 没把“定义”与“运行入口”分开 |
| `loader` 同时校验业务字段并格式化报告 | 一条规则变化牵动多个职责，单测困难 | 仍按执行顺序堆代码，没有按变化原因拆分 |
| 两个模块互相 import | 出现部分初始化错误，依赖方向难理解 | 模块边界或共享定义位置不合理 |
| `__init__.py` 导出所有内部名称 | 包外代码依赖实现细节，后续难重构 | 没有区分公共 API 与内部实现 |
| 同时存在文件 `api_test_toolkit.py` 和同名包 | 导入目标容易混淆 | 模块和包命名冲突 |
| 为每个小函数新建一个文件 | 阅读需要频繁跳转，目录失去业务语义 | 把“拆分”误解为文件越小越好 |

## 5. 最佳实践

1. **先画数据流，再建文件**：本例数据按“加载 → 校验 → 报告”流动，入口负责组合，依赖方向一眼可见。
2. **模块名使用业务职责名**：`loader`、`validator`、`reporter` 比 `utils1`、`common` 更能说明放什么、不放什么。
3. **保持模块导入无业务副作用**：顶层只定义常量、类型和函数；实际运行放到 `main()`。
4. **优先安装包，不手改搜索路径**：开发期用 `pip install -e`，让普通运行、pytest 和 CI 遵循同一导入规则。
5. **绝对导入与显式相对导入都要可读**：包外和入口优先绝对导入；包内相邻模块可用单点相对导入，避免多层跳转。
6. **公共 API 少而稳定**：通过 `__init__.py` 和 `__all__` 表达支持的入口，辅助实现留在具体模块。
7. **用入口集成测试验证真正的包运行方式**：单元测试证明各模块职责，子进程执行 `python -m package` 证明安装、导入和编排能连成闭环。
8. **测试异常边界而不混淆职责**：非法 JSON 属于加载测试，缺字段属于校验测试，文本布局属于报告测试。

这些做法适合会持续维护的脚本和小工具。只有十几行、不会复用的一次性脚本可以保留单文件；拆分的目标是降低变化成本，而不是追求目录数量。

## 6. 练习题

### 练习 1：重构真实单文件脚本（基础）

**对应验收：模块①·模块与包**

从你工作中选一个同时包含“读输入、处理数据、输出结果”的单文件测试脚本，先不要写代码，先给三类职责各写一句定义。然后复制脚本到练习目录并完成：

- 至少拆成三个职责模块和一个 `__main__.py`；
- 核心函数保留类型注解，不提前改成类；
- 在 `__init__.py` 只导出调用者真正需要的函数；
- 为每个职责模块写至少一个 pytest；
- 从项目根目录执行 `python -m 你的包名`。

验收标准：同事只看目录和模块名，就能指出输入、核心规则、输出分别在哪里；修改报告格式时无需改加载模块。

### 练习 2：增加 CSV 加载模块（进阶）

**对应验收：模块①·模块与包**

在练习副本中新建 `csv_loader.py`，读取 `case_id,endpoint,expected_status` 三列，并仍返回 `list[dict[str, object]]`。要求：

- 不修改 `validator.py` 和 `reporter.py` 的核心函数；
- 在公共 API 中加入命名清晰的 CSV 加载函数；
- 给正常 CSV、缺少表头和空文件写测试；
- 解释为什么“统一返回协议”让新加载方式可以复用旧校验模块。

提示：先设计 CSV 加载函数的返回值，再写读取细节；模块能否组合，取决于边界协议是否一致。

## 7. 小结与自检

- 模块是可导入的 `.py` 文件，包用于组织相关模块；拆分后仍由 import 把职责组合成完整流程。
- import 会搜索模块、执行首次加载并缓存；模块顶层应避免读取业务文件等副作用。
- `__init__.py` 可明确包边界并聚合稳定公共 API，`__main__.py` 提供 `python -m package` 入口。
- 绝对导入强调完整来源，相对导入强调当前包内关系；正确安装 `src/` 项目比手改 `sys.path` 更可靠。
- 按变化原因和职责拆分，不按行数机械切文件；保持依赖单向是避免循环导入的关键。
- 本章工程把加载、校验、报告与入口拆成多个模块，并通过公共导入和入口集成测试实证“模块①·模块与包”的验收要求。

自检问题（可选择跳过）：

1. 如果把 `json.load()`、字段校验和 `print()` 都放进 `loader.py`，即使主文件变短了，为什么仍不算按职责拆分？
2. 为什么推荐运行 `python -m api_test_toolkit`，而不是直接运行 `src/api_test_toolkit/__main__.py`？
