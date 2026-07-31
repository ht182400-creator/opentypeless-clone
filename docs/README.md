# OpenTypeless 文档中心（导航索引）

> 作用：把分散在 `docs/` 下的 20+ 份设计规格（spec / plan / notes）、产品愿景与分析文档串成一张可导航的地图。
> 维护：新增设计文档后，请在对应分组下补充条目并保持一致命名（`YYYY-MM-DD-主题.md`）。
> 最后更新：2026-07-31

---

## 0. 先读这两份（全局地图）

| 文档 | 说明 |
|---|---|
| [OpenTypeless_项目深度分析.md](./OpenTypeless_项目深度分析.md) | **文档专家视角的全局分析**：需求 / 整体框架 / 微观实现 / 使用 / 测试 / 评价建议，含代码引用。新人第一站。 |
| [../VISION.md](../VISION.md) | 产品愿景叙事（Typeless 开源版的定位、价值主张、自托管 vs 托管云）。 |
| [../handoff.md](../handoff.md) | 项目交接说明（根目录，含当前阶段状态与待办上下文）。 |

---

## 1. 架构与设计原则

| 文档 | 说明 |
|---|---|
| [2026-05-21-optimization-design.md](./2026-05-21-optimization-design.md) | 早期性能/架构优化设计。 |
| [2026-05-22-optimization-plan.md](./2026-05-22-optimization-plan.md) | 优化落地计划。 |
| [app-icon-sources.md](./app-icon-sources.md) | 应用图标素材来源与规范。 |
| [release-signing.md](./release-signing.md) | 发布签名（代码签名 / 公证）流程。 |

---

## 2. 国际化（i18n）

| 文档 | 说明 |
|---|---|
| [2026-05-22-i18n-design.md](./2026-05-22-i18n-design.md) | i18n 设计方案（react-i18next、10 语言策略）。 |
| [2026-05-22-i18n-plan.md](./2026-05-22-i18n-plan.md) | i18n 落地计划。 |

---

## 3. 发布与平台稳定化

| 文档 | 说明 |
|---|---|
| [2026-06-13-platform-release-stabilization-spec.md](./2026-06-13-platform-release-stabilization-spec.md) | 跨平台发布稳定化规格（Win/mac/Linux 构建与已知坑）。 |
| [2026-07-08-release-v1.1.48-notes.md](./2026-07-08-release-v1.1.48-notes.md) | v1.1.48 发布说明。 |
| [2026-07-11-typeless-feature-completion-p0-p1-spec.md](./2026-07-11-typeless-feature-completion-p0-p1-spec.md) | 功能补全 P0/P1 优先级规格。 |

---

## 4. 云端 / 账户 / 商业化

| 文档 | 说明 |
|---|---|
| [2026-06-20-appsumo-lifetime-cloud-client-spec.md](./2026-06-20-appsumo-lifetime-cloud-client-spec.md) | AppSumo 终身云客户端接入规格（订阅校验、配额、备份恢复）。 |

---

## 5. Ask 对话功能

| 文档 | 说明 |
|---|---|
| [2026-06-29-ask-anything-reliability-spec.md](./2026-06-29-ask-anything-reliability-spec.md) | Ask 可靠性规格（流式、重试、失败兜底）。 |
| [2026-07-07-ask-anything-floating-note-ux-spec.md](./2026-07-07-ask-anything-floating-note-ux-spec.md) | Ask 浮窗便签 UX 规格。 |

---

## 6. 写作场景 / 本地优先

| 文档 | 说明 |
|---|---|
| [2026-06-30-local-first-custom-scenes-spec.md](./2026-06-30-local-first-custom-scenes-spec.md) | 本地优先的自定义写作场景规格（语气/长度/格式预设）。 |

---

## 7. 开源（OpenLess）专项分析

| 文档 | 说明 |
|---|---|
| [2026-07-06-openless-gap-analysis-spec.md](./2026-07-06-openless-gap-analysis-spec.md) | 开源版与闭源 Typeless 的能力差距分析。 |
| [2026-07-06-openless-deep-dive-implementation-spec.md](./2026-07-06-openless-deep-dive-implementation-spec.md) | OpenLess 深度实现规格。 |
| [2026-07-06-openless-followup-completion-spec.md](./2026-07-06-openless-followup-completion-spec.md) | OpenLess 后续补全规格。 |
| [2026-07-08-openless-shandianshuo-typeless-gap-spec.md](./2026-07-08-openless-shandianshuo-typeless-gap-spec.md) | 闪电鼠（shandianshuo）渠道的 Typeless 差距分析。 |
| [2026-07-08-gap-closure-implementation-spec.md](./2026-07-08-gap-closure-implementation-spec.md) | 差距闭合实现规格。 |

---

## 8. 竞品与路线图

| 文档 | 说明 |
|---|---|
| [2026-06-27-typeless-competitive-roadmap-spec.md](./2026-06-27-typeless-competitive-roadmap-spec.md) | 竞品对比与产品路线图。 |

---

## 9. 平台专项：Windows 输出

| 文档 | 说明 |
|---|---|
| [2026-07-17-windows-modifier-safe-output-spec.md](./2026-07-17-windows-modifier-safe-output-spec.md) | Windows 修饰键安全输出规格（SendInput 规避 Ctrl/Alt/Win 卡死）。 |

---

## 10. UX / 录音限制

| 文档 | 说明 |
|---|---|
| [2026-07-23-desktop-copy-recording-limit-ux-spec.md](./2026-07-23-desktop-copy-recording-limit-ux-spec.md) | 桌面端录音时长限制 UX 规格。 |

---

## 11. 流程治理（PR / Issue）

| 文档 | 说明 |
|---|---|
| [2026-06-16-open-pr-triage-spec.md](./2026-06-16-open-pr-triage-spec.md) | 开源 PR 分流规范。 |
| [2026-06-25-latest-pr-review-spec.md](./2026-06-25-latest-pr-review-spec.md) | 近期 PR 评审记录。 |
| [2026-06-26-open-issues-remediation-spec.md](./2026-06-26-open-issues-remediation-spec.md) | 开源 Issue 修复处置规格。 |

---

## 12. superpowers 子目录

### 12.1 plans（落地计划）

- [2026-06-01-custom-local-stt.md](./superpowers/plans/2026-06-01-custom-local-stt.md)
- [2026-06-03-cloud-error-capsule-visibility.md](./superpowers/plans/2026-06-03-cloud-error-capsule-visibility.md)
- [2026-07-12-desktop-account-context-ui-refinement.md](./superpowers/plans/2026-07-12-desktop-account-context-ui-refinement.md)
- [2026-07-12-desktop-settings-simplification.md](./superpowers/plans/2026-07-12-desktop-settings-simplification.md)
- [2026-07-14-readme-v1.1.49-multilingual-refresh.md](./superpowers/plans/2026-07-14-readme-v1.1.49-multilingual-refresh.md)
- [2026-07-22-cloud-usage-efficiency.md](./superpowers/plans/2026-07-22-cloud-usage-efficiency.md)
- [2026-07-22-issue-83-recording-startup.md](./superpowers/plans/2026-07-22-issue-83-recording-startup.md)
- [2026-07-22-provider-recording-limits-managed-upload.md](./superpowers/plans/2026-07-22-provider-recording-limits-managed-upload.md)

### 12.2 specs（设计规格）

- [2026-06-01-custom-local-stt-design.md](./superpowers/specs/2026-06-01-custom-local-stt-design.md)
- [2026-06-02-release-hardening-review.md](./superpowers/specs/2026-06-02-release-hardening-review.md)
- [2026-06-03-cloud-error-capsule-visibility-design.md](./superpowers/specs/2026-06-03-cloud-error-capsule-visibility-design.md)
- [2026-07-12-desktop-settings-simplification-design.md](./superpowers/specs/2026-07-12-desktop-settings-simplification-design.md)
- [2026-07-14-readme-v1.1.49-multilingual-refresh-design.md](./superpowers/specs/2026-07-14-readme-v1.1.49-multilingual-refresh-design.md)
- [2026-07-22-issues-81-83-cloud-usage-design.md](./superpowers/specs/2026-07-22-issues-81-83-cloud-usage-design.md)

---

## 13. 阅读路线建议（按角色）

- **新人 / 接手工程师**：0 节（分析文档 + VISION）→ 第 3 节（平台稳定化）→ 第 9/10 节（平台专项）→ 源码。
- **产品经理 / 贡献者**：0 节 → 第 8 节（竞品路线图）→ 第 4/5/6 节（云/Ask/场景）→ 第 11 节（流程治理）。
- **开源布道 / 渠道**：第 7 节（OpenLess 系列）→ 第 4 节（AppSumo）→ 第 2 节（多语言）。
- **排查具体问题**：先用 [分析文档第 4.3 节故障排查表](./OpenTypeless_项目深度分析.md) 定位，再跳到对应专项 spec。

---

*索引由文档专家于 2026-07-31 建立，与 `OpenTypeless_项目深度分析.md` 互为补充。新增文档请同步更新本索引。*
