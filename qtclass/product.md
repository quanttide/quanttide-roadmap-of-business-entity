# 量潮课堂产品路线图

当前主线：qtclass studio 升级 0.2.0，将学习系统（实训旅程）与现有课程系统并列整合。

## 背景

studio 现有课程系统承载内容学习（浏览课程 → 播放 → 进度 → 立项）；实训旅程（报名 → 问卷 → 领任务 → 交付 → 评审 → 入册）已在实验室工作台完成河床参考实现，免费学员资格获取流程与付费培养流程已在手册定型。0.2.0 把学习系统作为独立子系统与课程系统并列，学员端形成「课程 + 实训」双入口。

## 命名

- 新领域命名为 journey（学员旅程）：provider 中 learn 包已被学习云代理占用，为避免碰撞，服务端领域与 API 以 journey 命名；客户端入口 UI 叫「实训」
- 旅程状态机与实验室 store 语义一致：applied → survey_done → invited → task_assigned → task_submitted → reviewing → graded → enrolled

## 架构

```
Studio (Flutter 0.2.0)                     Provider (Go)
┌─────────────────────────┐               ┌──────────────────────────┐
│ 课程系统（现有，不动）      │               │ course：课程数据（现有）     │
│  首页/列表/播放/立项       │               │ learn：学习云代理（现有）    │
│─────────────────────────│               │──────────────────────────│
│ 实训系统（新增 journey 域）│───API───▶     │ journey：旅程状态机（新增）  │
│  旅程时间线 + 节点动作面板  │               │  SQLite 持久化（新增）      │
└─────────────────────────┘               └──────────────────────────┘
```

- 状态迁移全部是系统内事件：每个迁移 = 一个 API 端点，服务端守卫
- 状态机即数据：Go 侧状态表 + Flutter 侧 STAGE_DETAIL/STATUS_STAGE 映射表，改流程 = 改表
- 身份绑定 studio 登录账号（实验室的姓名即身份由账号体系替代，姓名仅作显示）

## 数据模型

| 表 | 字段要点 |
|----|---------|
| applications | account_id、track（意向训练营）、status、survey_json、applied_at、survey_at |
| tasks | title、type、deadline、track——运营录入真实业务任务 |
| deliveries | application_id、task_id、file、submitted_at |
| reviews | application_id、stage（初审/终审）、verdict（通过/打回）、comment、reviewed_at——结论必留痕 |

## API

```
GET  /journey                    # 当前旅程状态（无记录=未报名）
POST /journey/apply              # 报名 {track}
POST /journey/survey             # 提交问卷 → survey_done，自动 grant → invited
GET  /journey/tasks              # 任务清单
POST /journey/tasks/{id}/assign  # 领任务 → task_assigned
POST /journey/deliveries         # 提交交付物（multipart）→ task_submitted
GET  /journey/reviews            # 评审结论（学员只读）
```

评审录入（初审/终审、打回附原因）为受保护端点，属课堂运营侧；管理界面不在本期。

## Studio 端

- 顶层导航并列：课程（现有全部不动）｜实训（新增），两系统互不侵入
- 新增 journey_screen（旅程时间线 + 当前节点动作面板，平移实验室信息架构）、survey_screen（五道动机题、必填校验）、task_list_screen、delivery_screen
- 评审状态在旅程面板只读展示（通过/打回附原因）
- 服务层新增 JourneyApi（照 LearnApi 模式：QTCLASS_API_BASE_URL + token），模型 JourneyState/Task/Delivery/Review

## 里程碑

1. M1 provider journey 域：状态机迁移 + SQLite + 端点 + Go 测试（全旅程迁移链、守卫、留痕）——约占四成，正确性核心
2. M2 studio 实训骨架：导航并列 + journey_screen（时间线/报名/问卷）接 API + widget 测试——约三成
3. M3 任务与交付：task_list/delivery 接通 + 评审只读——约一成半
4. M4 发布：provider/v0.2.0 → studio/v0.2.0 走发布流程，主仓库指针同步——约一成半

## 版本

- provider：v0.2.0-alpha.1 → v0.2.0（新增 journey 域 + SQLite 持久化）
- studio：v0.2.0（新子系统并列整合，minor 升级）
- site / root：不动；site 学习页为公开内容，与学员端旅程互补
