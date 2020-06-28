# Taro RFC

大部分特性更新、Bug 修复的代码变更可以遵循普通的 GitHub Flow 流程，通过发起 Pull Request 和 Code Review 完成。但有一些更新会导致系统架构、API 发生根本改变——对于这类的更新我们需要设计一个流程来保证 Taro 团队和社区达成共识。

RFC （Request For Comments）就是一种为了保证重大特性更新和架构更改能够顺利推进的一种流程机制。

## 哪类更改需要 RFC 流程

如前文所述，任何导致系统架构，API 发生改变时或是导致代码整体结构更改都需要 RFC 流程，例如：

- 添加一个需要开发者安装的包
- 修改 API 参数或删除 API
- 改变 Taro 既有惯用开发方式的方案
- 任何导致已发布版本不可用的更改

以下类型的改动不需要进行 RFC 流程：

- Bug 修复
- 不影响已发布版本使用的特性迭代
- 添加或删除警告
- 内部架构调整或代码重构
- 性能提升或兼容性提升

如果开发者提交了本应该走 RFC 流程的 Pull Request，Taro 团队会把 Pull Request 关闭并请发起 Pull Requst 的开发者在 RFC 仓库执行 RFC 流程。

## 为什么需要 RFC 流程？

繁琐的流程对开发和迭代效率毫无疑问是有害的，但出于以下的原因，让我们即便愿意降低迭代效率也要引入 RFC 机制：

1. 稳定性。现有 RFC 流程能让我们尽可能降低每次迭代对老用户的影响。RFC 流程同时会一定程度和版本发布挂钩，保证 Taro 的迭代尽可能符合语义化版本号规范。
2. 开放与公开。我们希望 Taro 的迭代尽可能地开放、公开、透明，每一个人都可以参与 RFC 的讨论，影响下一个版本的 Taro。同时我们也希望借此提升 Taro 社区的参与度，让更多人参与到 Taro 核心的建设。
3. 代码质量。现有 RFC 流程要求实现之前有设计文档，这对于具体的代码实现有非常大的帮助。
4. 追踪设计。我们希望每一个 RFC 文档和背后的 PR 都能反应我们新增、取消或修改一项特性背后的思考与妥协，这对于 Taro 长期的发展和想要深入了解 Taro 的开发者来说非常重要。

## RFC 流程机制

简而言之，如果需要 RFC 改动在未来的 Taro 版本在发布，需要先让 RFC 提案文档在 RFC 仓库合并。

- Fork Taro RFC 仓库： http://github.com/nervjs/taro-rfcs；
- 复制 `0000-template.md` 到 `rfcs/0000-my-feature.md` （`my-feature` 需要更改为一个能够描述该 RFC 的词语，`0000` 是 RFC 的序号，先不用更改）；
- 根据模板 active/0000-my-feature 填充 RFC 提案：RFC 需要描述清晰的动机与原因、展示具体的案例和设计的细节、以及可能造成的问题。RFC 提案进入 `Pending` 阶段；
- 向 Taro RFC 仓库发起 Pull Request。在这个阶段 Taro 团队和社区一同讨论 RFC 提案；
- RFC 提案在社区中达成共识，RFC 提案进入 `Active` 阶段；
- RFC 提案的代码实现在 Taro 主仓库被合并，RFC 提案进入 `Landed` 阶段，RFC 提案也会被合并进入 RFC 仓库；
- 最终 [Taro 团队](https://nervjs.github.io/taro/docs/team)会决定 RFC 提案的变更在哪个版本开始释出。

## 在 `Pending` 阶段的 RFC 提案

当一个 RFC 提案作为 Pull Request 请求合并时：

- RFC 提案会被 Taro 团队和社区一同讨论，最终得出的细节和结论不一定会和 RFC 的最初提案一致；
- RFC 提案可能会由于没有达成共识而被拒绝，至少有一位 Taro 团队的成员会给出拒绝的原因；
- RFC 提案在达成共识之后会进入 `Active` 阶段，Taro 团队或社区可以开始着手提案的代码实现；
- RFC 提案可能因为无人讨论因此无法达成共识，Taro 团队可以根据实际情况是否接受提案

## 在 `Active` 阶段的 RFC 提案

当一个 RFC 提案在社区中达成共识，进入 `Active` 阶段之后，并不意味着 RFC 提案的更改就是板上钉钉的事情，RFC 提案的代码实现不一定可以合入 Taro 代码的主仓库。但 RFC 提案进入了 `Active` 的确意味着 Taro 团队和社区认可 RFC 提案的更改，并认为这是将来应该去实现的一件事。

也就是说，当 RFC 提案进入 `Active` 阶段时并不认为这个改动被分配了优先级，具体的版本发布时间也需要在代码实现之后才能确定。

## RFC 提案的实现

RFC 提案文档的作者没有义务去进行 RFC 提案的代码实现。当然， RFC 提案的作者（或任何一位开发者）如果有意愿在 RFC 提案文档被接受后去进行代码实现，这也是非常受欢迎的。

已经被接受的 RFC 提案需要包含一个在 Taro 主仓库的 Pull Request 链接（即便 PR 还在起草或未完成阶段），RFC 提案的作者和 Taro 团队需要去督促 RFC 实现 PR 朝着 RFC 提案文档所描述的方向去推进。

Taro RFC 流程机制受到了来自 [Rust RFC Process](https://github.com/rust-lang/rfcs)、[React RFC Process](https://github.com/reactjs/rfcs) 和 [Vue RFC Process](https://github.com/vuejs/rfcs) 的启发。
