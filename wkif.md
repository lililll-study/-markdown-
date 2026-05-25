# BRS（立减金预算推荐服务）设计文档
## 一、项目概述
### 1.1 项目背景

<span style="font-size: 19px;">本项目为**支付立减金预算推荐服务**（Business Recommendation Service, BRS），基于 **LSTM 时序预测算法**开发的智能预算决策系统。该系统解决了支付营销活动中传统预算管理依赖人工经验、调整滞后、易出现超支风险的问题。

<span style="font-size: 21px;">**业务流程：**
- <span style="font-size: 19px;"> 从 ClickHouse 采集历史消费数据，推送到 Redis 队列
- <span style="font-size: 19px;"> Python 算法服务消费队列数据进行预测
- <span style="font-size: 19px;"> 预测结果返回 Redis，Java 拉取并写入 ClickHouse
- <span style="font-size: 19px;"> 支持双模式预测：日常小时级 / 大促分钟级
---

## 二、技术架构

### 2.1 业务流程图
<div style="text-align: center;">
    <img src="业务流程图 (1).png" alt="业务流程图" width="450">
</div>

---

## 三、核心数据流详解

### 3.1 完整数据处理流程

<span style="font-size: 21px;">**步骤 1: XXL-Job 触发任务**
任务 784: pushAllBusiFavorBudget2RedisQueue
参数示例: {"batchIds": ["ACT349DSC02573596"], "dataDay": 600}
**详细说明：**
<div style="text-align: justify; text-justify: inter-ideograph; max-width: 800px; margin: 0 auto;"><span style="font-size: 21px;">
    整个流程的起点是 XXL-Job 定时任务。有两个核心任务：任务 784 负责‘推’，任务 781 负责‘拉’。任务 784 的入参是一个 JSON，比如 {"batchIds": ["ACT349D"], "dataDay": 600}。batchIds 指定要预测哪些活动，dataDay 控制取最近多少天的历史数据。
    这个参数是灵活的：如果不传 batchIds，就查所有活动；dataDay 可以根据需要调大调小。我这样设计是为了方便手动调试和灰度测试。
</div>

---

<span style="font-size: 21px;">**步骤 2: Java 从 ClickHouse 查询原始数据**
SQL: selectThirdCodeDetailsByWinDay()
来源表: tb_third_code_detail (ClickHouse)
过滤条件: status='running', partition_date > today()-dataDay
输出: List： thirdcodedetail
**详细说明：**
<div style="text-align: justify; text-justify: inter-ideograph; max-width: 800px; margin: 0 auto;"><span style="font-size: 21px;">
核心过滤条件是status = 'running'：只查还在运行中的活动。partition_date > today() - dataDay：只查最近 dataDay 天的分区数据。SQL 会按天（或按小时）把数据聚合成一条条记录，返回给 Java 的是一个 List ThirdCodeDetail。这一步相当于把数据库里的原始行记录，转成了 Java 能用的对象列表。</div>

---

<span style="font-size: 21px;">**步骤 3: 数据预处理 & 构建 DTO**
字段映射: 
batch_id → batchId
type → productType
...
输出: AlgorithmReqDTO → JSON
**详细说明：**
<div style="text-align: justify; text-justify: inter-ideograph; max-width: 800px; margin: 0 auto;"><span style="font-size: 21px;">
原始数据里的字段名是下划线风格（比如 batch_id），但算法那边希望用驼峰（batchId）。所以我做了一个字段映射，把 ThirdCodeDetail 转成 AlgorithmReqDTO。同时，这一步会根据 dataDay 和 fmt 参数，把时间序列数据拆成 hourList 或 dayList。比如 dataDay 是 600，fmt 是 yyyyMMdd，那就按天聚合；如果 fmt 是 yyyyMMddHH，就按小时。最后把这个 DTO 序列化成 JSON 字符串，准备推到 Redis。</div>

---

<span style="font-size: 21px;">**步骤 4: 推送到 Redis 队列**
队列名: LJJ:BDGT:Q:P (处理队列)
操作: rpush()
格式: JSON String
**详细说明：**
<div style="text-align: justify; text-justify: inter-ideograph; max-width: 800px; margin: 0 auto;"><span style="font-size: 21px;">
把上一步生成的 JSON 字符串，用 rpush 推到 Redis 的一个队列里。队列名是固定的：LJJ:BDGT:Q:P（P 代表 Processing，处理队列）。用 Redis 做队列的好处是解耦：Java 只管往里塞，Python 只管往外取，两边不用互相等。而且 Redis 本身支持队列的原子操作，不用担心消息丢失或重复消费的问题（配合业务幂等处理）</div>

---

<span style="font-size: 21px;">**步骤 5: Python 消费并处理**
main_process_model(json_string) → prepare_ts_model_data()  # 数据预处理
- <span style="font-size: 18px;">按天/小时聚合
- <span style="font-size: 18px;">数据增强 (噪声扩增)
- <span style="font-size: 18px;">添加 lag 特征 [1,2,3,4,5,6,7,14]
- <span style="font-size: 18px;">归一化 (MinMaxScaler)

<span style="font-size: 21px;">模式判断：
- <span style="font-size: 18px;">budget_model_type_id == 1 (日常模式)

- <span style="font-size: 18px;">budget_model_type_id == 2 (异常模式)
format_ouput_strategy()  # 业务策略输出
**详细说明：**
<div style="text-align: justify; text-justify: inter-ideograph; max-width: 800px; margin: 0 auto;"><span style="font-size: 21px;">
先把原始数据按时间对齐，缺失的补 0。
做数据增强：比如加一点随机噪声，让模型泛化能力更好。
添加 lag 特征：用过去第 1、2、3、4、5、6、7、14 天的值作为特征，帮助模型捕捉周期性。
归一化：用 MinMaxScaler 把数据缩放到 [0,1] 区间。
id == 1（日常模式）：走 LSTM 模型预测，结果是连续的数值。
id == 2（异常模式）：不走模型，走 format_output_strategy() 业务策略。比如数据不足或者波动太大时，直接返回兜底建议。</div>

---

<span style="font-size: 21px;">**步骤 6: 结果写入 Redis 结果队列**
队列名: LJJ:BDGT:Q:R (结果队列) 
操作: lpush()
格式: Result List (JSON)
**详细说明：**
<div style="text-align: justify; text-justify: inter-ideograph; max-width: 800px; margin: 0 auto;"><span style="font-size: 21px;">
Python 把上一步生成的 JSON 数组，用 lpush 推到另一个 Redis 队列：LJJ:BDGT:Q:R（R 代表 Result，结果队列）。</div>

---

<span style="font-size: 21px;">**步骤 7: Java 消费结果并持久化**
任务 781: cacheBusiFavorBudgetByRedisJob
参数: {"dataLimit": 500}
操作:
<span style="font-size: 18px;">1. blpop() 从结果队列取出结果
2. 解析为 AlgorithmResDTO
3. 批量插入 ClickHouse:INSERT INTO tb_busi_favor_budget_test
**详细说明：**
<div style="text-align: justify; text-justify: inter-ideograph; max-width: 800px; margin: 0 auto;"><span style="font-size: 21px;">
任务 781 cacheBusiFavorBudgetByRedisJob 专门负责消费结果队列。它会用 blpop 阻塞式地从结果队列里取数据，如果没有结果就等着，有结果就立刻处理。
取到的结果是一个 JSON 数组，我会把它解析成 AlgorithmResDTO，然后调用 ClickHouse 的 Mapper 做批量插入。表是 tb_busi_favor_budget_test，字段包括批次 ID、预测值、版本号、时间戳等。
批量插入用的是 MyBatis 的 foreach，一次能插几百条，性能比逐条插好很多。</div>

---
