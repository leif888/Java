# Spec 索引（模板）

> Updated: <YYYY-MM-DD>

## 用途

`spec/` 是本项目架构与契约的**状态账本**：所有"对外行为、接口、架构边界"的设计与落地状态都记在这里，由 coding agent 在每次相关改动时持续对账。它不是一篇大文档，而是一棵分状态的目录树。

## 四类文档

- `governance/`：**持续生效**的规则（测试口径、发布 / 版本、兼容策略、架构边界），与 planned / implemented 状态无关。
- `planned/`：已设计、尚未完全落地的契约或未来工作，按 `<feature-domain>/` 分。
- `implemented/`：已落地的契约 + 指向实现的锚点（文件 / 函数 / 测试），作参考架构。
- `archived/`：搁置 / 废弃的设计，留下决策、不再是实现入口。

## 目录布局

```text
spec/
├── README.md
├── governance/
│   └── <RULE>_SPEC.md            # 如 TESTING_GOVERNANCE / RELEASE_VERSIONING / COMPAT
├── planned/
│   └── <feature-domain>/
│       └── <FEATURE>_SPEC.md
├── implemented/
│   └── <feature-domain>/
│       └── <FEATURE>_SPEC.md
└── archived/
    ├── deferred/
    └── deprecated/
```

## 生命周期

1. 新的设计型改动 → 在 `planned/<feature-domain>/` 写一条 spec（用 [SPEC_TEMPLATE.md](SPEC_TEMPLATE.md)）。
2. 代码 / 测试证明它落地 → 把 spec 移到 `implemented/<feature-domain>/`，补实现锚点，**同一工作项里更新本 README**。
3. 只部分落地 → 留在 `planned/`，记录已实现 / 推迟 / 剩余验收标准。
4. 不再需要 → 移到 `archived/`，保留决策摘要。

## 当前 spec 清单（示例，按实际维护）

| 路径 | 状态 | 一句话 |
|---|---|---|
| `governance/TESTING_GOVERNANCE_SPEC.md` | governance | 测试口径与门禁 |
| `planned/<domain>/<FEATURE>_SPEC.md` | planned | <设计中> |
| `implemented/<domain>/<FEATURE>_SPEC.md` | implemented | <已落地，锚点见 spec 内> |

> 提示：`spec/` 可以跟踪在主仓库里（和代码一起 review），也可以做成 gitignore 的独立 / 私有本地账本（规划信息不外推）。两种都行，关键是它存在、有状态、被持续对账。
