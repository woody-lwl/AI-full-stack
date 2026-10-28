# 1.2 数据类型与控制流

> 运行环境：Python 3.12+
>
> 对应模块验收：模块①·数据类型

**Validates: Requirements 1**

## 目录
- [1. 场景引入](#1-场景引入)
- [2. 可运行代码示例](#2-可运行代码示例)
- [3. 设计思考](#3-设计思考)
- [4. 常见坑](#4-常见坑)
- [5. 最佳实践](#5-最佳实践)
- [6. 练习题](#6-练习题)
- [7. 小结与自检](#7-小结与自检)

## 1. 场景引入

接口自动化执行前，你收到了一批从表格、测试平台或同事脚本汇总来的测试用例。它们看起来都是“几行数据”，但真实情况往往是：必填字段为空、同一个 `case_id` 出现两次、状态码被写成字符串 `"200"`，甚至某一行根本不是对象。

如果不先清洗，这些问题会在测试执行阶段变成误报：重复用例让统计失真，错误状态码让断言含义不清，空 ID 又使失败记录无法追踪。

**一句话直觉：数据结构负责把信息放对位置，控制流负责让每条数据按规则走向“保留”或“拒绝”。**

本章完成一个真实的批量清洗流程：输入一组接口测试用例，输出“可执行用例”和“问题清单”。你会看到：

- `list` 保存有顺序的一批记录或错误；
- `dict` 表示一条拥有命名字段的测试用例；
- `set` 快速判断 `case_id` 是否出现过；
- `if` 处理不同数据分支，`for` 逐条处理；
- 推导式用于“从已有集合转换出新集合”，而不是代替所有循环。

## 2. 可运行代码示例

完整工程位于 [`code/02-data-types-and-control-flow/`](code/02-data-types-and-control-flow/)。它与上一章的 [`code/01-env-and-structure/`](code/01-env-and-structure/) 相互独立，不会修改上一章项目；两个章节目录都按教程文件 stem 命名，而本章内部发行项目名仍是 `test-data-cleaner`、Python 包名仍是 `test_data_cleaner`。

请先预测：下面 5 条原始数据最终会保留几条？会产生几个问题？再运行程序核对。

### 2.1 数据结构怎么选

| 数据 | 选择 | 为什么合适 |
|---|---|---|
| 一批原始用例 | `list` | 需要保留输入顺序，错误信息才能对应原始行号 |
| 单条用例 | `dict` | `case_id`、`name` 等字段有明确名称，按键读取比按下标直观 |
| 已见过的 ID | `set` | 只关心“是否存在”，平均情况下成员判断很快，并天然不保存重复值 |
| 清洗结果、问题清单 | `list` | 两者都需要按发现顺序追加并输出 |

### 2.2 核心清洗代码

`src/test_data_cleaner/cleaner.py`：

```python
"""批量清洗接口测试用例数据。"""


def clean_test_cases(
    raw_cases: list[object],
) -> tuple[list[dict[str, object]], list[str]]:
    """清洗测试用例，返回可执行用例和可读的问题清单。"""
    cleaned_cases: list[dict[str, object]] = []
    issues: list[str] = []
    seen_case_ids: set[str] = set()

    # enumerate 同时给出行号和记录，便于定位原始脏数据。
    for row_number, raw_case in enumerate(raw_cases, start=1):
        if not isinstance(raw_case, dict):
            issues.append(f"第 {row_number} 条：记录必须是字典")
            continue

        case_id = raw_case.get("case_id")
        name = raw_case.get("name")
        expected_status = raw_case.get("expected_status")
        raw_tags = raw_case.get("tags", [])

        # 一次收集当前记录的全部问题，而不是发现第一个就退出。
        row_issues: list[str] = []
        if not isinstance(case_id, str) or not case_id.strip():
            row_issues.append("case_id 必须是非空字符串")
        if not isinstance(name, str) or not name.strip():
            row_issues.append("name 必须是非空字符串")
        if (
            isinstance(expected_status, bool)
            or not isinstance(expected_status, int)
            or not 100 <= expected_status <= 599
        ):
            row_issues.append("expected_status 必须是 100~599 的整数")
        if not isinstance(raw_tags, list) or not all(
            isinstance(tag, str) for tag in raw_tags
        ):
            row_issues.append("tags 必须是字符串列表")

        if row_issues:
            # 列表推导式：把每条局部错误转换成带行号的错误。
            issues.extend(
                [f"第 {row_number} 条：{message}" for message in row_issues]
            )
            continue

        normalized_case_id = case_id.strip()
        if normalized_case_id in seen_case_ids:
            issues.append(f"第 {row_number} 条：case_id 重复：{normalized_case_id}")
            continue

        # 集合推导式去重；sorted 固定顺序，方便断言和代码 review。
        normalized_tags = sorted(
            {tag.strip().lower() for tag in raw_tags if tag.strip()}
        )
        cleaned_case: dict[str, object] = {
            "case_id": normalized_case_id,
            "name": name.strip(),
            "expected_status": expected_status,
            "tags": normalized_tags,
        }

        seen_case_ids.add(normalized_case_id)
        cleaned_cases.append(cleaned_case)

    return cleaned_cases, issues
```

注意两个 `continue` 的含义：当前记录不合格时，立即跳到下一轮循环。这样后面的“去重、规范化、加入结果”只会处理已经通过格式检查的数据，主流程比多层嵌套的 `if` 更容易阅读。

### 2.3 准备一批真实脏数据

`src/test_data_cleaner/__main__.py`：

```python
"""运行一组包含典型脏数据的接口测试用例。"""

from test_data_cleaner.cleaner import clean_test_cases


def main() -> None:
    raw_cases: list[object] = [
        {
            "case_id": " login-001 ",
            "name": " 用户登录成功 ",
            "expected_status": 200,
            "tags": ["Smoke", "api", "smoke"],
        },
        {"case_id": "", "name": "缺少用例 ID", "expected_status": 400, "tags": []},
        {"case_id": "login-001", "name": "重复用例", "expected_status": 401, "tags": ["security"]},
        {"case_id": "order-001", "name": "状态码格式错误", "expected_status": "201", "tags": ["api"]},
        "这不是字典记录",
    ]

    cleaned_cases, issues = clean_test_cases(raw_cases)

    print("可执行用例：")
    for case in cleaned_cases:
        print(f"- {case['case_id']} | {case['name']} | {case['expected_status']}")

    print("\n发现的问题：")
    for issue in issues:
        print(f"- {issue}")


if __name__ == "__main__":
    main()
```

为了让示例代码完整可运行，项目已用 `pyproject.toml` 固定构建和测试依赖：`setuptools==75.8.0`、`pytest==8.4.2`、`hypothesis==6.112.2`。

### 2.4 创建隔离环境并运行

```bash
cd codelearn/01-python-basics/code/02-data-types-and-control-flow
/opt/homebrew/bin/python3.12 -m venv .venv
.venv/bin/python -m pip install -e ".[test]"
.venv/bin/python -m test_data_cleaner
```

预期输出：

```text
可执行用例：
- login-001 | 用户登录成功 | 200

发现的问题：
- 第 2 条：case_id 必须是非空字符串
- 第 3 条：case_id 重复：login-001
- 第 4 条：expected_status 必须是 100~599 的整数
- 第 5 条：记录必须是字典
```

结果保留 1 条合法用例，并分别识别出空值、重复 ID、状态码类型错误和非字典格式。`"201"` 没有自动转成 `201`，因为静默转换可能掩盖上游数据质量问题；清洗策略必须与团队约定一致。

### 2.5 用 pytest 验证边界

```bash
.venv/bin/python -m pytest
```

测试覆盖：正常数据规范化、空字段、重复 `case_id`、`True`/`None`/字符串/越界状态码、非字典记录、错误标签格式，以及随机输入下“输出 ID 始终唯一、状态码始终有效”的属性。

## 3. 设计思考

### 3.1 为什么去重使用 set，而不是另一个 list？

用 `list` 也能写 `if case_id in seen_ids`，而且小数据量下通常看不出差异。但列表成员判断需要从头逐个比较；集合为“成员是否存在”而设计，表达意图更准确，数据量增加后也更合适。

| 方案 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|
| `list` 记录已见 ID | 保留顺序、概念简单 | 成员查找随数据量增长而变慢 | 既要顺序又要重复值，且数据很少 |
| `set` 记录已见 ID | 去重语义直接、成员判断快 | 不应依赖集合遍历顺序 | 只关心某个 ID 是否出现过 |
| `dict` 以 ID 为键 | 可同时保存 ID 对应记录 | 若只判断存在，保存的信息偏多 | 后续还要按 ID 取出原记录 |

本例只需要判断是否重复，因此 `set` 是更贴合问题的数据结构。

### 3.2 为什么普通循环不能全换成推导式？

推导式适合单一的“映射或过滤”：例如将标签转小写、去空值、去重复。主清洗流程却包含多项校验、错误积累和多个提前跳过分支。把这些副作用塞进一个超长推导式会更难调试。

判断标准不是“能不能写成一行”，而是“一眼能否看出数据如何流动”。复杂分支用 `for + if`，简单转换用推导式。

### 3.3 重复数据保留哪一条？

本例采用“第一条合法记录优先”，规则稳定且容易解释。也可以选择最后一条覆盖，或将重复视为整批失败。没有永远正确的选择：测试执行数据通常应尽早暴露重复，因此本例保留第一条并报告问题，不静默覆盖。

### 3.4 回查 Requirement 1 的适用条款

`Req 1` 是范围标识，本章不能只写“学会数据类型”就宣称完整达标：

- **Req 1.1 适用**：示例有 `pyproject.toml`、`src/`、`tests/` 和独立虚拟环境；
- **Req 1.2 适用**：公开清洗函数带类型注解，清洗职责集中且数据流清楚；函数拆分的系统方法将在 1.3 深入；
- **Req 1.3 适用**：核心逻辑有自动化测试，并能由一条 pytest 命令运行；
- **Req 1.4 本例不适用**：核心逻辑没有文件、网络或时间依赖，不需要参数注入或 mock；
- **Req 1.5 适用**：模块文档、变量命名、规则说明和 README 让 reviewer 能独立读懂输入、分支与输出。

因此，本章重点实证模块①“数据类型与控制流”的验收，同时提供 Req 1 部分条款的练习证据，不替代后续章节对整个 Req 1 的最终验收。

## 4. 常见坑

| 坑 | 后果 | 根本原因 |
|---|---|---|
| 用列表保存已见 ID 并反复查找 | 数据量大时去重越来越慢 | 数据结构与“成员判断”需求不匹配 |
| 直接写 `if not expected_status` | `0`、`None`、空字符串被混为一类，类型错误不清晰 | 只判断真假值，没有验证类型和范围 |
| 忘记 `bool` 是 `int` 的子类 | `True` 被当作状态码 `1` 参与判断 | 不了解 Python 类型关系 |
| 用一个巨大推导式完成全部清洗 | 分支、副作用和错误定位难以理解 | 把“代码短”误当成“代码清晰” |
| 集合去重后直接输出 | 输出顺序可能不符合预期，测试结果不稳定 | 把集合当作有序展示结构 |
| 遇到第一项错误就丢弃记录 | 一条数据要反复修改多次才能看全问题 | 没有先积累本行所有错误 |
| 自动把 `"200"` 转成 `200` | 上游格式错误被隐藏，数据契约逐渐失控 | 没有先决定清洗与拒绝的边界 |

## 5. 最佳实践

1. **先写数据契约再写循环**：明确必填字段、类型、范围、去重规则，以及错误数据是拒绝还是修复。
2. **按访问需求选结构**：有序批次用 `list`，命名字段用 `dict`，成员判断和去重用 `set`；不要只因为熟悉就全部使用列表。
3. **先验证，再规范化，再入库**：格式不合法的数据不能进入后续去重和执行流程，避免污染状态。
4. **错误信息带定位信息**：至少包含行号、字段和原因，让测试人员能直接回到源数据修复。
5. **推导式只做简单转换**：当表达式包含多个分支、修改外部状态或难以加断点时，改用普通循环。
6. **稳定输出再测试**：集合用于内部去重，展示或断言前用 `sorted` 固定顺序。

这些做法适合批量测试数据、接口响应清洗和小型测试工具。若字段规则增长到几十项，应在后续章节把校验规则拆成职责单一的函数或数据模型，而不是无限延长一个循环。

## 6. 练习题

### 练习 1：清洗支付接口测试数据（基础）

**对应验收：模块①·数据类型；Req 1（适用 1.2、1.3）**

复制本章工程为你自己的练习目录，输入至少 8 条支付接口用例，并主动包含：

- 2 条合法记录；
- 1 条空 `case_id`；
- 1 组重复 `case_id`；
- `expected_status` 分别为 `None`、`"200"`、99 和 600 的记录；
- 重复且大小写不同的标签。

先在纸上写出哪些记录应保留，再运行清洗器对照。验收标准：合法记录顺序不变、ID 唯一、状态码均为 100~599 的非布尔整数，每条非法数据都有可定位的问题信息。

### 练习 2：增加 HTTP 方法规则（进阶）

**对应验收：模块①·数据类型；Req 1（适用 1.2、1.3、1.5）**

为每条记录增加 `method`，只允许 `GET`、`POST`、`PUT`、`PATCH`、`DELETE`。思考并实现：允许值应该用列表还是集合保存？输入 `" get "` 是规范化为 `GET`，还是报告错误？先写出团队规则，再补充自动化测试，最后请另一位同事只读 README 和代码说明数据流。

提示：如果需求是频繁做“是否允许”的判断，集合通常更贴切；但是否自动修复输入属于数据契约决策，不能仅由语法决定。

## 7. 小结与自检

- `list` 适合保存有顺序的批量数据，`dict` 适合表示命名字段，`set` 适合去重与成员判断。
- `for` 负责逐条处理，`if` 决定数据分支，`continue` 能让不合格记录尽早退出当前轮次。
- 推导式适合简单的映射和过滤；复杂校验保留普通循环，优先保证数据流清晰。
- 数据清洗必须明确“拒绝、修复、保留”的契约，不能为了让程序继续运行而静默吞掉问题。
- 本章通过独立工程、类型注解和 pytest 回查 Req 1 的适用条款，但 Req 1 的完整能力仍需后续章节共同验收。

自检问题（可选择跳过）：

1. 如果既要判断 ID 是否重复，又要在重复时取出第一条原记录，你会继续只用 `set`，还是改用 `dict`？为什么？
2. 为什么标签去重适合推导式，而整批用例清洗不适合压缩成一个推导式？
