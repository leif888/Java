# 活 spec 模板：让 AI 维护的项目长期管得住

**为什么需要它**：课堂上生成的 spec + 测试如果不维护，几个迭代后就会和代码脱节，退回"没人信的过期文档"。规格驱动开发（SDD）的对策，是把 spec 当成**持续编辑、版本化、能驱动实现的活产物**。

这套模板把"让 coding agent 持续维护 spec"落成三个可直接抄走的工件。课堂只讲结构与纪律，不依赖任何外部项目资料。

## 三个工件

| 工件 | 作用 |
|---|---|
| [AGENTS.md](AGENTS.md) | 规则文件：把"这个项目怎么改"的纪律写死给 agent（issue 分类、分支命名、spec 对账、"完成"的定义）。`AGENTS.md` 是多数 coding agent 都会自动读的约定文件名；用 Claude Code 也可叫 `CLAUDE.md`。|
| [spec/](spec/) | 状态账本：一棵分状态的目录树（`governance/ planned/ implemented/ archived/` + `README.md` 索引），记录项目所有架构与契约的当前状态。|
| [spec/SPEC_TEMPLATE.md](spec/SPEC_TEMPLATE.md) | 单条 spec 的写法模板：状态、契约、验收标准、实现锚点、兼容影响。|

## 核心循环：spec ↔ issue

```
issue 进来
  ├─ 局部 bug（期望已由文档/测试定义） → 直接最小修复 + 补回归 + 同步文档
  └─ 动了新功能/架构/公共接口/兼容线   → 停下，先对齐方案
                                          → 出/改 spec(planned) → 实现分支 → 合并验收
                                          → 落地后 spec 移到 implemented/，更新 README
分不清 → 按"影响设计"处理，先问方案
多个 issue 同根因 → 先抽象成统一方案，别一个个打补丁
```

**"完成"的定义（这套模板的核心铁律）**：一个改动不算完，直到**实现、测试、文档、示例、兼容信息、spec 状态，讲的是同一个故事**。这条把 spec 从"写完就忘"逼成"持续对账"。

## 怎么用

1. 把 `AGENTS.md` 放到仓库根目录，按你的项目改具体条目。
2. 建 `spec/` 目录（四个子目录 + `README.md`），用 `SPEC_TEMPLATE.md` 写第一条 spec。
3. 配合 GitHub 管线（[../05_github_pipeline/](../05_github_pipeline/)）：issue 模板里加一行"关联/影响哪条 spec"，PR 的验收标准写"与 spec 对齐"，CI 做自动门禁。

## spec 目录是公开还是私有？两种都行

- **跟踪在仓库里**（如 Spec Kit）：spec 和代码一起 review、对外可见。
- **独立 / 本地私有账本**：把 `spec/` gitignore、用一个独立的本地 git 管它的历史，规划信息不外推公共仓库（适合不想公开路线图的项目）。

二选一都行，关键是：**它存在、有状态、被持续对账**——而不是散在聊天记录和某个人脑子里。
