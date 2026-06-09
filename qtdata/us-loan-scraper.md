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

- 附件文件提取时全量加载文本到内存（单个大文件可能触发 OOM）
- py-sec-edgar 下载 + extract 同时进行时资源竞争

可考虑的优化方向：

- 附件文本提取改用流式/分块读取
- 下载与处理分离：先下载再分批处理
- 使用内存映射文件处理超大型 PDF

## 参考资料

- **数据下载器**：[ryansmccoy/py-sec-edgar](https://github.com/ryansmccoy/py-sec-edgar) — SEC EDGAR 专业下载器，支持 10-K/10-Q/8-K 等表单类型的批量下载与提取
- **SEC EDGAR 主页**：[https://www.sec.gov/edgar](https://www.sec.gov/edgar) — 美国证监会 EDGAR 数据库入口
- **SEC EDGAR 开发者指南**：[https://www.sec.gov/developer](https://www.sec.gov/developer) — SEC 数据访问规范与 Fair Access 指南
