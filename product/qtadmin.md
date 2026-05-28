# QtCloud HR — 集成蓝图

> 目标：将 `quanttide-hr-toolkit`的`packages/fastapi` 库集成到 `qtadmin`的 `provider` 应用中。

---

## 当前状态

```
packages/fastapi/
    └── src/fastapi_quanttide_hr/      ← 纯库，无外部依赖
        ├── models/                      Recruitment, Talent
        ├── schemas/                     Pydantic 模型
        ├── routers/                     REST 端点
        └── services/                    Pipeline 业务逻辑
```

库不管理数据库、不定义身份、不绑定品牌。应用层负责组装。

---

## 集成架构

```
qtadmin-provider (FastAPI)
  ├── override dependency: get_db        → 应用管理的数据库 session
  ├── include_router: recruitments.router → /recruitments/*
  ├── include_router: pipeline.router    → /pipeline
  │
  ├── qtadmin-auth  (UserProfile)        ── Talent.email 关联
  ├── qtadmin-org   (Position)           ── 后续扩展
  └── 自有数据库     (SQL/PostgreSQL)     ── 建表 + session
```

---

## 集成步骤

### Step 1：库纳入 qtadmin 依赖

```toml
# qtadmin/src/provider/pyproject.toml
dependencies = [
    "fastapi_quanttide_hr",
]
```

### Step 2：在 qtadmin 中挂载路由

```python
from fastapi import FastAPI
from fastapi_quanttide_hr.routers import pipeline, recruitments
from fastapi_quanttide_hr.database import Base, get_db as lib_get_db

# 应用创建 engine + session
engine = create_engine("sqlite:///qtadmin.db")
SessionLocal = sessionmaker(bind=engine)
Base.metadata.create_all(bind=engine)

def app_get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# 组装 app
app = FastAPI()
app.dependency_overrides[lib_get_db] = app_get_db
app.include_router(recruitments.router)
app.include_router(pipeline.router)
```

---

## 🔴 核心工作

### 与 HR 讨论，根据实际工作流设计 TalentStatus

> **优先级**：最高。当前 `TalentStatus` 的 8 个值是假设的，必须经过 HR 验证。

**现状**：
- 库中定义了 8 个固定 status：`new → contacted → exam_sent → exam_received → evaluating → interview → offer → closed`
- 这些值来自 hr@quanttide.com 邮件分析，但未与 HR 团队确认

**需要确认的问题**：

1. **现状 mapping**：HR 当前实际把候选人分哪几类？用的是什么术语？
2. **缺失状态**：有没有当前枚举没覆盖的状态（如 "待入职"、"已入职"、"放弃流程"）？
3. **多余状态**：有没有从不使用的状态？
4. **流转规则**：哪些流转是允许的？哪些是禁止的？（当前 `STATUS_TRANSITIONS` 是技术假设）
5. **状态属性**：每个状态是否需要额外的业务属性（如 "exam_sent" 需要记录发送时间、截止日期）？

**交付物**：
- 一份 HR 确认的 `TalentStatus` 定义
- 对应的 `STATUS_TRANSITIONS` 流转规则
- 如果 8 个值不满足，库中需要支持自定义/可配置的 status（或直接替换枚举）

**预计耗时**：1 次 HR 访谈 + 半天的模型调整。

---

## ✅ 验收标准

**核心目标**：使用该系统可以稳定地提供 HR 工作进度给秘书处。

具体含义：

| 维度 | 标准 |
|------|------|
| **数据准确** | Pipeline 看板数据与 HR 实际进度一致，无遗漏、无延迟 |
| **可读性** | 秘书处人员打开系统或收到报告，能一眼看清当前所有候选人进度分布 |
| **稳定性** | 系统持续可用，数据不丢失、不损坏 |
| **可追溯** | 每个 Talent 的流转历史可查（当前谁在哪个阶段，走了哪些路径） |
| **交付形式** | 至少满足以下一种形式：<br>1. 秘书处可访问的只读看板（Web UI）<br>2. 定期推送的进度报告（邮件/飞书消息）
