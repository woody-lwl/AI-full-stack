# 1.8 标准库

> 运行环境：Python 3.12+
>
> 对应模块验收：模块①·标准库

**Validates: 模块①·标准库**

## 目录
- [1. 场景引入](#1-场景引入)
- [2. 可运行代码示例](#2-可运行代码示例)
- [3. 设计思考](#3-设计思考)
- [4. 常见坑](#4-常见坑)
- [5. 最佳实践](#5-最佳实践)
- [6. 练习题](#6-练习题)
- [7. 小结与自检](#7-小结与自检)

## 1. 场景引入

每天凌晨，接口自动化平台会把不同执行节点的结果写成多份 JSON。值班同事只关心最近 24 小时：通过和失败各有多少、每个 suite 有哪些用例、失败用例如何每 2 条组成一批送去复测。

这看似只是几个循环，但很容易重复造轮子：手工拼路径、切割时间字符串、维护状态计数字典、判断分组键是否存在、用切片复制大列表。Python 已经为这些常见问题准备了经过广泛使用的工具。

**一句话直觉：标准库是随 Python 提供、经过广泛验证的通用工具箱；先识别问题类别，再寻找合适工具，而不是先手写。**

本章把任务拆成一条连贯数据流：

```text
Path 扫描 JSON
    → chain 惰性展平多份报告
    → datetime 解析带时区时间并筛选窗口
    → Counter 统计状态
    → defaultdict 构建 suite 分组
    → batched 惰性生成失败复测批次
```

先预测：示例中共有 7 条记录，其中一条早于窗口，一条晚于基准时间。窗口内应剩几条？失败用例按时间排序、每批 2 条时会形成几批？

## 2. 可运行代码示例

完整工程位于 [`code/08-standard-library/`](code/08-standard-library/)，业务代码没有第三方运行时依赖。

### 2.1 工程结构与示例契约

```text
08-standard-library/
├── .gitignore
├── README.md
├── pyproject.toml
├── examples/
│   ├── report-a.json
│   └── report-b.json
├── src/test_report_analyzer/
│   ├── __init__.py
│   ├── __main__.py
│   ├── analyzer.py
│   ├── config.py
│   ├── data_io.py
│   ├── errors.py
│   ├── models.py
│   └── service.py
└── tests/
    ├── test_analyzer.py
    ├── test_config.py
    ├── test_data_io.py
    └── test_main.py
```

每条 JSON 记录包含 `case_id`、`suite`、`status` 和 `finished_at`。时间必须是带偏移量的 ISO 8601，例如 `2026-09-29T11:30:00+08:00` 或 `2026-09-29T03:30:00Z`；状态只允许 `passed`、`failed`。

### 2.2 `Path` 扫描，`chain` 展平

`data_io.py` 中的核心边界如下：

```python
import json
from collections.abc import Iterable, Iterator
from itertools import chain
from pathlib import Path

from .analyzer import parse_aware_datetime
from .models import ReportRecord


def discover_report_files(report_dir: Path) -> list[Path]:
    """只扫描目录直属 JSON，并用排序保证结果稳定。"""
    if not report_dir.is_dir():
        raise ValueError(f"报告目录不存在或不是目录: {report_dir}")
    return sorted(report_dir.glob("*.json"))


def read_report(path: Path) -> tuple[ReportRecord, ...]:
    """把一份 JSON 报告转换为记录。"""
    raw = json.loads(path.read_text(encoding="utf-8"))
    return tuple(
        ReportRecord(
            case_id=item["case_id"],
            suite=item["suite"],
            status=item["status"],
            finished_at=parse_aware_datetime(item["finished_at"]),
        )
        for item in raw
    )


def iter_report_records(paths: Iterable[Path]) -> Iterator[ReportRecord]:
    """需要记录时才依次读取报告，不先拼一个大列表。"""
    return chain.from_iterable(read_report(path) for path in paths)
```

工程中的正式 `read_report()` 还包含 JSON 结构、缺失字段和非空字符串校验，这里展示的是标准库协作主线；可直接运行的完整实现已在源码中给出，没有省略占位代码。

上一章已讲过 UTF-8 与 JSON 读写，本章只关注工具选择：`Path.glob()` 表达“扫描路径”，`sorted()` 消除文件系统顺序的不确定性；`chain.from_iterable()` 表达“把多组记录视为一条记录流”，避免手写双层循环加 `append`。

### 2.3 `datetime` 建立可比较的时间线

`analyzer.py`：

```python
from collections import Counter, defaultdict
from collections.abc import Iterable, Iterator
from datetime import datetime, timedelta
from itertools import batched

from .models import AnalysisResult, ReportRecord


def parse_aware_datetime(value: str) -> datetime:
    """解析 ISO 8601，并拒绝不知道时区的时间。"""
    parsed = datetime.fromisoformat(value)
    if parsed.tzinfo is None or parsed.utcoffset() is None:
        raise ValueError(f"时间必须包含时区偏移: {value!r}")
    return parsed


def analyze_records(
    records: Iterable[ReportRecord],
    *,
    reference_time: datetime,
    lookback_hours: int,
) -> AnalysisResult:
    """分析闭区间 [基准时间-回看时长, 基准时间]。"""
    if reference_time.tzinfo is None or reference_time.utcoffset() is None:
        raise ValueError("reference_time 必须包含时区")
    if lookback_hours <= 0:
        raise ValueError("lookback_hours 必须是正整数")

    window_start = reference_time - timedelta(hours=lookback_hours)
    status_counts: Counter[str] = Counter()
    grouped: defaultdict[str, list[ReportRecord]] = defaultdict(list)
    retries: list[ReportRecord] = []

    for record in records:
        # aware datetime 即使偏移量不同，也能按同一时间线正确比较。
        if window_start <= record.finished_at <= reference_time:
            status_counts[record.status] += 1
            grouped[record.suite].append(record)
            if record.status == "failed":
                retries.append(record)

    # 构建完成后转为普通 dict，读不存在的 suite 不会偷偷创建键。
    by_suite = {
        suite: tuple(sorted(items, key=lambda item: (item.finished_at, item.case_id)))
        for suite, items in sorted(grouped.items())
    }
    retry_candidates = tuple(
        sorted(retries, key=lambda item: (item.finished_at, item.case_id))
    )
    return AnalysisResult(
        reference_time=reference_time,
        window_start=window_start,
        status_counts=status_counts,
        by_suite=by_suite,
        retry_candidates=retry_candidates,
    )


def iter_retry_batches(
    records: Iterable[ReportRecord], batch_size: int
) -> Iterator[tuple[ReportRecord, ...]]:
    """Python 3.12 的 batched 按需产生 tuple 批次。"""
    if batch_size <= 0:
        raise ValueError("batch_size 必须是正整数")
    failed = (record for record in records if record.status == "failed")
    return batched(failed, batch_size)
```

这里没有直接调用 `datetime.now()`，而是注入 `reference_time`。默认 CLI 使用固定的 `2026-09-29T12:00:00+08:00`，因此教程明天运行也有相同结果；真实系统可在入口注入当前 aware 时间。

窗口定义为闭区间：正好位于开始和结束时刻的记录都计入，结束时间之后的“未来记录”不计入。这个边界必须由业务明确，不能让实现者凭感觉决定。

### 2.4 `Counter` 计数，`defaultdict` 分组

手写计数通常是：

```python
if status not in counts:
    counts[status] = 0
counts[status] += 1
```

`Counter` 把意图直接写成 `counts[status] += 1`，且读取缺失键返回 `0`。因此无失败记录时，CLI 仍可安全输出 `counts["failed"]`。

分组时，`defaultdict(list)` 让 `grouped[suite].append(record)` 不需要每次先判断 suite 是否出现。不过它有一个重要副作用：读取缺失键也会创建空列表。所以本例只在函数内部构建时使用，返回前转为普通 `dict`。这是“使用方便容器”与“控制公共行为”之间的边界。

### 2.5 从 `os.environ` 读取字符串配置

`config.py` 的关键部分：

```python
import os
from collections.abc import Mapping
from dataclasses import dataclass
from datetime import datetime

from .analyzer import parse_aware_datetime


@dataclass(frozen=True, slots=True)
class AnalyzerConfig:
    lookback_hours: int
    batch_size: int
    reference_time: datetime


def load_config(environ: Mapping[str, str] | None = None) -> AnalyzerConfig:
    values = os.environ if environ is None else environ
    lookback_hours = _positive_int(values, "LOOKBACK_HOURS", default=24)
    batch_size = _positive_int(values, "BATCH_SIZE", default=2)
    reference_time = parse_aware_datetime(
        values.get("REFERENCE_TIME", "2026-09-29T12:00:00+08:00")
    )
    return AnalyzerConfig(lookback_hours, batch_size, reference_time)


def _positive_int(values: Mapping[str, str], name: str, *, default: int) -> int:
    text = values.get(name, str(default))
    value = int(text)  # 环境变量永远先以字符串进入程序。
    if value <= 0:
        raise ValueError(f"{name} 必须是正整数")
    return value
```

正式源码会把转换错误包装为 `ConfigurationError`，使 CLI 输出清晰错误。注入一个映射后，测试可以传 `{"BATCH_SIZE": "3"}`，不用修改全局进程环境。

这里只用 `os.environ` 处理环境变量，路径操作全部交给 `Path`；不要在同一项目中一半 `os.path.join()`、一半 `Path / child`，否则类型和风格会反复转换。

### 2.6 安装、运行与核对

```bash
cd codelearn/01-python-basics/code/08-standard-library
/opt/homebrew/bin/python3.12 -m venv .venv
.venv/bin/python -m pip install "setuptools==75.8.0"
.venv/bin/python -m pip install --no-build-isolation -e ".[test]"
.venv/bin/python -m test_report_analyzer
.venv/bin/python -m pytest --color=no
```

默认输出应为：

```text
reference=2026-09-29T12:00:00+08:00 window_start=2026-09-28T12:00:00+08:00
files=2 records=5
status: passed=2 failed=3
suite 冒烟: count=2 cases=order-003,login-001
suite 回归: count=1 cases=profile-004
suite 支付: count=2 cases=refund-005,payment-002
retry batch 1: refund-005,payment-002
retry batch 2: order-003
```

两个 `Z` 时间会按 UTC 瞬间与 `+08:00` 基准正确比较。较早的 `legacy-999` 和晚于基准的 `future-006` 被排除；最后一个复测批次不足 2 条时仍会产出。

## 3. 设计思考

### 3.1 先识别问题类别，而不是先写循环

看到需求时可以先做这张“问题—工具”映射：

| 问题类别 | 标准库工具 | 它替代的常见手写逻辑 |
|---|---|---|
| 文件路径与扫描 | `pathlib.Path` | 字符串拼分隔符、后缀判断 |
| 时间解析与窗口 | `datetime` / `timedelta` | 切字符串、手算秒数 |
| 频次统计 | `Counter` | 初始化键再递增 |
| 一对多分组 | `defaultdict(list)` | `if key not in dict` |
| 多序列展平 | `chain.from_iterable` | 双层循环复制列表 |
| 固定大小分批 | Python 3.12 `batched` | 下标切片和尾批判断 |
| 运行配置 | `os.environ` | 把环境差异写死在源码 |

这不是为了“代码越短越好”，而是让代码使用团队熟悉的词汇表达意图。标准库实现经过大量真实场景验证，边界行为有文档，也通常由 C 或经过优化的 Python 实现支撑。

### 3.2 标准库与手写方案的取舍

| 维度 | 标准库组合 | 手写循环/容器 |
|---|---|---|
| 可靠性 | 常见边界经过广泛使用与维护 | 每个项目重新承担缺陷风险 |
| 可读性 | `Counter`、`batched` 直接表达领域动作 | 读者要从下标和条件推断意图 |
| 边界行为 | 有明确约定，如尾批仍返回 | 容易遗漏空输入、尾批、缺失键 |
| 性能 | `chain`、`batched` 支持惰性消费 | 列表拼接可能复制全部数据 |
| 灵活性 | 适合通用问题 | 极特殊规则可完全定制 |

惰性不等于永远更快：如果后续需要多次遍历、排序或随机访问，最终仍需物化为 list/tuple。本例读取阶段可惰性展平，但分组和按时间排序必须保存窗口内记录，这是需求本身决定的成本。

### 3.3 为什么边界与纯分析分开？

`data_io.py` 负责路径和 JSON，`config.py` 负责字符串配置，`analyzer.py` 只接收记录和明确参数。这样可以：

- 用内存记录精确测试时区和窗口边界；
- 用 `tmp_path` 单独验证目录扫描；
- 给 `load_config()` 注入字典，不污染测试进程环境；
- 同一分析逻辑未来可接数据库或 API，而不重写计数、分组和批处理。

所有计数器、分组容器都在每次调用内部创建，因此不同分析实例不会串状态。

### 3.4 何时标准库不够？

标准库优先不等于拒绝第三方库。出现以下需求时应评估成熟依赖：

- 数百万行表格的列式计算、连接和缺失值分析，可评估 Polars/Pandas；
- 复杂自然语言时区名称、交易日历或特殊时间规则，可评估专用日期库；
- 需要 JSON Schema 级别的数据契约与丰富错误位置，可评估验证框架；
- 任务要分布式执行、持久化重试状态，仅靠 `batched` 已不能满足可靠性。

引入前比较维护活跃度、许可证、安全记录、包体积和团队学习成本。不要因为标准库代码多两行就引依赖，也不要为了“零依赖”手写一个脆弱的日期解析器。

## 4. 常见坑

| 坑 | 后果 | 根本原因 |
|---|---|---|
| 混用 naive 与 aware `datetime` | 比较时报错，或错误理解同一时刻 | 时间缺少时区语义 |
| 只比较时间字符串 | 不同偏移量下字典序不代表先后 | 把表示文本当成时间线 |
| 直接依赖 `datetime.now()` | 示例和测试随运行时刻漂移 | 时钟没有作为输入边界 |
| 认为 Counter 缺失键会报错 | 写多余初始化分支 | 不熟悉 `Counter` 的零值语义 |
| 用 `counter[missing]` 判断键是否存在 | 得到 0，但无法区分未出现和出现 0 次 | 数值查询与成员查询含义不同；成员判断应使用 `in` |
| 读取 `defaultdict[unknown]` | 查询动作意外新增空分组 | 默认工厂在 `__getitem__` 时执行 |
| 重复遍历同一个 `chain`/`batched` | 第二次为空 | 迭代器一次消费，不是可重复容器 |
| 为了打印先 `list(chain(...))` | 大报告一次占满内存 | 过早物化破坏惰性 |
| 忘记环境变量都是字符串 | `"2" + 1` 报错，`bool("false")` 仍为真 | 未在配置边界显式解析 |
| 不校验批次大小 | 0 或负数触发异常或无意义行为 | 外部配置未建立合法范围 |
| 同时用 `os.path` 和 `Path` 拼路径 | 代码反复转换字符串与 Path | 同一职责选了两套工具 |
| 把标准库当成万能方案 | 复杂任务代码膨胀、性能不足 | 没有评估问题规模和专业库收益 |

## 5. 最佳实践

1. **需求先分类，再查标准库**：路径、时间、计数、分组、迭代和配置分别寻找领域工具，而不是立即写辅助函数。
2. **整条时间链都使用 aware datetime**：输入必须含偏移量，基准时间同样含时区；明确窗口是否包含边界与未来数据。
3. **让时钟可注入**：教程用固定基准保证复现；生产入口可传 `datetime.now(timezone.utc)`，纯分析函数不自己取时间。
4. **容器按阶段使用**：构建阶段用 `defaultdict`，公共结果转普通 `dict`；计数用 `Counter`，成员存在性仍用 `in`。
5. **标记惰性返回值**：函数名使用 `iter_`，返回类型写 `Iterator`；需要复用时由调用方显式转为 tuple/list。
6. **只物化需求必须的数据**：排序、分组需要保存数据；单次扫描和批处理保持惰性，避免无意义复制。
7. **配置在入口解析并校验**：环境变量转成正整数和 aware datetime 后，业务层只接收可靠类型。
8. **同一职责只选一套抽象**：路径统一 `Path`；`os` 在本章只负责 `environ`。
9. **通过边界测试学习工具语义**：覆盖窗口端点、不同偏移量、空输入、缺失 Counter 键、短尾批和迭代器一次消费。
10. **第三方依赖要有收益证据**：当数据规模、契约复杂度或可靠性要求超过标准库适用范围，再进行依赖评估。

## 6. 练习题

### 练习 1：把手写分批替换为 `batched`（基础）

**对应验收：模块①·标准库**

下面逻辑会先复制整个列表：

```python
def manual_batches(items, size):
    result = []
    for start in range(0, len(items), size):
        result.append(items[start : start + size])
    return result
```

改为 Python 3.12 的 `itertools.batched`，并测试空输入、刚好整除和不足一批。说明新方案返回 tuple 迭代器后，调用方式为何需要变化。

### 练习 2：增加失败 suite 排名（进阶）

**对应验收：模块①·标准库**

使用 `Counter` 统计窗口内每个 suite 的失败数，通过 `most_common()` 输出排名。先定义数量相同时是否需要稳定的 suite 名排序，再写测试锁定规则。

提示：标准库提供了主要能力，但业务相同值的排序规则仍要由你决定。

### 练习 3：用 `islice` 限制预览（挑战）

**对应验收：模块①·标准库**

CLI 增加 `PREVIEW_BATCHES` 环境变量，只预览前 N 个复测批次。使用 `itertools.islice()`，并用带消费记录的生成器证明后续批次没有被提前读取。

思考：预览后还能从头再次遍历原迭代器吗？如果业务既要预览又要完整提交，应怎样安排数据流？

## 7. 小结与自检

- 标准库优先的核心不是少写代码，而是先识别通用问题，复用可靠、可读且有明确边界的工具。
- `Path` 表达路径，aware `datetime` 表达同一时间线，`Counter` 表达计数，`defaultdict` 表达构建期分组。
- `chain` 和 `batched` 支持惰性处理，但迭代器只能消费一次；排序或重复访问时应有意识地物化。
- 环境变量始终是字符串，必须在配置边界解析、校验，再把可靠类型交给业务逻辑。
- 标准库适合通用中小规模任务；当复杂度、数据规模或可靠性要求明显超出能力范围时，应评估成熟第三方库，而不是继续造轮子。

自检问题（可选择跳过）：

1. 为什么本例内部使用 `defaultdict(list)`，返回结果时却转换成普通 `dict`？
2. `chain.from_iterable()` 返回的对象为什么不能先用于统计总数，再从头用于分析？如果确实需要两次遍历，你会如何取舍内存与重新读取成本？
