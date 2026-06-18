# 把 Azure SQL 数据库作为知识库连接到 Copilot Studio

> 让 Copilot Studio 智能体直接「读懂」一个 Azure SQL 数据库里的结构化数据，用自然语言就能查询。示例做的是一个**房地产房源智能体**：经纪人用大白话提问，智能体从 SQL 里查出符合条件的房源并作答。

本项目是对 Matthew Devaney 教程 [Copilot Studio – Connect An Azure SQL Database As Knowledge](https://www.matthewdevaney.com/copilot-studio-connect-an-azure-sql-database-as-knowledge/) 的**中文重制版**，把整套架构、从零搭建步骤、鉴权配置、知识增强技巧（同义词/描述/数据词汇表）以及生产环境的已知限制，完整地用中文整理出来，便于学习、复现与面试讲解。

---

## 一、这个项目解决什么问题

企业的核心数据大多躺在关系型数据库里（订单、库存、客户、房源……）。要让业务人员查这些数据，传统做法是写 SQL 或做 BI 报表，有门槛、不灵活。

本项目把一个 **Azure SQL 数据库**直接挂成 Copilot Studio 智能体的**知识源**，于是：

- 经纪人问「**有没有 3 间卧室的房子？**」→ 智能体自动从 `realestate_listings` 表里筛 `bed = 3` 的房源；
- 问「**给我看特拉华州（Delaware）的房子**」→ 通过**数据词汇表**把「Delaware」映射到州代码 `DE`；
- 问「**MLS 编号 5 的房源信息**」→ 通过**同义词**把「MLS 编号」映射到 `listing_id` 列。

全程不写一行 SQL，自然语言直达数据库。

---

## 二、核心思路与关键设计

### 1. 用 Service Principal（服务主体）做鉴权 —— 最重要的设计
连接 Azure SQL 时，鉴权方式推荐用 **Service Principal（Microsoft Entra ID 应用注册）**，而不是每个用户各自的账号。

- **为什么**：服务主体的连接可以**被所有智能体用户共享**，不需要每个使用者单独建立 SQL 连接、单独授权；
- 在数据库侧把这个服务主体建成**只读外部用户（db_datareader）**，做到「**最小权限**」——智能体只能读、不能改。

### 2. 三层「语义增强」让模型听懂业务话
光连上表还不够，列名是 `listing_id`、状态值是 `DE`，但用户说的是「MLS 编号」「特拉华州」。靠三样东西把**业务语言**翻译成**数据库语言**：

| 机制 | 作用 | 例子 |
|------|------|------|
| **同义词 Synonyms** | 给列起别名 | `listing_id` ← MLS 编号 / Property ID / Reference Number |
| **描述 Descriptions** | 给列补充语义 | 给 `bed`、`bath` 等列写清楚含义 |
| **数据词汇表 Glossary** | 给「值」补充说明 | `DE` ↔ Delaware（各州全称 ↔ 两位代码）|

详见 [`docs/02-知识增强配置.md`](docs/02-知识增强配置.md)。

---

## 三、整体架构

```
经纪人（自然语言提问）
   │
   ▼
Copilot Studio 智能体
   │  知识源：Azure SQL（只读）
   │  语义增强：同义词 / 列描述 / 数据词汇表
   ▼
Azure SQL 数据库
   ├── realestate_listings  房源表（30 条示例）
   └── realestate_agents    经纪人表（6 条示例，被房源表外键引用）

鉴权链路：
Copilot Studio ──(Service Principal: Tenant + Client ID + Secret)──► Azure SQL
                  数据库侧：该服务主体 = 只读外部用户 db_datareader
网络：SQL Server 开启「允许 Azure 服务访问」，本地用 SSMS 连接需加防火墙 IP
```

---

## 四、目录结构

```
copilot-studio-azure-sql-knowledge/
├── README.md                       # 本文件：项目总览
├── sql/
│   ├── Create-Azure-SQL-Tables.sql # 建表 + 30 条房源/6 个经纪人示例数据（原始脚本）
│   └── 表结构说明.md                # 两张表的中文字段说明与关系
├── docs/
│   ├── 01-搭建步骤详解.md           # 从零到可用的完整 8 步
│   ├── 02-知识增强配置.md           # 同义词 / 描述 / 数据词汇表怎么配
│   ├── 03-已知限制与排查.md         # 行数上限等生产环境必须知道的坑
│   └── 04-面试讲解要点.md           # 讲解大纲、亮点、问答预案
└── data/
    └── 示例问题.md                  # 可让智能体回答的示例提问
```

---

## 五、快速开始

> 本项目是一份「把 Azure SQL 接入 Copilot Studio」的完整方案，核心产物是 SQL 脚本 + 配置步骤，没有需要编译的代码。

**前置条件**
- 一个 **Azure 订阅**（创建 Azure SQL 数据库）；
- **SQL Server Management Studio (SSMS)**（本地建表、建外部用户）；
- 一个 **Copilot Studio** 环境；
- 在 Microsoft Entra ID 里有**应用注册**权限（创建 Service Principal）。

**步骤概览**（详见 [`docs/01-搭建步骤详解.md`](docs/01-搭建步骤详解.md)）
1. 创建 Azure SQL 数据库与服务器；
2. 配置网络：开启「允许 Azure 服务访问」；
3. 装 SSMS，连上数据库；
4. 跑 [`sql/Create-Azure-SQL-Tables.sql`](sql/Create-Azure-SQL-Tables.sql) 建两张表并插入示例数据；
5. 在 Azure 注册应用，拿到 **Tenant ID / Client ID / Client Secret**；
6. 在 SSMS 里把该应用建成**只读外部用户**（db_datareader）；
7. 在 Copilot Studio 用 **Azure SQL** 知识源 + **Service Principal** 鉴权接入两张表；
8. 配置同义词 / 列描述 / 数据词汇表，提升理解准确度。

**实测提问** 见 [`data/示例问题.md`](data/示例问题.md)；**面试讲解** 见 [`docs/04-面试讲解要点.md`](docs/04-面试讲解要点.md)。

---

## 六、技术栈

- **Microsoft Copilot Studio**（智能体 + Azure SQL 知识连接器 + 语义增强）
- **Azure SQL Database**（结构化数据存储）
- **Microsoft Entra ID / Service Principal**（应用注册做共享鉴权）
- **SQL Server Management Studio (SSMS)**（建表、建外部用户）

---

## 七、致谢与说明

- 原始教程与 SQL 脚本作者：**Matthew Devaney** — [博客原文](https://www.matthewdevaney.com/copilot-studio-connect-an-azure-sql-database-as-knowledge/)、[视频](https://youtu.be/ao640wcA1iQ)、[原始仓库](https://github.com/matthewdevaney/Copilot-Studio-Tutorials)。
- 本仓库为**个人学习用中文重制版**，`sql/` 下脚本来自原作者仓库，版权归原作者所有，仅用于学习与演示。示例数据中的姓名、电话、邮箱均为虚构测试数据。
