# 1.1 环境与项目结构

> 运行环境：Python 3.12+
>
> 对应模块验收：模块①·环境

**Validates: Requirements 1.1**

## 目录
- [1. 场景引入](#1-场景引入)
- [2. 可运行代码示例](#2-可运行代码示例)
- [3. 设计思考](#3-设计思考)
- [4. 常见坑](#4-常见坑)
- [5. 最佳实践](#5-最佳实践)
- [6. 练习题](#6-练习题)
- [7. 小结与自检](#7-小结与自检)

## 1. 场景引入

你写了一个校验测试数据的脚本，在自己电脑上运行正常，于是通过聊天工具发给同事。同事拿到后却发现：不知道该用哪个 Python 版本、缺少第三方库、运行路径不对，也不敢直接修改。最后你只能远程解释“先装这个，再切到那个目录”。

**一句话直觉：规范项目结构就是把“只有作者脑中知道的运行方式”，变成任何人都能照着执行的项目契约。**

本章先不追求复杂业务，只解决三个工程问题：

1. 用虚拟环境隔离每个项目的依赖；
2. 用 `pyproject.toml` 声明项目元数据和测试依赖；
3. 用 `src/` 与 `tests/` 分开存放源码和测试。

完成后，你得到的不是“一个能跑的 `.py` 文件”，而是一个同事可以安装、运行和验证的最小项目。

## 2. 可运行代码示例

示例代码位于 [`code/01-env-and-structure/`](code/01-env-and-structure/)，功能是读取一条 JSON 测试用例并检查必要字段。外层目录 `01-env-and-structure` 用于与本章 Markdown 文件 stem 一一对应；内部发行项目名仍是 `test-case-validator`，Python 包名仍是 `test_case_validator`，两类名称承担不同职责。请先预测：运行示例时会输出什么？然后再执行命令验证。

### 2.1 项目结构

```text
01-env-and-structure/
├── .gitignore
├── README.md
├── pyproject.toml
├── examples/
│   └── valid_case.json
├── src/
│   └── test_case_validator/
│       ├── __init__.py
│       ├── __main__.py
│       ├── loader.py
│       └── validator.py
└── tests/
    ├── test_loader.py
    └── test_validator.py
```

每个位置都有明确职责：

| 路径 | 职责 |
|---|---|
| `pyproject.toml` | 声明 Python 版本、项目元数据、依赖和 pytest 配置 |
| `src/test_case_validator/` | 可交付的业务源码，不混放测试或样例数据 |
| `tests/` | 自动化测试，从项目外部视角验证源码 |
| `examples/` | 可直接试跑的输入样例 |
| `README.md` | 告诉新使用者如何安装、运行、测试 |

### 2.2 先创建隔离环境

进入示例目录后执行：

```bash
cd codelearn/01-python-basics/code/01-env-and-structure
/opt/homebrew/bin/python3.12 -m venv .venv
source .venv/bin/activate
python --version
python -m pip install -e ".[test]"
```

`python -m venv .venv` 会为当前项目创建独立解释器环境。激活后，`python` 和 `pip` 都指向 `.venv`，不会把 pytest 装进系统 Python。

> Windows PowerShell 的激活命令是 `.venv\Scripts\Activate.ps1`。虚拟环境是本机临时产物，项目已在 `.gitignore` 中忽略 `.venv/`，不要提交它。

### 2.3 用 `pyproject.toml` 声明项目与依赖

```toml
[build-system]
requires = ["setuptools==75.8.0"]
build-backend = "setuptools.build_meta"

[project]
name = "test-case-validator"
version = "0.1.0"
description = "用于学习标准 Python 项目结构的最小测试用例校验器"
readme = "README.md"
requires-python = ">=3.12"
dependencies = []

[project.optional-dependencies]
test = ["pytest==8.4.2"]

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
addopts = "-q"
```

为什么依赖要写在文件里，而不是只在终端里 `pip install`？因为终端操作不会跟随代码交付，`pyproject.toml` 才是团队可复现的依赖清单。这里业务代码只用标准库，所以 `dependencies` 为空；pytest 只服务开发和测试，因此放在 `test` 可选依赖组，并固定为 `8.4.2`。

### 2.4 读取和校验一条测试用例

`src/test_case_validator/loader.py`：

```python
import json
from pathlib import Path


def load_test_case(path: Path) -> dict[str, object]:
    """从 JSON 文件读取一条测试用例记录。"""
    with path.open(encoding="utf-8") as file:
        data = json.load(file)

    # 输入必须是一个 JSON 对象，而不是数组、字符串等其他类型。
    if not isinstance(data, dict):
        raise ValueError("测试用例文件的根节点必须是 JSON 对象")
    return data
```

`src/test_case_validator/validator.py`：

```python
from collections.abc import Mapping


def validate_test_case(record: Mapping[str, object]) -> list[str]:
    """校验测试用例记录，返回所有可读的错误信息。"""
    errors: list[str] = []

    name = record.get("name")
    if not isinstance(name, str) or not name.strip():
        errors.append("name 必须是非空字符串")

    expected_status = record.get("expected_status")
    # bool 是 int 的子类，因此需要显式排除 True 和 False。
    if (
        isinstance(expected_status, bool)
        or not isinstance(expected_status, int)
        or not 100 <= expected_status <= 599
    ):
        errors.append("expected_status 必须是 100~599 的整数")

    request = record.get("request")
    if not isinstance(request, Mapping):
        errors.append("request 必须是对象")
    else:
        path = request.get("path")
        if not isinstance(path, str) or not path.startswith("/"):
            errors.append("request.path 必须是以 / 开头的字符串")

    return errors
```

`src/test_case_validator/__main__.py` 是可运行入口：

```python
from pathlib import Path

from test_case_validator.loader import load_test_case
from test_case_validator.validator import validate_test_case


def main() -> None:
    """读取项目自带样例并输出校验结果。"""
    project_root = Path(__file__).resolve().parents[2]
    case_path = project_root / "examples" / "valid_case.json"
    record = load_test_case(case_path)
    errors = validate_test_case(record)

    if errors:
        raise SystemExit("用例校验失败：\n- " + "\n- ".join(errors))

    print(f"用例校验通过：{record['name']}")


if __name__ == "__main__":
    main()
```

激活虚拟环境并安装项目后，运行：

```bash
python -m test_case_validator
```

预期输出：

```text
用例校验通过：查询用户成功
```

### 2.5 用 pytest 一次性验证

`tests/test_validator.py` 中的一个核心测试如下：

```python
from test_case_validator.validator import validate_test_case


def test_valid_case_has_no_errors() -> None:
    record = {
        "name": "查询用户成功",
        "request": {"method": "GET", "path": "/users/42"},
        "expected_status": 200,
    }

    assert validate_test_case(record) == []
```

运行全部测试：

```bash
python -m pytest
```

`python -m pytest` 比直接输入 `pytest` 更明确：它保证使用当前虚拟环境里的 Python 来启动 pytest。

## 3. 设计思考

### 3.1 为什么使用虚拟环境？

不同项目可能依赖同一库的不同版本。如果全部安装到系统 Python，升级项目 A 的依赖可能破坏项目 B。虚拟环境把解释器入口和第三方包隔离在项目自己的 `.venv/` 中，使依赖变化局限于当前项目。

它隔离的是**已安装环境**，而 `pyproject.toml` 记录的是**应该安装什么**：两者配合才能让项目可复现。

### 3.2 为什么把源码放进 `src/`？

如果包直接放在项目根目录，测试可能因为“当前目录恰好可导入”而通过，即使项目没有正确安装。`src/` 布局让源码不再天然出现在导入路径中，更接近用户安装后的真实使用方式，也能避免源码、配置、样例与测试混在一起。

`tests/` 单独存在，则让测试代码不进入业务包。读目录时也能立即判断：哪里是交付代码，哪里是验证代码。

### 3.3 何时不需要复杂结构？

| 场景 | 建议 | 原因 |
|---|---|---|
| 一次性、几十行、只有自己运行 | 单文件脚本即可 | 生命周期很短，工程化收益有限 |
| 会交给同事、需要第三方依赖 | 至少使用 venv、`pyproject.toml`、README | 首要问题是可复现和可交接 |
| 有多模块业务逻辑并持续维护 | 使用 `src/tests` 布局 | 职责边界清楚，便于测试和扩展 |

不要为了“看起来专业”提前创建十几层目录。结构的复杂度应服务于协作、测试和维护，而不是复制大型项目模板。

## 4. 常见坑

| 坑 | 后果 | 根本原因 |
|---|---|---|
| 把依赖装进系统 Python | 项目互相影响，换电脑后无法复现 | 没有隔离依赖作用域 |
| 只发源码，不提供依赖声明 | 同事只能根据报错猜要安装什么 | 环境知识只存在作者脑中 |
| 把 `.venv/` 提交到版本库 | 仓库巨大，并且跨系统通常不可用 | 混淆“依赖声明”和“本机安装产物” |
| 源码、测试、样例全放在根目录 | 文件增多后职责不清，容易误导入 | 没有按用途划分边界 |
| 直接用错误目录下的 `python` | 出现“明明安装了却找不到包” | 没确认当前解释器属于哪个环境 |
| 测试只在 IDE 中运行 | 命令行或 CI 无法复现 | 运行方式依赖个人 IDE 配置 |

遇到导入或依赖问题时，先运行 `python -c "import sys; print(sys.executable)"`，确认正在使用 `.venv` 中的解释器，再检查安装命令。

## 5. 最佳实践

1. **固定创建顺序**：先建目录与 `pyproject.toml`，再创建 `.venv`，最后安装依赖。这样环境由项目声明驱动，而不是靠记忆。
2. **始终记录一条统一入口**：在 README 中写清安装、运行、测试命令；团队成员和 CI 使用同一套命令。
3. **只提交可重建的信息**：提交 `pyproject.toml` 和源码，忽略 `.venv/`、缓存、覆盖率等本机产物。
4. **从小结构开始**：本例只有一个业务包；当职责真的增多时再拆模块，不提前制造空目录。
5. **交接前做“陌生人验证”**：新建一个干净虚拟环境，仅依据 README 安装并运行测试。若成功，才说明项目具有可复现性。

这些实践适合需要复用、协作或持续维护的脚本。真正的一次性排查代码可以更轻量，但只要它开始被第二个人使用，就应考虑升级为项目。

## 6. 练习题

### 练习 1：从零复刻项目骨架（基础）

**对应验收：模块①·环境；Req 1.1**

不要复制示例目录，自己新建 `api-response-checker/`：

- 创建 `.venv`，确认解释器是 Python 3.12+；
- 创建 `pyproject.toml`、`src/api_response_checker/`、`tests/`；
- 写一个返回固定字符串的最小函数和一个 pytest 测试；
- 在 README 中记录安装、运行、测试命令；
- 删除 `.venv` 后按 README 重建，再次让 pytest 通过。

最后用目录树自查：依赖声明、源码目录、测试目录是否都存在？

### 练习 2：模拟同事交接（进阶）

**对应验收：模块①·环境；Req 1.1**

把练习 1 的项目交给同事，或复制到一个新的临时目录。要求不做口头解释，只允许阅读 README。记录对方卡住的步骤，并改进 README 或项目配置，直到对方能运行程序和全部测试。

这个练习验证的不是语法，而是项目能否脱离作者本人继续工作。

## 7. 小结与自检

- 虚拟环境负责隔离“实际安装的依赖”，`pyproject.toml` 负责声明“项目需要的依赖”。
- `src/` 放可交付源码，`tests/` 放验证代码，分离后职责和导入行为更清晰。
- `.venv/` 是可重建的本机产物，应加入 `.gitignore`，不应交付。
- README 中的命令是项目契约的一部分，必须能在干净环境中复现。
- 好结构不是目录越多越好，而是让当前规模下的运行、测试和协作更可靠。

自检问题（可选择跳过）：

1. 为什么只有 `.venv`、没有 `pyproject.toml`，仍然无法让同事复现你的环境？
2. 如果一个脚本从“只运行一次”变成“每周由三位同事运行”，你会先补哪三项工程化内容？

