# 今日学习一角： | 2025.9.6

| 维度           | 关系型数据库（MySQL、PostgreSQL）                   | 大数据 SQL 引擎（Hive、Spark SQL）   | 云原生分析型（BigQuery、Snowflake）               | NoSQL（MongoDB、Redis）          |
| ------------ | ------------------------------------------ | ---------------------------- | ---------------------------------------- | ----------------------------- |
| **数据规模**     | GB ~ TB                                    | TB ~ PB                      | TB ~ PB                                  | KB ~ TB（取决于类型）                |
| **事务支持**     | 强（ACID）                                    | 弱（主要分析，不强调事务）                | 弱（分析型）                                   | 一般不支持复杂事务                     |
| **延迟**       | 毫秒级                                        | 秒 ~ 分钟级                      | 秒级                                       | 毫秒级                           |
| **部署方式**     | 本地 / 云                                     | 集群                           | 云托管                                      | 本地 / 云                        |
| **SQL 标准支持** | 较高（PostgreSQL 最接近标准）                       | 部分支持（Hive SQL、Spark SQL 有差异） | 高 + 扩展功能                                 | 不一定支持 SQL                     |
| **典型场景**     | 业务系统、ERP、CRM                               | 数据仓库、离线分析                    | BI 报表、交互式分析                              | 缓存、文档存储、实时数据                  |
| **代表**       | MySQL、PostgreSQL、Oracle、SQL Server、MariaDB | Hive、Spark SQL、Presto、Trino  | BigQuery（Google）、Snowflake、Redshift（AWS） | MongoDB、Redis、Cassandra、HBase |
