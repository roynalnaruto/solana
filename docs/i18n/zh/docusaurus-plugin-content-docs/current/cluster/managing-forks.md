---
title: 管理分叉
---

在插槽边界处，账本是可能会产生分叉的。 产生的数据结构形成一个称为 _块存储（blockstore）_ 的树。 验证器解释blockstore时，它必须维护区块链中每个分叉的状态。 我们称每个实例为 _活跃分叉（active fork）_。 验证节点有责任对这些分叉进行权衡，从而最终选择一个分叉。

验证节点通过向该分叉的插槽负责人投票来选择一个分叉。 投票将让验证节点提交一个称为 _锁定期（lockout period）_的持续时间。 在该锁定期到期之前，验证节点无法对其他分叉进行投票。 随后，对同一分叉的每次投票都会将把锁定期延长一倍。 经过一些群集配置的投票数\(当前为32 \) 后，锁定期的持续时间达到了所谓的_最长锁定期（max lockout）_。 在达到最长锁定期之前，验证节点可以选择一直等待，直到锁定期结束然后再投票给另一个分叉。 当它在另一个分叉上投票时，所执行称为 _回滚（rollback）_ 的操作，此时状态会及时回滚到共享检查点，然后跳转到刚刚投票分叉的初始位置。 分叉能够回滚的最大限度称为 _回滚深度（rollback depth）_。 回滚深度是达到最长锁定期所需的票数。 每当验证节点投票时，超出回滚深度的所有检查点都将无法访问。 也就是说，在任何情况下，验证节点回滚的限度都不用超过回滚深度。 因此，它可以安全地 _修剪（prune）_ 无法到达的分叉，并将超出回滚深度的所有检查点 _局限（squash）_ 于根检查点中。

## 活跃分叉

活跃的分叉是一系列检查点，其长度至少比回滚深度长一倍。 最短分叉的长度刚好为回滚深度的一倍。 例如：

![分叉](/img/forks.svg)

以下序列是 _活跃分叉_：

- {4, 2, 1}
- {5, 2, 1}
- {6, 3, 1}
- {7, 3, 1}

## 修剪和挤压

验证节点可以对树中的任何检查点进行投票。 在上图中，就是除了树叶以外的每个节点。 投票后，验证节点修剪从比回滚深度更远的距离派生的节点，然后通过将其可以挤压到根部的任何节点，利用机会来最小化其内存使用量。

从上面的示例开始，回滚深度为 2，考虑对 5 投票与对 6 投票。 首先，对 5 进行投票：

![修剪后的分叉](/img/forks-pruned.svg)

新的根为 2，任何不为 2 后代的活跃分叉都会被修剪。

或者，对 6 进行投票：

![分叉](/img/forks-pruned2.svg)

该树的根始终为 1，因为从 6 开始的活跃分叉距它仅有 2 个检查点。
