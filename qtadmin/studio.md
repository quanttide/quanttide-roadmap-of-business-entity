# 治理可视化路线图

Studio 的目标是让治理思想「以系统化的方式被展示和执行」——展示的不是功能，是治理方法。

## 现状

- 业务屏残留：org（组织）、qtconsult（咨询）、recruitment（招聘）三个历史页面
- router 仍引用已剥离的业务域包（dashboard、think、qtclass），存在死引用风险
- 治理可视化空白：asset、knowl、delib、strategy 均无页面

## 路线

### 近期：清理与 asset 起步

- 清理 router 死引用，确认可构建
- 从 asset 起步：资产治理可视化（archive / status / quality），读取 CLI 数据文件实现双端共享

### 中期：知识链路可视化

- knowl 链路展示：acquire / extract / summary，状态承载（settled / evolving / draft）可视化
- 政策知识视图：政策列表、状态、主题总结的公开化展示

### 远期：支柱页面

- delib 议事屏：议事记录与决策留痕
- strategy 战略屏：方向、张力、假设库

## 原则

- 业务规则走配置化（FileSource），不编入 freezed
- screens 边界清晰：主项目只做路由聚合，领域包承载页面
- AI 产出必须可落地，落不了地的产出是债务，不进入封装
