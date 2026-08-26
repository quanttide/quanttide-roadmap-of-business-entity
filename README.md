# 量潮科技工作蓝图

量潮科技 roadmap（工作蓝图）——按真实组织镜像的结构组织，跨领域主题在公司级层定义全景，各领域只维护自己的视角。

## 结构

```
roadmap/
├── index.md          # 蓝图入口（战略定位/发展策略/当前重点）
├── intro/            # 公司级层：跨领域主题（全景单一来源）
│   └── training-base.md   # 实训基地（串联课堂/招聘的枢纽——双路径全景）
├── qtclass/          # 课堂领域
│   ├── index.md            # 领域整体路线图
│   └── training-base.md    # 实训基地课堂侧（路径 A）
├── qtrecurit/        # 招聘领域
│   ├── index.md            # 领域整体路线图
│   └── training-base.md    # 实训基地招聘侧（路径 B）
├── org / qtadmin / qtcloud / qtdata / strategy
└── AGENTS.md         # 维护规范
```

## 组织原则

1. **结构即真实**：roadmap 目录镜像组织真实结构——跨领域实体不塞进单一领域
2. **intro = 公司级层**：跨领域主题（实训基地等）全景放 intro/——公共事实只写一次
3. **领域只讲自己的侧面**：qtclass/training-base.md、qtrecurit/training-base.md 与 intro 同名同构——交叉引用不重复
4. **index 与专项分离**：index.md = 领域整体路线图；training-base.md = 领域内专项——专项不混进整体
