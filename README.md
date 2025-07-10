# Text2SQL 评估系统

这是一个完整的Text2SQL评估系统，用于评估大语言模型在SQL生成任务上的性能。该系统支持多轮对话、工具调用、多轨迹生成和全面的结果分析。

## 功能特性

- 🔄 **多轮对话管理**: 支持与SGLang服务器的多轮对话交互
- 🛠️ **SQL工具调用**: 集成SQL执行工具，支持实际数据库查询
- 🎯 **多轨迹生成**: 支持为每个样本生成多条轨迹，提供样本级别和轨迹级别的准确率
- ⚙️ **模型参数配置**: 支持灵活配置模型名称、温度、最大token数等参数
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
pip install -r requirements.txt
```

## 快速开始

### 1. 准备数据

确保你有以下数据：
- 评估数据集文件（JSON格式）
- 数据库文件（SQLite格式）
- 运行中的SGLang服务器

### 2. 启动SGLang服务器

```bash
# 启动SGLang服务器（示例）
python -m sglang.launch_server \
    --model-path /path/to/your/model \
    --host 0.0.0.0 \
    --port 30000
```

### 3. 运行评估

```bash
# 基础评估（单轨迹）
python main_eval.py \
    --dataset_path spider_data/filter_test.json \
    --db_root_path spider_data \
    --server_url http://localhost:30000

# 多轨迹评估
python main_eval.py \
    --dataset_path spider_data/filter_test.json \
    --db_root_path spider_data \
    --server_url http://localhost:30000 \
    --n 5 \
    --temperature 0.8 \
    --model_name "qwen2.5-7b-instruct"
```

## 命令行参数

### 必需参数

- `--dataset_path`: 评估数据集文件路径
- `--db_root_path`: 数据库文件根目录
- `--server_url`: SGLang服务器URL

### 核心参数

- `--n`: 每个样本生成的轨迹数量（默认：1）
- `--model_name`: 模型名称（默认：qwen2.5-7b-instruct）
- `--temperature`: 温度系数（默认：0.6）
- `--max_tokens`: 最大token数（默认：30000）
- `--stream`: 是否启用流式输出（默认：False）

### 其他参数

- `--output_dir`: 结果输出目录（默认：./results）
- `--sample_size`: 评估样本数量（默认：全部）
- `--max_turns`: 最大对话轮数（默认：6）
- `--concurrent_requests`: 并发请求数（默认：5）
- `--conversation_timeout`: 对话超时时间（默认：300秒）
- `--log_level`: 日志级别（默认：ERROR）

## 多轨迹功能

### 概念说明

- **轨迹**: 对于同一个问题，模型生成的一次完整对话过程
- **样本级别准确率**: 在n条轨迹中，只要有一条轨迹正确，该样本就算正确
- **轨迹级别准确率**: 所有轨迹的平均正确率

### 使用示例

```bash
# 生成3条轨迹，使用较高温度增加多样性
python main_eval.py \
    --dataset_path spider_data/filter_test.json \
    --db_root_path spider_data \
    --server_url http://localhost:30000 \
    --n 3 \
    --temperature 0.8

# 生成5条轨迹，使用不同模型
python main_eval.py \
    --dataset_path spider_data/filter_test.json \
    --db_root_path spider_data \
    --server_url http://localhost:30000 \
    --n 5 \
    --model_name "your_model_name" \
    --temperature 0.7 \
    --max_tokens 20000
```

## 数据格式

### 评估数据集格式

```json
[
  {
    "id": "sample_1",
    "question": "How many users are there in the database?",
    "db_id": "company_db",
    "data_source": "spider",
    "ground_truth_sql": "SELECT COUNT(*) FROM users;",
    "difficulty": "easy"
  }
]
```

### 目录结构

```
db_root_path/
├── test_database/
│   └── company_db/
│       ├── schema.sql
│       └── company_db.sqlite
```

## 输出结果

### 输出文件

运行完成后，会在输出目录生成以下文件：

- `raw_results_YYYYMMDD_HHMMSS.json`: 原始评估结果（包含所有轨迹）
- `analysis_YYYYMMDD_HHMMSS.json`: 详细分析结果
- `report_YYYYMMDD_HHMMSS.txt`: 人类可读的评估报告

### 评估指标

#### 多轨迹指标

- **样本级别准确率**: 只要有一条轨迹正确就算样本正确
- **轨迹级别准确率**: 所有轨迹的平均正确率
- **平均轨迹数**: 每个样本的平均轨迹数量

#### 传统指标

- **SQL提取率**: 从模型响应中成功提取SQL的比例
- **SQL有效性**: 提取的SQL语法是否正确
- **执行成功率**: SQL查询执行成功的比例

### 示例输出

```
============================================================
EVALUATION SUMMARY
============================================================
Run ID: 20250710_140000
Total Samples: 100
Total Trajectories: 500
Avg Trajectories per Sample: 5.00
Sample-level Accuracy: 0.8500
Trajectory-level Accuracy: 0.6200
SQL Extraction Rate: 0.9800
Results saved to: ./results
============================================================
```

## 数据准备工具

### 过滤测试数据

使用 `filter_test_data.py` 脚本过滤测试数据：

```bash
python filter_test_data.py \
    --input_file spider_data/test.json \
    --output_file spider_data/filter_test.json \
    --db_path spider_data/test_database
```

该脚本会：
1. 检查每个样本对应的数据库文件是否存在
2. 过滤掉缺少 `schema.sql` 或 `*.sqlite` 文件的样本
3. 保存过滤后的数据集

## 故障排除

### 常见问题

1. **数据库文件未找到**
   - 使用 `filter_test_data.py` 检查数据完整性
   - 确认数据库文件路径正确

2. **SGLang服务器连接失败**
   - 检查服务器URL和端口
   - 确认服务器正在运行

3. **多轨迹评估缓慢**
   - 减少 `--n` 参数值
   - 增加 `--concurrent_requests` 参数
   - 降低 `--temperature` 减少生成时间

### 调试方法

```bash
# 启用DEBUG日志
python main_eval.py --log_level DEBUG [其他参数]

# 小样本测试
python main_eval.py --sample_size 5 --n 2 [其他参数]

# 检查日志文件
tail -f sql_eval.log
```

## 性能优化

### 多轨迹优化

- 根据需求调整轨迹数量（--n）
- 使用适当的温度参数平衡多样性和质量
- 监控总轨迹数量以控制评估时间

### 并发调优

- 根据服务器性能调整 `--concurrent_requests`
- 考虑轨迹数量对并发的影响
- 监控系统资源使用情况

## 项目结构

```
sql_eval/
├── main_eval.py              # 主评估脚本
├── conversation_manager.py   # 对话管理
├── data_preprocess.py        # 数据预处理
├── evaluator.py             # 评估器
├── result_analyzer.py       # 结果分析
├── tool_client.py           # SQL工具客户端
├── filter_test_data.py      # 数据过滤工具
├── requirements.txt         # 依赖列表
└── README.md               # 文档
```
