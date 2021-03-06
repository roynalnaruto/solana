---
title: 乐观确认
---

## 原理

`vote(X, S)`-投票将增加“参考”位，`X`这是该验证节点通过切换证明，投票通过的分叉的**最新**祖先。 只要验证节点进行的连续投票都是彼此相同的，`X`应该用于所有的这些投票。 验证节点对一个插槽`s`进行投票时，它不是前一个的后代，`X`将设置为新的插槽`s`。 所有投票将采用以下`vote(X, S)`的形式，其中`S`为插槽等待投票的排序列表`(s, s.lockout)`。

对于一个投票`vote(X, S)`，让`S.last==vote.last`成为最后一个插槽`S`。

现在，我们定义一些“乐观罚没”的罚没条件。 直觉描述如下：

- `直觉`: 如果验证节点提交 `voite(X, S)`，同一个验证节点不应该在一个不同的分叉上投票（这个分叉有“重叠”。） 更具体而言，这个验证程序不应该再投另一个票`voite(X', S')` 范围 `[X, S.` 与范围重叠 `[X', S'.last]`, `X != X'`，如下所示：

```text
                                  +-------+
                                  |       |
                        +---------+       +--------+
                        |         |       |        |
                        |         +-------+        |
                        |                          |
                        |                          |
                        |                          |
                    +---+---+                      |
                    |       |                      |
                X   |       |                      |
                    |       |                      |
                    +---+---+                      |
                        |                          |
                        |                      +---+---+
                        |                      |       |
                        |                      |       |  X'
                        |                      |       |
                        |                      +---+---+
                        |                          |
                        |                          |
                        |                          |
                        |                          |
                        |                      +---+---+
                        |                      |       |
                        |                      |       |  S'.last
                        |                      |       |
                        |                      +-------+
                        |
                    +---+---+
                    |       |
                 s  |       |
                    |       |
                    +---+---+
                        |
                        |
                        |
                        |
                    +---+---+
                    |       |
             S.last |       |
                    |       |
                    +-------+
```

(可选票的示例(X', S') 和选票(X, S))

在上图中，请注意，对`S.last`的投票必须在对`S'.last`的投票之后发送(由于锁定，较高的投票必须在以后发送)。 因此，投票顺序必须是： `X ... S'.last ... S.last`。 这意味着在对`S'.last`进行投票后，验证节点必须已在某个插槽`s > S'.last > X`上切换回了另一个分叉。 因此，对`S.last`的投票应该使用`s`作为“参考点”，而不是`X`，因为那是分叉上的最后一个“开关”。

为了实现这一点，我们定义了“乐观罚没”的罚没条件。 给定两个不同的投票 `voite(X, S)`和 `voite(X, S)`，经过同一个验证程序的投票必须满足：

- `X <= S.last`, `X' <= S'.last`
- `S` 中所有的 `s` 都是彼此的祖先/后代，`S'` 中的所有 `s'` 都是同伴/同辈。
-
- `X == X'` 暗示 `S` 是 `S'` 父级，或者 `S'` 是 `S` 的父级
- `X' > X` 意味着 `X' > S.last` 和 `S'.last > S.last` 所有 `s` 在 `S` 之中，`s + lockout(s) < X'`
- `X > X'` 意味着 `X > S.last` 和 `S.last > S'.last` 所有 `s` 在 `S'`之中，`s + lockout(s) < X`

(最后两个规则意味着范围不能重叠)：否则，验证程序将被罚没。

`范围(投票)` - 投票 `v = vote(X, S)`, 定义 `Range(v)` 为插槽 `[X, S.last]`的范围。

`SP(old_vote，new_vote)`-这是验证节点最新投票`old_vote`的“切换证明”。 每当验证节点切换其参考位(请参见上面的投票部分) 时，此类证明都是必需的。 切换证明包括对`old_vote`的引用，因此存在`old_vote`的范围是什么的记录(以使该范围内的其他冲突开关可被罚没)。 这样的开关仍必须遵守锁定条件。

交换证明表明，网络的`> 1/3`被锁定在`old_vote.last`插槽中。

证明是元素`(validator_id，validator_vote(X，S))`的列表，其中：

1. 所有验证节点ID的质押总和`> 1/3`

2. 对于每 `(validator_id, validator_voite(X, S))`，在 `S` 中存在一些插槽 `s` 其中： _ a.`s` 不是两者的共同祖先 `validator_vote.last` 和 `vote.last` 和 `new_vote.last`。 _ b. `s` 不是 `validator_voote.last` 的后代。 \* c. `s + s.lockout() >= old_vote.last` (隐含着验证节点仍然被锁定在`old_vote.last`的`s` 插槽)。

在没有有效切换证明的情况下切换分叉是可罚没的。

## 定义：

乐观确认 - 如果一个模块 `B` 已经实现"乐观确认"，假如 `>2/3` 质押以投票方式投票 `v`，其中 `Range(v)` 对每个这样的 `v` 包括 `B.slot`。

已完成 - 如果至少有一个正确的验证节点已植入 `B` 或是 `B` 的后代，那么就说区块 `B` 已经完成。

已恢复 - 如果另一个 `B'` 不是 `B` 的父级或后代，则说区块 `B`已恢复。

## 保障：

除非至少罚没了一个验证节点，否则已经达成乐观确认的区块`B`将不会恢复。

## 证明:

假定是出于矛盾， 一个区块 `B` 已经在某些槽位 `B + n`对于某些 `n`实现了`乐观确认` 并且：

- 另一个区块 `B'` 不是已经完成的区块 `B `的父级或后代。
- 没有验证节点违反任何罚没条件。

由 `乐观确认`的定义，意味着 `> 2/3` 验证程序中的每一个都显示了一些投票 `v` 表单 `Vote(X, S)` `X <= B <= v.last`。 称这组验证节点为 `乐观验证节点`。

现在给定 `乐观验证节点`的一个验证节点 `v` ，有`v`， `Vote(X, S)` 和 `Vote(X', S')`的两个投票，其中 `X <= B <= S.last` 和 `X' <= B <= S'.last`，然后 `X == X'` 否则违反了“乐观罚没”条件(每次投票的“幅度”会重叠在 `B`)。

因此，将`乐观投票`定义为`乐观验证节点`做出的一组投票，其中，对于每个乐观验证节点`v`而言，集合中包括的`v`做出的投票就是`最大`投票`(X, S)`，在满足`X <= B <= S.last` 的`v`做出的任何投票中，具有最高的`S.last`。 因为我们从上方知道`X`对由`v`做出的所有此类投票都是唯一的，所以我们知道有如此独特的对`maximal`进行的投票。

### 引理 1:

`声明：`给出了由`乐观验证节点`集中的验证节点`V`做出的`Vote(X, S)`投票，而`S`包含了针对`s`插槽表决，其中：

- `s + s.lockout > B`,
- `s` 不是 `B` 的前辈或后代，

那么 `X > B`

```text
                                  +-------+
                                  |       |
                        +---------+       +--------+
                        |         |       |        |
                        |         +-------+        |
                        |                          |
                        |                          |
                        |                          |
                        |                      +---+---+
                        |                      |       |
                        |                      |       |  X'
                        |                      |       |
                        |                      +---+---+
                        |                          |
                        |                          |
                        |                      +---+---+
                        |                      |       |
                        |                      |       |  B (Optimistically Confirmed)
                        |                      |       |
                        |                      +---+---+
                        |                          |
                        |                          |
                        |                          |
                        |                      +---+---+
                        |                      |       |
                        |                      |       |  S'.last
                        |                      |       |
                        |                      +-------+
                        |
                    +---+---+
                    |       |
                 X  |       |
                    |       |
                    +---+---+
                        |
                        |
                        |
                        |
                        |
                        |
                    +---+---+
                    |       |
            S.last  |       |
                    |       |
                    +---+---+
                        |
                        |
                        |
                        |
                    +---+---+
                    |       |
      s + s.lockout |       |
                    +-------+
```

`证明`: 假定为了与验证节点 `V` 从设置的“乐观验证节点”进行了这样一次投票 `Voite(X, S)` 其中 `S` 包含对一个插槽的投票，`s` 不是 `B` 的一个父辈或后代，其中 s`s + s.lockout > B`, 但是 `X <= B`。

令`Vote(X', S')`为验证节点`V`所做的`乐观投票`集合中的投票。 根据该集合的定义(所有选票均乐观地确认为`B`)，`X' <= B <= S'.last` (请参见上图)。

这意味着，因为假定它在 `X <= B`，然后在 `X <= B`上，所以根据罚没规则， `X == X'` or `X < X'`(否则会重叠范围`(X', S'.last)`)。

`Case X == X'`:

考虑 `s`。 我们知道 `s != X` ，因为它假定 `s` 不是`B`的一个祖先或后代 , `X` 是 `B` 的祖先。 因为 `S'.last` 是 `B`的后代，意味着 `s` 也不是`S.last`的祖先或后代。 然后因为 `S.last` 来自 `s`，然后又 `S.last` 不能是 `S.last`的祖先或后代。 这意味着根据"乐观罚没"规则，`X != X` 。

`Case X < X'`:

直觉上，这意味着 `Vote(X, S)` 是在 `Vote(X, S)`之前发生的。

根据上述假设， `s + s.lockout > B > X'`。 因为`s`不是`X'`的祖先，所以当此验证节点首次尝试以`Vote(X', S'')`的形式向`X'`提交切换表决时，将违反锁定。

由于这些情况均不成立，因此该假设必定无效，并且假设已得到证明。

### 引理 2:

回想一下，`B'`是在与乐观地确认的`B`区块不同的分叉上最终确定的区块。

`声明`: 任何投票 `Vote(X, S)` 在 `乐观投票` 集合中必须为 `B' > X`

```text
                                +-------+
                                |       |
                       +--------+       +---------+
                       |        |       |         |
                       |        +-------+         |
                       |                          |
                       |                          |
                       |                          |
                       |                      +---+---+
                       |                      |       |
                       |                      |       |  X
                       |                      |       |
                       |                      +---+---+
                       |                          |
                       |                          |
                       |                      +---+---+
                       |                      |       |
                       |                      |       |  B (Optimistically Confirmed)
                       |                      |       |
                       |                      +---+---+
                       |                          |
                       |                          |
                       |                          |
                       |                      +---+---+
                       |                      |       |
                       |                      |       |  S.last
                       |                      |       |
                       |                      +-------+
                       |
                   +---+---+
                   |       |
    B'(Finalized)  |       |
                   |       |
                   +-------+
```

`证明`: 让 `Vote(X, S)` 在 `乐观投票` 集合中的一次投票。 然后按定义，给出"最佳确认" 区块 `B`, `X <= B <= S.last`。

因为 `X` 是 `B`的父类，并且 `B'` 不是 `B`，那么：

- `B' != X`
- `B'` 不是 `X` 的父类

现在请考虑是否 `B'` < `X`:

`Case B' < X`: 我们会证明这样是违反锁定的。 从上面我们知道 `B'` 不是 `X` 的父级。 从上面我们知道`B‘`不是`X`的父代。然后，因为`B'`是根植的，所以验证节点不应该能够对不是来自区块`B'`的更高插槽`X`进行投票。

### 安全性证明：

现在，我们旨在显示`乐观验证节点`集合中至少有一个验证节点违反了罚没的规则。

首先要注意的是，为了使`B'`已经植入根目录，必须有`> 2/3`的质押对`B'`或`B'`的后代进行投票。 假设`乐观验证节点`集还包含`> 2/3`的质押验证节点，则得出`> 1/3`的质押验证节点：

- 根植于 `B'` 或 `B`的后代
- 也提交了一个表 `Vote(X, S) ` 的投票 `v`，其中 `X <= B <= v.last`。

让 `Delinquent` 设置为符合以上标准的验证集。

根据定义，为了根植于 `B'`, 每个验证节点 `V` 在 `Delinquent` 都必须对表单进行一些"切换投票" `Vote(X_v, S_v)`，其中：

- `S_v.last > B'`
- `S_v.last` 是 `B'` 的后代，因此它不能是 `B` 的后代。
- 因为 `S_v.last` 不是 `B` 的后代， 然后 `X_v` 不能是 `B` 的后代或祖先。

根据定义，这个过时的验证人`V`还在`乐观投票`中进行了 `Vote(X, S)`，其中根据该集合的定义(经过乐观确认的`B`)，我们知道 `S.last >= B >= X`。

通过`引理2`，我们知道 `B' > X`，以及上文的 `S_v.last > B'`，所以`S_v.last > X`。 因为 `X_v != X` (不能从上方成为`B`的后代或祖先)，那么根据罚没规则，我们知道 `X_v > S.last`。 上面有 `S.last >= B >= X` ，因此对于所有这样的切换投票，有 `X_v > B`。

现在按顺序排列所有这些切换投票，假设`V`是`乐观验证节点`中的验证节点，该它首先提交这样的切换投票`Vote(X', S')`，其中 `X' > B`。 我们知道存在这样的验证节点，因为从上面知道所有违规验证节点都必须提交这样的投票，并且违规验证节点是`乐观验证节点`的子集。

令`Vote(X, S)`为验证节点`V`(最大化`S.last`)在`乐观投票`中的唯一投票。

给定`Vote(X, S)`，因为 `X' > B >= X`，然后给出 `X' > X`，因此按照“乐观罚没”规则，得出 `X' > S.last`。

为了对 `X'` 执行这样的“切换投票”，切换证明 `SP(Vote(X, S), Vote(X', S'))` 必须显示 `> 1/3` 的质押锁定此验证节点的最新投票`S.last`。 将此 `>1/3` 与`乐观投票者`集合中的一组验证节点的质押 `>1/3` 这个事实结合，就意味着`乐观投票者`中至少有一个乐观验证节点`W`。集合必须已提交投票(回顾切换证明的定义)，验证节点`V`的插槽`X'`的切换证明中包含的`Vote(X_w, S_w)`，其中`S_w`包含了一个插槽`s`，所以有：

- `s` 不是 `S.last` 和 `X'` 的共同祖先
- `s` 不是 `S.last` 的后代。
- `s' + s'.lockout > S.last`

因为 `B` 是 `S.last`的祖先，那么下面也是真的：

- `s` 不是 `B` 和 `X'` 的共同祖先
- `s' + s'.lockout > B`

包含在 `V` 的切换证明中。

现在，由于`W`也是`乐观投票者`的成员，因此由上面的`引理1`决定，由`W`投票，即 `Vote(X_w, S_w)`，其中`S_w`包含对插槽`s`的投票，其中 `s + s.lockout > B`，并且`s`不是`B`的祖先，那么 `X_w > B`。

因为验证节点`V`在其为`X'`插槽进行切换的证明中包括了投票`Vote(X_w, S_w)`，所以隐含了验证节点`V'`在投票给 vote `Vote(X_w, S_w)` **之前**，`V` 就对插槽 `X'` 进行了 `Vote(X', S')` 投票。

但这是一个矛盾，因为我们选择 `Vote(X', S')`作为`乐观投票者`集合中任何验证节点的第一票，其中 `X' > B` 且`X'`不是`B`的后代。
