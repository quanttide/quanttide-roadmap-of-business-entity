# 美国贷款合同爬虫蓝图

## 模式识别

难点：识别 Credit Agreement 文件

当前问题：所有匹配 loan/credit 关键词的 8-K 附件都被处理，未做 Credit Agreement 分类。

需要挖掘的识别特征：

| 特征维度 | 示例 |
|----------|------|
| 文件标题 | "CREDIT AGREEMENT" / "LOAN AGREEMENT" |
| 合同方 | "Borrower" 与 "Lender" 同时出现 |
| 章节结构 | Article I: Definitions, Article II: The Loans |
| 典型条款 | "Interest Rate", "Maturity Date", "Events of Default" |
| 排除特征 | 不含 "Indenture"、"Notes" 等债券特征词 |

建议改造：增加前置分类判断——是 Credit Agreement 才进入后续提取流程，否则跳过。

## 性能优化

难点：批量下载爆内存

当前措施：

- V18 版本已实现流式写入 CSV（非全量内存缓存）
- 每个 CIK 处理完释放内存 + 每 10 轮 GC.collect()

仍存在的风险：

- **全量索引 OOM**：初始下载索引一次性拉取整个 CIK 的历史目录，历史较久的 CIK 包含海量提交记录，可能触发 OOM
- **大文本内存穿透**：附件文件提取时全量加载文本到内存，单个大文件（含长表格或扫描件转录）可能触发 OOM
- **I/O 与 CPU 资源竞争**：py-sec-edgar 下载与 extract 同时进行，峰值内存叠加导致进程崩溃

已确认的优化方案（经过代码研究和生产验证）：

- **流式提取**：附件文本提取改用按行/分块读取（`yield` 生成器），匹配完成后即释放内存块
- **内存映射**：针对超过 50MB 的超大附件，使用 `mmap` 模块实现零拷贝内存访问
- **索引流式构建**：采用分页或按年滚动查询，替代全量一次性拉取
- **生产者-消费者解耦**：[验证通过] 下载与提取彻底分离——Producer 负责限速下载（10 req/s，满足 SEC Fair Access 要求）并落盘，Consumer 独立进程从任务队列读取文件进行提取，消除资源竞争
- **Phase 实施策略**：Phase 1 改造文件读取（最小改动），Phase 2 引入 Producer-Consumer 架构，Phase 3 增加内存和速率监控探针

## 参考资料

- **数据下载器**：[ryansmccoy/py-sec-edgar](https://github.com/ryansmccoy/py-sec-edgar) — SEC EDGAR 专业下载器，支持 10-K/10-Q/8-K 等表单类型的批量下载与提取
- **SEC EDGAR 主页**：[https://www.sec.gov/edgar](https://www.sec.gov/edgar) — 美国证监会 EDGAR 数据库入口
- **SEC EDGAR 开发者指南**：[https://www.sec.gov/developer](https://www.sec.gov/developer) — SEC 数据访问规范与 Fair Access 指南
