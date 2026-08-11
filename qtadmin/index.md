# 量潮管理后台路线图

量潮管理后台（qtadmin）是企业治理思想和制度的平台化——展示的不是功能，是治理方法。与量潮云（生产工具）彻底分开，视角首先是秘书处的管理视角。

## 阶段位置

分化已走完：业务域（qtclass、qtcloud、qtconsult、qtdata、qtrecurit）全部归位各平台仓库，CLI 收敛为五个职能域命令。当前处于连接期起步——补齐三支柱载体与治理可视化。

## 三支柱现状

| 支柱 | 现状 | 载体 |
|------|------|------|
| asset 资产管理 | ✅ CLI 已落地 | archive、status、quality |
| delib 议事设计 | ❌ 零实现 | 制度素材完整（见 [delib.md](./delib.md)） |
| strategy 战略发现 | ❌ 零实现 | 方法论已沉淀（见 [strategy.md](./strategy.md)） |

## 版本节奏

- CLI v0.0.17、Studio v0.1.3，首个 tag 待发
- Provider 维护态，随数据契约恢复演进
- 各支柱按里程碑推进，见分文件路线

## 文件索引

- [asset.md](./asset.md)：资产管理支柱——三层建模、workspace、组织分析工具
- [delib.md](./delib.md)：议事支柱——议事记录与决策留痕最小闭环
- [strategy.md](./strategy.md)：战略支柱——假设库、事实审计与反事实推演
- [studio.md](./studio.md)：治理可视化——清理残留、双端共享、支柱页面
- [foundation.md](./foundation.md)：基础能力——职能域下沉、数据契约、封装边界
