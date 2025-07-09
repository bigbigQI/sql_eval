# Text2SQL 评估系统

这是一个完整的Text2SQL评估系统，用于评估大语言模型在SQL生成任务上的性能。该系统支持多轮对话、工具调用和全面的结果分析。

## 功能特性

- 🔄 **多轮对话管理**: 支持与SGLang服务器的多轮对话交互
- 🛠️ **SQL工具调用**: 集成SQL执行工具，支持实际数据库查询
- 📊 **全面评估**: 多维度评估指标，包括执行准确率、格式准确率等
- 📈 **结果分析**: 详细的错误分析、性能分析和改进建议
- 🚀 **并发处理**: 支持多并发请求，提高评估效率
- 📁 **多数据源**: 支持SynSQL、Spider、BIRD等多种数据源

## 系统架构

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   评估数据集     │    │   SGLang Server │    │   SQL数据库     │
│   - 问题        │    │   - 模型推理    │    │   - 执行环境    │
│   - 数据库信息   │───►│   - 多轮对话    │◄──►│   - 结果验证    │
│   - Golden SQL │    │   - 工具调用    │    │   - 性能测试    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                      评估脚本                                    │
│  - Prompt构造  - 请求管理  - 结果解析  - 性能评估  - 报告生成    │
└─────────────────────────────────────────────────────────────────┘
```

## 安装依赖

```bash
pip install aiohttp pandas sqlite3 pathlib
```

可选依赖：
```bash
pip install func_timeout  # 用于SQL执行超时控制
```

## 快速开始

### 1. 准备数据

确保你有以下数据：
- 评估数据集文件（支持.json、.jsonl、.parquet格式）
- 数据库文件（SQLite格式）
- 运行中的SGLang服务器

### 2. 启动SGLang服务器

```bash
# 启动SGLang服务器（示例）
python -m sglang.launch_server \
    --model-path /path/to/your/model \
    --host 0.0.0.0 \
    --port 8001
```

### 3. 运行评估

```bash
python -m sql_eval.main_eval \
    --dataset_path /path/to/your/dataset.json \
    --db_root_path /path/to/your/databases \
    --server_url http://localhost:8001 \
    --sample_size 1 \
    --output_dir ./results
```

## 详细使用说明

### 命令行参数

#### 必需参数

- `--dataset_path`: 评估数据集文件路径
- `--db_root_path`: 数据库文件根目录
- `--server_url`: SGLang服务器URL

#### 可选参数

- `--output_dir`: 结果输出目录（默认：./results）
- `--sample_size`: 评估样本数量（默认：全部）
- `--max_turns`: 最大对话轮数（默认：6）
- `--concurrent_requests`: 并发请求数（默认：5）
- `--conversation_timeout`: 对话超时时间（默认：300秒）
- `--sql_timeout`: SQL执行超时时间（默认：30秒）
- `--max_result_chars`: SQL结果最大字符数（默认：9000）
- `--max_result_rows`: SQL结果最大行数（默认：50）
- `--log_level`: 日志级别（默认：INFO）

### 数据格式

#### 评估数据集格式

```json
[
  {
    "id": "sample_1",
    "question": "How many users are there in the database?",
    "db_id": "company_db",
    "data_source": "synsql",
    "sql": "SELECT COUNT(*) FROM users;",
    "external_knowledge": "",
    "difficulty": "easy"
  }
]
```

#### 目录结构

```
db_root_path/
├── spider/
│   └── database/
│       └── company_db/
│           └── company_db.sqlite

```

## 模块说明

### 1. 数据预处理模块 (`data_preprocess.py`)

负责数据加载、Prompt构造和数据库schema获取。

```python
from sql_eval import DatabaseManager, PromptBuilder, DataProcessor

# 创建数据库管理器
db_manager = DatabaseManager("/path/to/databases")

# 创建Prompt构造器
prompt_builder = PromptBuilder(db_manager)

# 创建数据处理器
data_processor = DataProcessor(db_manager, prompt_builder)

# 加载数据
samples = data_processor.load_dataset("dataset.json")
```

### 2. 对话管理模块 (`conversation_manager.py`)

管理与SGLang服务器的多轮对话。

```python
from sql_eval import ConversationManager, SQLToolClient

# 创建SQL工具客户端
sql_client = SQLToolClient("/path/to/databases")

# 创建对话管理器
async with ConversationManager(
    server_url="http://localhost:8001",
    sql_tool_client=sql_client,
    max_turns=6
) as manager:
    result = await manager.run_conversation(
        initial_messages=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Generate SQL for: How many users?"}
        ],
        db_id="company_db",
        data_source="synsql"
    )
```

### 3. 评估模块 (`evaluator.py`)

执行SQL结果的评估和比较。

```python
from sql_eval import Text2SQLEvaluator, SQLToolClient

# 创建评估器
sql_client = SQLToolClient("/path/to/databases")
evaluator = Text2SQLEvaluator(sql_client)

# 评估单个样本
result, metrics = evaluator.evaluate_single_sample(
    prediction_text="<answer>SELECT COUNT(*) FROM users;</answer>",
    ground_truth_sql="SELECT COUNT(*) FROM users;",
    db_id="company_db",
    data_source="synsql"
)
```

### 4. 结果分析模块 (`result_analyzer.py`)

分析评估结果并生成报告。

```python
from sql_eval import ResultAnalyzer

# 创建分析器
analyzer = ResultAnalyzer()

# 分析结果
analysis = analyzer.analyze_results(evaluation_results, sample_data)

# 生成报告
report = analyzer.generate_report("report.txt")
```

## 输出文件说明

运行完成后，会在输出目录生成以下文件：

- `raw_results_YYYYMMDD_HHMMSS.json`: 原始评估结果
- `analysis_YYYYMMDD_HHMMSS.json`: 详细分析结果
- `report_YYYYMMDD_HHMMSS.txt`: 人类可读的评估报告
- `sql_eval.log`: 运行日志

## 评估指标

### 核心指标

- **执行准确率**: SQL查询执行结果与标准答案的匹配率
- **SQL提取率**: 从模型响应中成功提取SQL的比例
- **SQL有效性**: 提取的SQL语法是否正确
- **预测成功率**: 预测SQL执行成功的比例
- **标准答案成功率**: 标准答案SQL执行成功的比例

### 分析维度

- **按难度分析**: 简单、中等、困难问题的性能对比
- **按数据源分析**: 不同数据源的性能对比
- **错误类型分析**: 解析错误、执行错误、结果不匹配等
- **性能分析**: 执行时间分布、并发性能等

## 故障排除

### 常见问题

1. **数据库文件未找到**
   - 检查数据库文件路径是否正确
   - 确认数据库文件存在且可访问

2. **SGLang服务器连接失败**
   - 检查服务器URL是否正确
   - 确认服务器正在运行
   - 检查网络连接

3. **SQL执行超时**
   - 增加`--sql_timeout`参数
   - 检查数据库文件是否损坏
   - 优化SQL查询复杂度

4. **内存不足**
   - 减少`--concurrent_requests`参数
   - 增加`--sample_size`限制评估样本数量
   - 减少`--max_result_chars`和`--max_result_rows`

### 调试方法

1. **启用DEBUG日志**
   ```bash
   python -m sql_eval.main_eval --log_level DEBUG [其他参数]
   ```

2. **小样本测试**
   ```bash
   python -m sql_eval.main_eval --sample_size 10 [其他参数]
   ```

3. **检查日志文件**
   ```bash
   tail -f sql_eval.log
   ```

## 性能优化

### 并发调优

- 根据服务器性能调整`--concurrent_requests`
- 监控系统资源使用情况
- 避免过高的并发导致服务器过载

### 数据库优化

- 确保数据库文件存储在高速存储设备上
- 定期检查数据库文件完整性
- 使用SSD提高查询性能

### 网络优化

- 确保评估客户端和SGLang服务器之间网络延迟低
- 考虑在同一台机器上运行以避免网络开销

## 扩展开发

### 添加新的数据源

1. 在`DatabaseManager.get_db_path()`中添加新的数据源逻辑
2. 更新数据处理器以支持新的数据格式
3. 添加相应的测试用例

### 自定义评估指标

1. 继承`EvaluationMetrics`类
2. 在`Text2SQLEvaluator`中实现新的评估逻辑
3. 更新结果分析器以支持新指标

### 添加新的模型服务器

1. 在`ConversationManager`中添加新的API适配器
2. 实现相应的消息格式转换
3. 添加配置参数支持
