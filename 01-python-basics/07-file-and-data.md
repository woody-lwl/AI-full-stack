# 1.7 文件与数据处理

> 运行环境：Python 3.12+
>
> 对应模块验收：模块①·文件；Req 1

**Validates: Requirements 1.1, 1.2, 1.3, 1.5**

## 目录
- [1. 场景引入](#1-场景引入)
- [2. 可运行代码示例](#2-可运行代码示例)
- [3. 设计思考](#3-设计思考)
- [4. 常见坑](#4-常见坑)
- [5. 最佳实践](#5-最佳实践)
- [6. 练习题](#6-练习题)
- [7. 小结与自检](#7-小结与自检)

## 1. 场景引入

接口自动化平台导出了一份 CSV 测试结果，产品同事想看通过/失败汇总，流水线则需要一份 JSON 报告。真实数据并不总是干净的：同一个 `case_id` 可能重复，状态可能写成 `success`，耗时可能是 `-1`，文件也可能缺列、为空或不是 UTF-8 编码。

如果把读取文件、字段校验、汇总和写 JSON 全塞进一个循环，代码虽然短，却很难回答这些问题：坏行是跳过还是终止？跳过后去了哪里？不创建真实文件能否测试转换规则？换一个输出位置是否要改核心逻辑？

**一句话直觉：文件是程序与外部世界的边界，数据处理要把“读取/解析、校验转换、输出”分开。**

本章实现一个 `test_result_converter`：读取 UTF-8 CSV，按数据契约清洗测试结果，再输出包含汇总、有效记录和问题清单的 JSON。数据流如下：

```text
results.csv
    │ read_csv_rows：文件读取、编码、CSV 表头
    ▼
list[dict[str, str | None]]
    │ convert_rows：纯数据校验、类型转换、问题收集
    ▼
ConversionResult
    │ build_report：计数、排序、稳定结构
    ▼
dict → write_json_report → report.json
```

先预测示例结果：输入共有 7 行数据，其中一行 ID 重复、一行状态非法、一行耗时为负。最终 `total` 应该是多少？`issues` 又应该有几条？运行后再核对。

## 2. 可运行代码示例

完整工程位于 [`code/07-file-and-data/`](code/07-file-and-data/)，仅使用 Python 标准库处理 CSV 和 JSON。

### 2.1 工程结构与数据契约

```text
07-file-and-data/
├── .gitignore
├── README.md
├── pyproject.toml
├── examples/
│   └── results.csv
├── src/
│   └── test_result_converter/
│       ├── __init__.py
│       ├── __main__.py
│       ├── converter.py
│       ├── data_io.py
│       ├── errors.py
│       ├── models.py
│       ├── report.py
│       └── service.py
└── tests/
    ├── test_converter.py
    ├── test_data_io.py
    ├── test_package.py
    ├── test_report.py
    └── test_service.py
```

输入契约必须先于代码写清楚：

| 列 | 目标类型 | 规则 | 失败策略 |
|---|---|---|---|
| `case_id` | `str` | 去除两端空白、非空、在有效记录中唯一 | 记录 issue，跳过当前行 |
| `name` | `str` | 去除两端空白、非空，允许中文 | 记录 issue，跳过当前行 |
| `status` | `str` | 转小写后只能是 `passed` 或 `failed` | 记录 issue，跳过当前行 |
| `duration_ms` | `int` | 必须能转成非负整数 | 记录 issue，跳过当前行 |

整份文件为空、缺少必要列、编码错误或 CSV 语法错误时，无法可靠解释后续数据，因此终止转换并抛出 `InputFileError`。单行字段错误只影响该行，进入 `issues` 后继续处理。两类策略都不会静默吞数据。

### 2.2 用 `Path`、`with` 和 `DictReader` 读取 CSV

`src/test_result_converter/data_io.py`：

```python
"""CSV 输入与 JSON 输出边界。"""

import csv
import json
from collections.abc import Mapping
from pathlib import Path
from typing import Any

from .errors import InputFileError, OutputFileError

REQUIRED_COLUMNS = ("case_id", "name", "status", "duration_ms")


def read_csv_rows(source: Path) -> list[dict[str, str | None]]:
    """以 UTF-8 读取 CSV，并在返回前验证表头数据契约。"""
    try:
        # newline="" 让 csv 模块统一处理不同平台的换行规则。
        with source.open("r", encoding="utf-8", newline="") as file:
            reader = csv.DictReader(file, strict=True)
            if reader.fieldnames is None:
                raise InputFileError(f"CSV 文件为空: source={source}")

            missing = [column for column in REQUIRED_COLUMNS if column not in reader.fieldnames]
            if missing:
                missing_text = ",".join(missing)
                raise InputFileError(
                    f"CSV 缺少必要列: source={source} missing={missing_text}"
                )

            return list(reader)
    except InputFileError:
        raise
    except FileNotFoundError as exc:
        raise InputFileError(f"CSV 文件不存在: source={source}") from exc
    except UnicodeDecodeError as exc:
        raise InputFileError(f"CSV 不是有效 UTF-8 编码: source={source}") from exc
    except csv.Error as exc:
        raise InputFileError(f"CSV 格式错误: source={source} reason={exc}") from exc
    except OSError as exc:
        raise InputFileError(f"CSV 读取失败: source={source} reason={exc}") from exc


def write_json_report(target: Path, report: Mapping[str, Any]) -> None:
    """创建输出目录，并以稳定、可读的 UTF-8 JSON 写入报告。"""
    try:
        target.parent.mkdir(parents=True, exist_ok=True)
        with target.open("w", encoding="utf-8", newline="\n") as file:
            json.dump(report, file, ensure_ascii=False, indent=2)
            file.write("\n")
    except (OSError, TypeError, ValueError) as exc:
        raise OutputFileError(f"JSON 报告写入失败: target={target}") from exc
```

关键点：

- `pathlib.Path` 把路径拼接、父目录创建、文件打开统一为对象方法，比手工拼接 `/` 更跨平台；
- `with` 在正常完成或发生异常时都会关闭文件，避免文件描述符泄漏；
- CSV 文件应使用 `newline=""` 打开，让 `csv` 模块自己识别 `\n`、`\r\n` 以及引号内换行；写 CSV 时同样如此，否则某些平台可能产生空行；
- 明确 `encoding="utf-8"`，不要依赖不同机器的系统默认编码；
- `DictReader` 用表头作为键，代码表达的是 `row["status"]`，而不是容易写错的 `row[2]`；如果目标仍是 CSV，可以配对使用 `csv.DictWriter`，并通过 `fieldnames` 固定输出列顺序；
- 延续 1.6 的做法，底层异常只在 I/O 边界转成领域异常并用 `from exc` 保留原因；本章重点仍是文件契约与数据流，不重复设计日志层级。

### 2.3 将校验和类型转换写成纯函数

`src/test_result_converter/models.py`：

```python
"""转换过程使用的数据模型。"""

from dataclasses import dataclass


@dataclass(frozen=True, slots=True)
class TestResult:
    """一条通过数据契约校验的测试结果。"""

    case_id: str
    name: str
    status: str
    duration_ms: int


@dataclass(frozen=True, slots=True)
class DataIssue:
    """一条未进入报告记录的数据问题。"""

    row_number: int
    case_id: str | None
    reason: str


@dataclass(frozen=True, slots=True)
class ConversionResult:
    """纯转换阶段产生的有效记录和问题。"""

    records: tuple[TestResult, ...]
    issues: tuple[DataIssue, ...]
```

`src/test_result_converter/converter.py`：

```python
"""不接触文件系统的测试结果校验与转换。"""

from collections.abc import Iterable, Mapping

from .models import ConversionResult, DataIssue, TestResult

ALLOWED_STATUSES = frozenset({"passed", "failed"})


def convert_rows(rows: Iterable[Mapping[str, str | None]]) -> ConversionResult:
    """校验 CSV 行；保留有效记录，并显式收集所有无效行。"""
    records: list[TestResult] = []
    issues: list[DataIssue] = []
    seen_case_ids: set[str] = set()

    # 第 1 行是表头，所以数据行从第 2 行开始计数。
    for row_number, row in enumerate(rows, start=2):
        case_id = (row.get("case_id") or "").strip()
        name = (row.get("name") or "").strip()
        status = (row.get("status") or "").strip().lower()
        duration_text = (row.get("duration_ms") or "").strip()

        reason = _validate_row(
            case_id=case_id,
            name=name,
            status=status,
            duration_text=duration_text,
            seen_case_ids=seen_case_ids,
        )
        if reason is not None:
            issues.append(DataIssue(row_number, case_id or None, reason))
            continue

        duration_ms = int(duration_text)
        records.append(TestResult(case_id, name, status, duration_ms))
        seen_case_ids.add(case_id)

    return ConversionResult(tuple(records), tuple(issues))


def _validate_row(
    *,
    case_id: str,
    name: str,
    status: str,
    duration_text: str,
    seen_case_ids: set[str],
) -> str | None:
    """返回一条行级问题；返回 None 表示该行符合契约。"""
    if not case_id:
        return "case_id 不能为空"
    if case_id in seen_case_ids:
        return "case_id 重复，已保留第一条有效记录"
    if not name:
        return "name 不能为空"
    if status not in ALLOWED_STATUSES:
        return "status 必须是 passed 或 failed"

    try:
        duration_ms = int(duration_text)
    except ValueError:
        return "duration_ms 必须是非负整数"

    if duration_ms < 0:
        return "duration_ms 必须是非负整数"
    return None
```

`convert_rows()` 不打开文件、不写终端，也不依赖当前时间。测试可以直接传入内存中的 `list[dict]`，快速覆盖重复 ID、非法状态和非法耗时。这里采用“每行只记录第一个问题”，让问题数量和跳过行数一一对应；如果业务要求一次报告一行的全部问题，可以把 `_validate_row()` 改为返回 `list[str]`，但要先明确 issue 的统计语义。

### 2.4 构建稳定 JSON 并编排工作流

`src/test_result_converter/report.py`：

```python
"""把转换结果组装成稳定的 JSON 报告结构。"""

from typing import Any

from .models import ConversionResult


def build_report(result: ConversionResult) -> dict[str, Any]:
    """按稳定字段顺序和 case_id 排序生成可序列化报告。"""
    records = sorted(result.records, key=lambda record: record.case_id)
    issues = sorted(result.issues, key=lambda issue: issue.row_number)
    passed = sum(record.status == "passed" for record in records)
    failed = sum(record.status == "failed" for record in records)

    return {
        "total": len(records),
        "passed": passed,
        "failed": failed,
        "records": [
            {
                "case_id": record.case_id,
                "name": record.name,
                "status": record.status,
                "duration_ms": record.duration_ms,
            }
            for record in records
        ],
        "issues": [
            {
                "row_number": issue.row_number,
                "case_id": issue.case_id,
                "reason": issue.reason,
            }
            for issue in issues
        ],
    }
```

`src/test_result_converter/service.py`：

```python
"""编排读取、转换、报告组装与写入。"""

from pathlib import Path
from typing import Any

from .converter import convert_rows
from .data_io import read_csv_rows, write_json_report
from .report import build_report


def convert_file(source: Path, target: Path) -> dict[str, Any]:
    """把一个 CSV 文件转换为 JSON 报告，并返回报告内容。"""
    rows = read_csv_rows(source)
    result = convert_rows(rows)
    report = build_report(result)
    write_json_report(target, report)
    return report
```

稳定输出有意做了四件事：顶层字段顺序固定；有效记录按 `case_id` 排序；问题按原 CSV 行号排序；JSON 使用 `indent=2` 和 `ensure_ascii=False`。这样中文可直接阅读，同一批逻辑数据即使输入顺序改变，报告 diff 也不容易充满无意义变化。

### 2.5 安装、运行与核对

`pyproject.toml` 声明 Python 3.12+、固定构建后端和测试依赖；`output/` 是运行产物，已写入 `.gitignore`。

```bash
cd codelearn/01-python-basics/code/07-file-and-data
/opt/homebrew/bin/python3.12 -m venv .venv
.venv/bin/python -m pip install "setuptools==75.8.0"
.venv/bin/python -m pip install --no-build-isolation -e ".[test]"
.venv/bin/python -m test_result_converter
.venv/bin/python -m pytest --color=no
```

CLI 的稳定汇总部分应为：

```text
total=4 passed=2 failed=2 issues=3
```

生成的 `output/report.json` 为：

```json
{
  "total": 4,
  "passed": 2,
  "failed": 2,
  "records": [
    {
      "case_id": "login-001",
      "name": "登录成功",
      "status": "passed",
      "duration_ms": 120
    },
    {
      "case_id": "order-002",
      "name": "创建订单失败",
      "status": "failed",
      "duration_ms": 340
    },
    {
      "case_id": "refund-003",
      "name": "中文退款",
      "status": "passed",
      "duration_ms": 88
    },
    {
      "case_id": "search-006",
      "name": "搜索超时",
      "status": "failed",
      "duration_ms": 1500
    }
  ],
  "issues": [
    {
      "row_number": 5,
      "case_id": "order-002",
      "reason": "case_id 重复，已保留第一条有效记录"
    },
    {
      "row_number": 6,
      "case_id": "profile-004",
      "reason": "status 必须是 passed 或 failed"
    },
    {
      "row_number": 7,
      "case_id": "payment-005",
      "reason": "duration_ms 必须是非负整数"
    }
  ]
}
```

pytest 覆盖正常转换、中文 UTF-8、空文件、缺列、错误编码、CSV 格式错误、重复 ID、非法状态、空/文本/负数耗时、输出目录创建、JSON 回读和命令入口。文件测试全部使用 `tmp_path`，每个测试都有独立临时目录。

## 3. 设计思考

### 3.1 为什么按“读取—转换—输出”拆分？

三个阶段由不同原因变化：CSV 可能换成接口或数据库输入，字段契约可能增加 `environment`，JSON 报告可能换成 HTML。把变化原因分开后：

- `data_io.py` 只关心外部格式、编码、路径和资源关闭；
- `converter.py` 只关心一行数据能否成为 `TestResult`；
- `report.py` 只关心汇总结构和稳定排序；
- `service.py` 只编排数据流，review 时从五行代码就能看懂主流程。

这对应 Req 1.2 与 1.5：公开函数都有类型注解，模块职责和数据流不需要额外口头解释。项目保留标准 `src/`、`tests/` 和依赖声明，对应 Req 1.1；核心行为由 pytest 一次运行验证，对应 Req 1.3。

### 3.2 本地文件依赖如何隔离？

文件系统是外部依赖，但不代表所有测试都要 mock `Path.open()`：

1. 纯规则测试直接调用 `convert_rows()`，只传内存字典，完全不访问磁盘；
2. 真正需要验证编码、换行、目录创建和 JSON 回读时，使用 pytest 的 `tmp_path` 注入输入/输出路径；
3. `convert_file(source, target)` 不把路径写死，调用方可以选择示例目录、临时目录或未来的工作目录。

`tmp_path` 是真实文件系统上的临时目录，比 mock 更适合验证标准库 I/O 行为；测试结束后由 pytest 清理，测试之间不共享文件，所以不依赖执行顺序。这是 Req 1.4 在本地文件场景中的实现与测试证据：通过参数注入和临时目录隔离外部副作用，而不是让核心逻辑绑定默认路径。

### 3.3 何时用 CSV，何时用 JSON？

| 格式 | 优点 | 局限 | 适用场景 |
|---|---|---|---|
| CSV | 表格紧凑，Excel 易打开，人工批量编辑方便 | 类型全靠约定，不擅长嵌套；不同工具对编码、分隔符处理可能不同 | 固定列的用例、测试结果批量导入导出 |
| JSON | 能表达数组、对象、数字和嵌套结构；API 与程序处理方便 | 人工编辑大量表格不方便，文件通常更大 | API 数据、带汇总和问题明细的结构化报告 |

本例输入是固定四列，CSV 对测试同事友好；输出同时包含汇总、记录和问题数组，JSON 更自然。不要按“哪个更高级”选择，而要看数据形状和使用者。

### 3.4 行级错误应该跳过还是终止？

- **严格失败**：任意坏行都终止。优点是不会产出不完整报告；适合计费、迁移等要求全有或全无的任务。
- **记录后继续**：有效数据仍产出，坏行进入 `issues`。适合批量测试结果清洗，但调用方必须看问题数量。
- **静默跳过**：实现最省事，却让输入 100 行、输出 80 行时无法解释丢失的 20 行，本章禁止。

本例选择第二种，并定义 `total` 是“进入报告的有效记录数”，`len(issues)` 是“被拒绝的行数”。数据指标的口径必须写清楚，否则汇总数字即使计算正确也可能被误读。

### 3.5 什么时候需要重构为流式处理？

`read_csv_rows()` 当前把文件读成列表，结构直观，适合教学和中小型测试报告。若输入增长到数百万行，内存占用会成为问题，可将 reader 作为迭代器逐行转换，并考虑边处理边写临时文件。但流式写 JSON 数组会增加失败回滚和输出完整性的复杂度；先测量文件规模，再决定是否优化，不为假想性能提前牺牲可读性。

## 4. 常见坑

| 坑 | 后果 | 根本原因 |
|---|---|---|
| 不写 `encoding` | 同一文件在不同机器表现不同，中文可能报错或乱码 | 依赖操作系统默认编码 |
| CSV 用普通文本切 `line.split(",")` | 带逗号、引号或换行的字段被拆错 | 把 CSV 当成简单分隔文本 |
| 打开 CSV 时不设 `newline=""` | 跨平台读取异常或写出额外空行 | 文件层和 csv 模块同时处理换行 |
| 用列位置 `row[2]` | 调整表头顺序后读错字段 | 没使用表头表达数据契约 |
| 缺列后继续运行 | 大量行都变成空值问题，掩盖文件级错误 | 没区分结构错误与行级错误 |
| `except Exception: continue` | 数据被静默丢弃，无法追踪输入输出差异 | 没定义错误记录策略 |
| `int(value)` 前不处理空字符串 | 空值抛 `ValueError`，整批意外终止 | 类型转换前未执行缺失值策略 |
| 重复 ID 直接覆盖字典 | 先出现的数据悄悄消失 | 去重策略未明确 |
| JSON 默认 `ensure_ascii=True` | 中文变成 `\u` 转义，可读性与 review 体验差 | 采用默认序列化选项未考虑读者 |
| 报告记录顺序不稳定 | 每次运行产生大量无意义 diff | 没定义稳定排序规则 |
| 测试写固定 `output/report.json` | 用例互相污染，顺序变化时偶发失败 | 文件路径不可注入，副作用未隔离 |
| 一律用 mock 测文件 | 编码、换行、真实 JSON 可回读等问题漏测 | mock 了本来应该验证的标准库边界 |

## 5. 最佳实践

1. **先写数据契约再写循环**：列出必要列、目标类型、规范化规则、缺失值与错误策略。
2. **路径统一使用 `Path`**：用 `/` 拼子路径、`parent.mkdir()` 创建目录，避免手工拼路径分隔符。
3. **文本编码显式使用 UTF-8**：读写双方保持一致；编码不符时清晰失败，不使用 `errors="ignore"` 丢字符。
4. **文件资源交给 `with` 管理**：无论转换成功还是异常退出，都及时关闭文件。
5. **CSV 交给 `csv` 模块解析**：读时用 `DictReader`，写表格时用 `DictWriter`，并配合 `newline=""`。
6. **文件级错误与行级错误分开**：缺表头等结构问题终止；可恢复行问题进入明确的 `issues`。
7. **纯转换不接触文件系统**：让同一规则可以处理 CSV 行、API 数据或测试内存数据。
8. **稳定化机器产物**：固定字段顺序、记录排序和缩进；中文报告使用 `ensure_ascii=False`。
9. **对文件边界做真实小测试**：使用 `tmp_path` 验证目录创建、UTF-8、JSON 回读和异常链；不要依赖默认输出目录。
10. **让 README 记录可复现命令**：他人无需知道你的 IDE 配置，也能安装、运行和测试。

这些做法适用于测试结果转换、批量用例导入和 AI 文档预处理。一次性小脚本可以减少模块数量，但仍应保留明确编码、数据契约和非静默错误处理。

## 6. 练习题

### 练习 1：增加环境字段（基础）

**对应验收：模块①·文件；Req 1.2、1.3**

给 CSV 增加必要列 `environment`，只允许 `test`、`staging`、`prod`。让 JSON 记录保留该字段，并补充缺列、非法值和中文用例名测试。

提示：先修改契约与模型，再修改转换和报告；不要只在 CLI 中临时判断。

### 练习 2：输出慢用例排行（进阶）

**对应验收：模块①·文件；Req 1.3、1.5**

在报告中增加 `slowest_records`，按 `duration_ms` 从大到小取前 3 条；耗时相同时用 `case_id` 保证顺序稳定。写测试验证少于 3 条、恰好 3 条和耗时相同三种情况。

思考：排行属于输入校验、报告组装还是文件输出职责？为什么？

### 练习 3：导出问题 CSV（挑战）

**对应验收：模块①·文件；Req 1.1、1.4**

除 JSON 报告外，再用 `csv.DictWriter` 把 `issues` 输出为 `output/issues.csv`。固定列为 `row_number,case_id,reason`，用 `tmp_path` 注入目标位置，验证中文、换行和无问题时仅有表头。

提示：写 CSV 时同样使用 `encoding="utf-8"`、`newline=""` 和 `with`；不要让纯转换函数知道第二个输出文件。

## 7. 小结与自检

- 文件是外部边界；读取/解析、纯校验转换、报告组装和输出应按职责分开。
- `Path` 管理路径，`with` 管理资源，UTF-8 管理文本编码，`newline=""` 把 CSV 换行规则交给标准库。
- 数据契约要明确必要列、类型、缺失值和失败策略；坏数据要么清晰终止，要么进入 `issues`，不能静默消失。
- CSV 适合固定表格的人工交换，JSON 适合嵌套报告和程序接口；格式选择由数据形状和使用者决定。
- 固定字段顺序、排序、缩进和 `ensure_ascii=False` 能让报告可读、可回归、易 review。
- 纯函数用内存数据测试，真实文件边界用注入的 `tmp_path` 测试，在隔离副作用的同时验证真正的编码和序列化行为。

自检问题（可选择跳过）：

1. 为什么缺少 `duration_ms` 列要终止整次转换，而某一行的 `duration_ms=-1` 可以记录 issue 后继续？
2. 如果测试直接写项目的 `output/report.json`，它为什么可能单独运行通过、整套测试却偶发失败？`tmp_path` 解决的根本问题是什么？
