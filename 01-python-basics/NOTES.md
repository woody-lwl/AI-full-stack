# 模块① Python 编程基础学习记录

## 1.1 环境与项目结构 — 2026-09-28
- 学了什么：使用 Python 3.12 虚拟环境、`pyproject.toml` 依赖声明和 `src/tests` 布局，搭建可安装、可运行、可测试的最小项目骨架。
- 关键要点（3~5 条）：
  - `.venv` 隔离本机已安装依赖，`pyproject.toml` 记录项目所需依赖，两者职责不同。
  - `src/` 存放可交付源码，`tests/` 存放自动化测试，避免源码、样例和验证代码混放。
  - pytest 固定为 `8.4.2`，通过测试可选依赖组安装，不修改系统 Python。
  - `.venv/` 已写入示例项目 `.gitignore`，虚拟环境不属于项目交付产物。
  - README 统一记录安装、运行与测试命令，使同事可以不依赖口头说明复现项目。
- 还有什么疑问 / 待深入：后续学习依赖锁定文件，以及项目规模扩大后的模块拆分；不在本章提前展开。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/01-env-and-structure/`
  - 环境创建与安装：`/opt/homebrew/bin/python3.12 -m venv .venv`；`.venv/bin/python -m pip install -e ".[test]"`，成功安装 `pytest==8.4.2` 和示例项目。
  - 版本检查：`.venv/bin/python --version` → `Python 3.12.14`。
  - 示例运行：`.venv/bin/python -m test_case_validator` → `用例校验通过：查询用户成功`。
  - 自动化测试：`.venv/bin/python -m pytest` → `10 passed in 0.02s`。
## 1.2 数据类型与控制流 — 2026-09-29
- 学了什么：用 `list`、`dict`、`set`、条件、循环和推导式实现接口测试数据清洗，识别空值、重复 `case_id`、非法状态码与错误记录格式。
- 关键要点（3~5 条）：
  - `list` 适合保存有顺序的批量记录，`dict` 适合表示命名字段，`set` 适合成员判断与去重。
  - 清洗流程遵循“先校验、再规范化、最后加入结果”，并用 `continue` 让非法记录尽早结束当前循环。
  - 复杂分支保留普通 `for + if`，推导式只负责标签规范化等简单转换，优先保证数据流清晰。
  - 重复 ID 采用“保留第一条合法记录并报告问题”的明确策略，不静默覆盖。
  - 边界测试覆盖空字段、重复值、布尔/字符串/越界状态码、非字典记录与错误标签格式；Hypothesis 验证输出 ID 唯一且状态码有效。
- 还有什么疑问 / 待深入：字段规则继续增加时，如何把当前清洗流程拆成职责单一的函数；留到 1.3“函数”章节学习，不在本章提前实现。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/02-data-types-and-control-flow/`
  - 重建隔离环境：`rm -rf .venv && /opt/homebrew/bin/python3.12 -m venv .venv`，仅重建本项目 `.venv`。
  - 安装固定构建后端：`.venv/bin/python -m pip install "setuptools==75.8.0"` → `Successfully installed setuptools-75.8.0`。
  - 安装项目与测试依赖：`.venv/bin/python -m pip install --no-build-isolation -e ".[test]"` → 成功构建并安装 `test-data-cleaner==0.1.0`、`pytest==8.4.2`、`hypothesis==6.112.2`；无需额外安装 wheel。
  - 版本检查：`.venv/bin/python --version` → `Python 3.12.14`。
  - 示例运行：`.venv/bin/python -m test_data_cleaner` → 保留 `login-001` 1 条合法用例，报告空 `case_id`、重复 ID、字符串状态码和非字典记录共 4 个问题，与教程预期一致。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `11 passed in 0.68s`。

## 目录映射说明
- 代码目录统一按对应教程 Markdown 的 stem 命名，便于章节与示例长期一一对应；目录内部的 Python 包名和发行项目名继续保留业务语义。以上已有运行/测试结果仅更新工作目录，不改写真实验证数据。
## 1.3 函数 — 2026-09-29
- 学了什么：把接口测试中重复的状态码、必填字段和响应耗时断言，重构为职责单一、带类型注解、可组合的函数，并用统一错误列表聚合检查结果。
- 关键要点（3~5 条）：
  - 函数通过明确的参数、处理和返回值封装稳定业务规则，减少多份重复实现逐渐不一致的风险。
  - 核心响应数据可用位置参数传入；配置策略使用关键字参数，调用处更清楚且不易传错顺序。
  - 核心检查函数返回 `list[str]`，打印留在入口层，使纯计算与副作用分离，结果可供 pytest、命令行和报告复用。
  - `DEFAULT_REQUIRED_FIELDS` 使用不可变 `tuple`；每次组合检查都创建局部 `errors`，避免默认参数或全局状态导致跨调用污染。
  - 职责单一应按“变化原因”划分，而不是机械限制函数行数；组合函数只负责编排和聚合，不重复实现规则。
- 还有什么疑问 / 待深入：当检查规则增长到很多项时，如何把函数拆到不同模块并组织导入；留到 1.4“模块与包”学习，本任务未提前实现。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/03-functions/`
  - 创建隔离环境：`/opt/homebrew/bin/python3.12 -m venv .venv`。
  - 安装固定构建后端：`.venv/bin/python -m pip install "setuptools==75.8.0"` → `Successfully installed setuptools-75.8.0`。
  - 安装项目与测试依赖：`.venv/bin/python -m pip install --no-build-isolation -e ".[test]"` → 成功构建并安装 `api-response-checker==0.1.0`、`pytest==8.4.2`。
  - 版本检查：`.venv/bin/python --version` → `Python 3.12.14`。
  - 示例运行：`.venv/bin/python -m api_response_checker` → `login-001` 通过；`order-001` 按教程顺序报告状态码、两个缺失字段和耗时超限共 4 个问题。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `8 passed in 0.01s`。

## 1.4 模块与包 — 2026-09-29
- 学了什么：把同时负责读取 JSON、校验字段、生成报告和运行流程的超单文件接口测试脚本，按变化原因拆成 `loader`、`validator`、`reporter` 与 `__main__` 模块，并用 `api_test_toolkit` 包组织和正确导入。
- 关键要点（3~5 条）：
  - 模块是可导入的 `.py` 文件，包用于组织相关模块；拆分应依据职责和变化原因，而不是机械按文件行数切分。
  - 数据按“加载 → 校验 → 报告”单向流动，入口只负责编排，使各业务模块可独立测试，并从结构上避免循环导入。
  - `__init__.py` 通过重新导出和 `__all__` 声明稳定公共 API；包内相邻模块可用显式相对导入，入口和包外调用使用清晰的绝对导入。
  - `__main__.py` 提供 `python -m api_test_toolkit` 标准入口；模块顶层不执行读取业务文件等副作用。
  - `src/` 项目通过可编辑安装进入 Python 导入搜索路径，不依赖修改 `sys.path` 或碰巧从特定目录运行。
- 还有什么疑问 / 待深入：多个模块需要共同管理相关状态与行为时，何时应继续使用函数，何时适合引入类；留到 1.5“面向对象”学习，本任务未提前实现。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/04-modules-and-packages/`
  - 创建隔离环境：`/opt/homebrew/bin/python3.12 -m venv .venv`。
  - 安装固定构建后端：`.venv/bin/python -m pip install "setuptools==75.8.0"` → `Successfully installed setuptools-75.8.0`。
  - 安装项目与测试依赖：`.venv/bin/python -m pip install --no-build-isolation -e ".[test]"` → 成功构建并安装 `api-test-toolkit==0.1.0`、`pytest==8.4.2`。
  - 版本检查：`.venv/bin/python --version` → `Python 3.12.14`。
  - 示例运行：`.venv/bin/python -m api_test_toolkit` → 共读取 3 条用例；`login-001` 通过，`order-002` 报告缺少 `endpoint`，`refund-003` 报告状态码超出 100~599，汇总为 1 通过、2 失败，与教程预期一致。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `12 passed in 0.04s`；覆盖各模块职责、正常/缺字段/非法字段、非法 JSON、错误 JSON 容器、空报告、包公共导入和真实 `python -m` 入口集成。

## 1.5 面向对象 — 2026-09-29
- 学了什么：以一次接口测试运行为真实场景，用 `TestRun` 把持续存在的用例、执行结果和汇总行为放在同一边界；用冻结 dataclass 表达简单数据对象，并通过组合报告器与单层继承区分 has-a / is-a。
- 关键要点（3~5 条）：
  - 类适合管理跨多次调用持续存在的相关状态与不变量；纯输入输出转换继续用函数，简单数据优先用 dataclass，不为“显得专业”而使用 OOP。
  - `__init__` 创建每个实例独立的 `_cases` 与 `_results`；测试证明两个 `TestRun` 实例的状态不会串扰。
  - Python 单下划线是内部实现约定，不是绝对私有；只读属性、tuple 快照与受控写方法共同避免外部绕过规则修改状态。
  - `TestRun` 组合 `TextReporter`，表达 has-a 并让运行逻辑与报告格式独立变化；组合通常优先于继承。
  - `ApiTestCase is-a TestCase` 是浅而稳定的继承关系，子类可替代父类并只覆盖描述行为；继承不用于单纯复用几行代码。
- 还有什么疑问 / 待深入：当对象方法遇到文件格式错误、未知用例或外部超时时，如何设计清晰的异常边界与可定位日志；留到 1.6“异常与日志”学习，本任务未提前实现。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/05-oop/`
  - 创建隔离环境：`/opt/homebrew/bin/python3.12 -m venv .venv`。
  - 安装固定构建后端：`.venv/bin/python -m pip install "setuptools==75.8.0"` → `Successfully installed setuptools-75.8.0`。
  - 安装项目与测试依赖：`.venv/bin/python -m pip install --no-build-isolation -e ".[test]"` → 成功构建并安装 `test-run-manager==0.1.0`、`pytest==8.4.2`。
  - 版本检查：`.venv/bin/python --version` → `Python 3.12.14`。
  - 示例运行：`.venv/bin/python -m test_run_manager` → 共 3 条用例，2 通过、1 失败、0 待执行；`ApiTestCase` 在报告中展示 HTTP 方法与路径，输出与教程预期一致。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `13 passed in 0.04s`；覆盖初始状态、添加/重复用例、未知/重复结果、通过/失败/待执行汇总、冻结与只读边界、组合报告器、继承替换、公共 API、实例状态隔离和真实 `python -m` 入口。

## 1.6 异常与日志 — 2026-09-29
- 学了什么：以 JSON 批量测试任务为场景，建立“底层转换技术异常、流程层决定继续或终止、入口层记录最终失败”的异常边界，并用 logging 留下可定位且不泄密的诊断证据。
- 关键要点（3~5 条）：
  - 异常负责沿调用链传递失败，日志负责记录时间、模块、`case_id`、来源文件、动作和原因；两者职责不同，不应每层重复记录同一错误。
  - loader/executor 只捕获 `FileNotFoundError`、`JSONDecodeError`、`TimeoutError` 等具体技术异常，用 `raise ... from ...` 转成 `TestDataError` 或 `CaseTimeoutError` 并保留异常链。
  - runner 用 `try/except/else/finally` 区分超时恢复、正常结果处理与必要收尾；只处理有明确策略的超时，未知 `RuntimeError` 继续向上暴露。
  - 状态码断言失败是预期业务结果，记录 WARNING 并汇总，不等同于程序异常；整批数据不可用由入口使用 `logger.exception()` 记录 ERROR 和堆栈。
  - 模块通过 `logging.getLogger(__name__)` 获取 logger，入口统一配置；日志使用参数化模板，并通过字段允许列表避免 token、密码和完整隐私数据进入日志。
- 还有什么疑问 / 待深入：真实 HTTP 请求的超时、连接错误和有限重试如何通过依赖注入稳定测试；留到接口自动化与 mock 章节结合幂等性继续学习。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/06-exceptions-and-logging/`
  - 创建隔离环境：`/opt/homebrew/bin/python3.12 -m venv .venv`。
  - 安装固定构建后端：`.venv/bin/python -m pip install "setuptools==75.8.0"` → `Successfully installed setuptools-75.8.0`。
  - 安装项目与测试依赖：`.venv/bin/python -m pip install --no-build-isolation -e ".[test]"` → 成功构建并安装 `test-job-runner==0.1.0`、`pytest==8.4.2`。
  - 版本检查：`.venv/bin/python --version` → `Python 3.12.14`。
  - 示例运行：`.venv/bin/python -m test_job_runner` → `login-001` INFO 通过，`order-002` WARNING 断言未通过，`refund-003` WARNING 超时后跳过；最终 `total=3 completed=3 passed=1 assertion_failed=1 timed_out=1`，日志均包含来源与动作且未输出示例 token。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `16 passed in 0.06s`；覆盖正常输入、文件不存在、非法 JSON、错误顶层结构、空字段、自定义异常链、超时处理与继续执行、日志级别/上下文、敏感值不泄露、入口堆栈及不可恢复异常不被吞掉。

## 1.7 文件与数据处理 — 2026-09-29
- 学了什么：以 CSV 测试结果转换为 JSON 报告为场景，实践 UTF-8 文本读写、CSV 表头契约、行级数据清洗、稳定 JSON 输出，以及文件副作用与纯转换逻辑的分离。
- 关键要点（3~5 条）：
  - 使用 `Path`、`with`、显式 `encoding="utf-8"` 和 `newline=""` 管理跨平台文件路径、资源、编码与 CSV 换行。
  - 把文件级错误与行级错误分开：空文件、缺列、错误编码和 CSV 语法错误终止转换；重复 ID、非法状态或耗时进入 `issues` 后继续。
  - 将读取、纯转换、报告组装和写入按职责拆分，使核心清洗规则可直接用内存数据测试，文件边界则用 `tmp_path` 隔离。
  - JSON 报告固定字段顺序，记录按 `case_id` 排序，问题按原行号排序，并用 `ensure_ascii=False` 保留可读中文。
  - 测试覆盖正常转换、中文 UTF-8、空文件、缺列、编码/CSV 格式异常、重复 ID、非法 status/duration、输出目录创建、JSON 回读、入口行为和 `tmp_path` 隔离。
- 还有什么疑问 / 待深入：当 CSV 增长到数百万行时，如何在保持失败可恢复和输出完整性的前提下改为流式处理；留待真实规模出现后再基于测量重构。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/07-file-and-data/`
  - 重建隔离环境：仅删除本项目 `.venv`，使用 `/opt/homebrew/bin/python3.12 -m venv .venv` 成功重建。
  - 安装固定构建后端：`.venv/bin/python -m pip install "setuptools==75.8.0"` → `Successfully installed setuptools-75.8.0`。
  - 安装项目与测试依赖：`.venv/bin/python -m pip install --no-build-isolation -e ".[test]"` → 成功构建并安装 `test-result-converter==0.1.0`、`pytest==8.4.2`。
  - 版本检查：`.venv/bin/python --version` → `Python 3.12.14`。
  - 示例运行：`.venv/bin/python -m test_result_converter` → `total=4 passed=2 failed=2 issues=3`。
  - JSON 核对：`output/report.json` 可按 UTF-8 回读，中文直接显示；4 条 records 与 3 条 issues 的内容、排序和行号均与教程预期一致。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `18 passed in 0.06s`；在测试侧将导入的 `TestResult` 语义别名为 `ResultRecord`，消除了 pytest 对候选测试类的误收集 warning，生产数据类名称与公共 API 保持不变。

## 1.8 标准库 — 2026-09-29
- 学了什么：以扫描多份 JSON 测试报告为连贯场景，用 Python 标准库完成最近时间窗筛选、状态统计、suite 分组和失败用例分批复测，并练习先识别问题类别再选择工具。
- 关键要点（3~5 条）：
  - 使用 `Path.glob()` 扫描并稳定排序报告路径；路径职责统一交给 `pathlib`，`os` 只用于读取环境变量，避免两套路径抽象混用。
  - 使用带时区的 aware `datetime` 和 `timedelta` 定义闭区间时间窗；固定并注入演示基准时间，避免 CLI 输出和测试随当前时间漂移。
  - 使用 `Counter` 表达状态计数，利用缺失键返回 0 的语义；使用 `defaultdict(list)` 构建分组后转为普通 `dict`，阻止查询缺失 suite 时产生副作用。
  - 使用 `chain.from_iterable()` 惰性展平多份报告，使用 Python 3.12 `batched()` 惰性生成复测批次；明确迭代器只能消费一次，排序或重复访问时才有意识地物化。
  - 环境变量始终先以字符串进入程序，统一在配置边界解析并校验 `LOOKBACK_HOURS`、`BATCH_SIZE` 和带时区的 `REFERENCE_TIME`；每次分析均创建独立计数与分组状态。
- 还有什么疑问 / 待深入：当报告增长到数百万条、需要列式聚合或持久化分布式重试时，如何用性能数据评估标准库方案与成熟第三方库；留到真实规模与后续工程化章节再决策。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/08-standard-library/`
  - 创建隔离环境：`/opt/homebrew/bin/python3.12 -m venv .venv`。
  - 安装固定构建后端：`.venv/bin/python -m pip install "setuptools==75.8.0"` → `Successfully installed setuptools-75.8.0`。
  - 安装项目与测试依赖：`.venv/bin/python -m pip install --no-build-isolation -e ".[test]"` → 成功构建并安装 `test-report-analyzer==0.1.0`、`pytest==8.4.2`。
  - 版本检查：`.venv/bin/python --version` → `Python 3.12.14`。
  - 示例运行：`.venv/bin/python -m test_report_analyzer` → 扫描 2 份报告，窗口内 5 条记录，`passed=2 failed=3`；按冒烟/回归/支付分组，并生成 `refund-005,payment-002` 与 `order-003` 两个复测批次，输出与教程预期一致。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `17 passed in 0.04s`，无 warning；覆盖路径扫描、ISO 时间及时区边界、时间窗内外、Counter、suite 分组、惰性分批、空输入、环境变量默认/非法值、调用间状态隔离和 CLI 集成。

## 1.9 类型注解与代码规范 — 2026-09-30
- 学了什么：为测试用例选择器补全并收紧类型契约，使用 mypy 检查静态类型边界，使用 Ruff 同时执行 lint 与格式门禁，并修复优先级排序在泛型上下文中的类型推断问题。
- 关键要点（3~5 条）：
  - `_PRIORITY_ORDER` 保持 `dict[Priority, int]`，通过 `_priority_rank(priority: Priority) -> int` 集中优先级映射，不用 `cast`、`ignore` 或放宽为 `dict[str, int]` 掩盖问题。
  - 外层 `",".join(...)` 会给 `sorted()` 提供字符串上下文；先把排序结果显式绑定为 `list[Priority]`，可避免元素类型被反向拓宽为普通 `str`。
  - `select_cases()` 只要求可迭代能力，因此接收 `Iterable[TestCase]`；`summarize_selection()` 需要 `len()`，因此接收 `Sequence[TestCase]`。
  - 静态类型检查、行为测试、lint 和格式检查职责不同，必须共同通过，不能用其中一项替代其他门禁。
  - Ruff 的规则同样覆盖 docstring 中易混淆的 Unicode 标点和导入分组；规范问题应做局部修复，不降低规则。
- 还有什么疑问 / 待深入：当多个泛型函数嵌套调用时，如何系统判断类型是由实参、赋值目标还是外层调用的上下文推断；后续可结合更多 mypy 诊断案例继续练习。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/09-type-hints-and-lint/`
  - 示例运行：`.venv/bin/python -m test_case_selector` → `input=5 selected=2 skipped=3`，选中 `login-smoke,payment-smoke`。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `15 passed in 0.05s`，无 warning。
  - 静态类型检查：`.venv/bin/python -m mypy src tests` → `Success: no issues found in 8 source files`。
  - Lint 检查：`.venv/bin/python -m ruff check .` → `All checks passed!`。
  - 格式检查：`.venv/bin/python -m ruff format --check .` → `8 files already formatted`。

## 1.10 设计思维（分层/抽象/解耦）— 2026-09-30
- 学了什么：把同时读取用例、调用接口、校验响应和输出报告的面条脚本，重构为 domain 纯规则、application 流程编排、ports 最小契约、adapters 外部实现与 entrypoint 组装边界，并用 Mermaid 区分运行时数据流和源码依赖方向。
- 关键要点（3~5 条）：
  - 分层按“变化原因”划分，而不是按行数或架构模板切目录；domain 不依赖文件、HTTP 或打印，application 只保留“加载→执行→校验→输出”的流程语言。
  - `CaseSource`、`ApiClient`、`ResultSink` 使用 `Protocol` 表达协作所需的最小能力；纯 `validate_response()` 没有机械包装成接口，避免为抽象而抽象。
  - 构造函数依赖注入把具体实现的创建权留给入口；测试可注入 `FakeApiClient`、内存用例源和 collecting sink，在不读真实文件、不发真实网络的条件下验证核心流程。
  - 运行时 application 会调用注入的 adapter，但源码只依赖端口；domain 不反向导入外层，具体 adapter 的替换不会修改核心规则。
  - 设计是取舍：一次性小脚本可保持单文件，稳定小工具可简单模块拆分，只有出现明确外部变化点或测试替换点时才引入协议与适配器。
- 还有什么疑问 / 待深入：当同一端口出现多种创建策略、重试策略或装饰需求时，怎样选择工厂、策略、装饰器且避免模式滥用；留到 1.11“设计模式在测试中的应用”。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/10-design-thinking/`
  - 创建隔离环境：`/opt/homebrew/bin/python3.12 -m venv .venv`；版本为 `Python 3.12.14`。
  - 安装固定构建后端与开发门禁：`setuptools==75.8.0`，再以 `--no-build-isolation -e ".[dev]"` 安装 `pytest==8.4.2`、`mypy==1.16.1`、`ruff==0.11.13`。
  - 示例运行：`.venv/bin/python -m layered_test_runner` → `total=3 passed=2 failed=1`；`login-001`、`health-003` 通过，`order-002` 因 fake 返回 500 失败，与教程预期一致且未访问真实网络。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `17 passed in 0.02s`，无 warning；覆盖纯领域规则、编排顺序、fake 注入与无网络、替换用例源/结果输出、三类外部异常边界、空用例、完整入口数据流和实例隔离。
  - 静态类型检查：`.venv/bin/python -m mypy src tests` → `Success: no issues found in 10 source files`。
  - Lint 检查：`.venv/bin/python -m ruff check .` → `All checks passed!`。
  - 格式检查：`.venv/bin/python -m ruff format --check .` → `10 files already formatted`。

## 1.11 设计模式在测试中的应用 — 2026-09-30
- 学了什么：以可配置接口响应校验框架为场景，用函数工厂集中校验配置并创建规则，用策略对象封装状态码与必填字段算法，用可注入 clock/sink 的装饰器统一记录确定性耗时，同时判断何时应保留简单分支而不使用模式。
- 关键要点（3~5 条）：
  - 工厂负责“合法地创建哪个对象”，策略负责“运行哪种算法”，装饰器负责“在调用前后附加什么横切行为”；三者按不同变化原因分工。
  - `ValidationRunner` 只依赖带只读 `name` 属性和 `validate()` 方法的 `Protocol`；新增测试专用策略无需修改执行器，且冻结 dataclass 与静态类型契约保持一致。
  - 函数工厂适合少量固定规则；只有需要动态注册、共享依赖或缓存时才考虑工厂类。简单且稳定的判断继续使用 `if` 或映射表更经济。
  - `functools.wraps` 保留函数元数据，`try/finally` 让正常与异常路径都记录耗时；装饰器顺序会改变总耗时/单次耗时语义，clock 与 sink 自身的异常策略也必须明确。
  - 时间和事件接收器由外部注入，测试使用固定时钟与内存列表，不访问网络、不依赖真实耗时，并验证异常传播和实例状态隔离。
- 还有什么疑问 / 待深入：规则规模增长并需要第三方动态注册时，如何把当前函数工厂演进为受控注册表，同时避免插件系统隐藏依赖；留到 2.6“测试框架设计”结合真实扩展需求再学习。
- 运行/测试证据：
  - 工作目录：`codelearn/01-python-basics/code/11-design-patterns-in-testing/`
  - 环境与依赖：使用 `/opt/homebrew/bin/python3.12` 创建 `.venv`；安装 `setuptools==75.8.0`，再以 `--no-build-isolation -e ".[dev]"` 安装项目及 `pytest==8.4.2`、`mypy==1.16.1`、`ruff==0.11.13`。
  - 示例运行：`.venv/bin/python -m test_validation_patterns` → 创建 `status_code,required_fields` 两个策略，按顺序报告状态码不匹配和缺少 `token`，固定时钟输出 `elapsed_ms=25.0`，与 README 和教程一致。
  - 自动化测试：`.venv/bin/python -m pytest --color=no` → `19 passed in 0.05s`，无 warning；覆盖策略成功/失败、聚合顺序、工厂合法/非法配置、新策略替换、装饰器元数据、固定时钟、异常传播、状态隔离和真实模块入口。
  - 静态类型检查：`.venv/bin/python -m mypy src tests` → `Success: no issues found in 12 source files`。
  - Lint 检查：`.venv/bin/python -m ruff check .` → `All checks passed!`。
  - 格式检查：`.venv/bin/python -m ruff format --check .` → `12 files already formatted`。

## 1.12 模块①验收检查点
- 状态：**教程产物门禁：通过；学习者能力门禁：学习者自评通过（2026-10-01）**。
- 验收边界：本记录只证明 1.1~1.11 的教程、示例工程和自动化门禁符合当前 spec；不能据此推断学习者本人已经掌握；能力门禁以学习者自评为准，1.12 与父任务 1 已于 2026-10-01 勾选。
- 文档/结构门禁：
  - 11 篇章节 Markdown 与 `code/` 下 11 个章级目录数量一致、stem 一一对应，无缺失或多余目录。
  - 每章均包含目录及“场景引入、可运行代码示例、设计思考、常见坑、最佳实践、练习题、小结与自检”七个核心部分。
  - 每章均有章头验收标识、Python 3.12+ 声明及指向同 stem 工程的相对链接；每个工程 README 均能反向链接对应章节，并给出明确入口与测试命令。
  - `NOTES.md` 已包含 1.1~1.11 的所学内容、关键要点、遗留问题及历史运行/测试证据。
- 本次自动化门禁（均使用各工程已有 `.venv`，未安装或升级依赖）：
  - 11/11 个工程执行 `.venv/bin/python --version` 均为 `Python 3.12.14`；11/11 个 README 声明的模块入口均成功运行。
  - 完整 pytest 结果依次为：1.1 `10 passed`、1.2 `11 passed`、1.3 `8 passed`、1.4 `12 passed`、1.5 `13 passed`、1.6 `16 passed`、1.7 `18 passed`、1.8 `17 passed`、1.9 `15 passed`、1.10 `17 passed`、1.11 `19 passed`；合计 **156 passed**，无 warning、无失败。
  - 1.9、1.10、1.11 的 `mypy src tests`、`ruff check .`、`ruff format --check .` 全部通过；分别检查 8、10、12 个文件。
  - 使用 Python 3.12 AST 额外审计 11 个 `src/`：共识别 91 个公开函数/方法，参数或返回值类型注解缺失数为 0。
- Requirement 1.1~1.5 产物证据：
  - **Req 1.1 标准结构**：11 个工程均具备 `pyproject.toml`、`src/`、`tests/`、README 和可用 `.venv`，依赖与 Python 版本要求有文件化声明。
  - **Req 1.2 职责与类型**：1.3 展示按检查规则拆分职责；1.4 展示按变化原因拆模块；1.5 区分函数、数据对象和有状态类；公开函数/方法注解审计无缺失。
  - **Req 1.3 自动化测试**：11 个工程完整 pytest 共 156 项全部通过，覆盖正常、边界、错误输入与模块入口。
  - **Req 1.4 外部依赖隔离**：1.7 文件测试使用 `tmp_path`；1.10 通过端口和构造函数注入替换文件源、API 客户端与结果输出，并验证 fake 不建立网络连接；1.11 注入 clock 与 timing sink，使时间逻辑可确定测试。
  - **Req 1.5 可理解性**：1.4 的 loader/validator/reporter/入口职责、1.10 的 domain/application/ports/adapters/entrypoint 依赖方向，以及章节 Mermaid 数据流、README 目录职责和聚焦测试共同提供 review 证据。是否能由学习者本人独立设计和讲解，仍须能力门禁确认。
- 模块①章节验收证据：11 个章节场景分别覆盖项目骨架、数据清洗、函数抽取、模块拆分、OOP 建模、异常与日志、CSV/JSON 转换、标准库组合、类型与规范、分层解耦、工厂/策略/装饰器；对应入口和测试均已通过。
- 发现的问题：无阻断问题，无 warning；本检查点未修改任何教程/示例源码；`tasks.md` 仅更新 1.12 与父任务 1 的完成状态。
- 能力门禁记录（2026-10-01）：验收方式为学习者自评通过，未进行导师出题验收。建议在 2.1 pytest 进阶的练习中顺带回查 1.10 分层/依赖注入与 1.11 策略/装饰器的掌握情况。
