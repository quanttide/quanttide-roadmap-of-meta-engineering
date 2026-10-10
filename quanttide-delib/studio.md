# qtcloud-delib studio 本体重建

做法定为删除重写：不迁移、不兼容，旧模型（provider 的 `Topic`/`Resolution` 两对象与 studio 的 `Resolution` 模型）整体作废，按三层本体从头建。旧库表不再维护，种子数据按新模型重新灌；迁移盘点、字段四态、功能保留这些事随之全部消失。词表问题也随删除一并解决——不再有 Topic 占着「议题」，AgendaItem 专有其名。

## 地基

三个共享实体，先建，两侧引用：

```text
Assembly    ::= id   : 机构（公司、联盟、实训基地，单机构场景落默认机构）
                rules : 本机构的议事规则
Member      ::= id     : 代表
                org    : 派出机构
AgendaItem  ::= id       : 议题编号
                summary  : 摘要
                onAgenda : Bool        # 先上议程，才碰草案
```

账号 ID 在这层映射成「某机构的代表」，代表组成代表大会；议程属于机构的一次会议，跨机构交换的是 AgendaItem。

## 过程层

五流程整个收进 Draft，从内部工作流程原样继承，不削格：

```text
Draft ::= item     : 议题编号          # 只挂一条议题
          text     : 版本链            # 每次修订出一版
          proposer : 动议人（代表）
          seconders: 附议人
          debate   : 辩论记录
          votes    : {for, against, abstain}
          state    : proposed → seconded → debated → voted
                     → passed | rejected

second : pre  state = proposed     post seconded
debate : pre  state = seconded     post debated
vote   : pre  state = debated      post voted → passed（发号产生决议） | rejected（可改版重提）
```

门留在讨论过程上：不附议不能辩，不辩论不能表决，不表决不能归档。单文档管理保留到表决点为止，`passed` 那一刻正文定版，此后改动走新版本。

## 效力层

表决通过才发号，效力链从 `pending` 起步：

```text
Resolution ::= id    : 决议号，一号一命
                item  : 议题编号
                org   : 通过它的机构
                draft : 由哪份草案的哪次表决产生
                phase : pending → certified → published

C(r) ≡ phase = certified ∨ phase = published   # 社区决议的判定条件
certify : pre  phase = pending    post certified    # 创始人认证
publish : pre  C(r)               post published    # 官网闸门
```

表决不等于社区决议：接口只画到 `pending`，认证是效力侧自己的下一格。文本自发号起冻结，改动走修正记录，不改原文。

## 关系与不变量

```text
Assembly 有议程，议程收 AgendaItem，AgendaItem 下挂 Draft，Draft 经表决升成 Resolution
inv1  被称为社区决议 ⇒ C(r)
inv2  发在议事中心官网 ⇒ certified(r)
inv3  审议中的草案 ⇒ 其议题在议程上
```

## 建的顺序与验收

顺序：地基（机构、代表、议程）→ 过程层（Draft 五流程与四道门）→ 效力层（发号、认证、发布）→ 按新模型重灌种子 → 验收。验收两查：正查，一份五流程文档走完，决议号、机构归属、效力相位能从它重放出来；反查，任何已发布的社区决议能回溯到唯一一份草案与它所在的议程条目。两查通了，重建完成。
