
# SRS 算法
SRS（Spaced Repetition System）算法的作用是确定用户现在应该复习什么，或者说是，每个记忆项目下次复习应该发生的时间。任何实现的目标都是为了抵消遗忘曲线的影响：
![[Pasted image 20231224141129.png]]
一旦我们学习或复习了一段知识，遗忘就开始了。SRS算法需要确定两次复习之间的最佳间隔，以确保我们没有完全遗忘它（记忆保留率 `retention=0%`），同时尽量限制复习的次数。实际上，大多数算法使用遗忘指数的10%（即90%的项目能够正确记住），这样我们就不会有太多的项目需要重新复习，同时保持复习次数接近最佳水平。[1] 

# TS-FSRS工作流

> ts-fsrs 是一个基于TypeScript开发的[ES modules包](http://link.zhihu.com/?target=https%3A//gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c) （`ts-fsrs@3.2.3`已支持`commonjs`环境），用于实现[自由间隔重复调度器（FSRS）算法](http://link.zhihu.com/?target=https%3A//github.com/open-spaced-repetition/free-spaced-repetition-scheduler/blob/main/README_CN.md) 的工具。它可以帮助开发者将FSRS应用到他们的记忆卡应用中，从而提升用户的学习体验。

利用`ts-fsrs`，开发者不必过多关注内部卡片状态改变原理，只需要根据用户选择的评分系数以及对应操作来执行对于的方法，卡片将根据其当前状态和评价来进行不同调度操作：
![[fsrs-TS-FSRS WorkFlow.drawio 1.png]]

新卡片`New`：
- 当评分`Rating`为`Again`,`Hard`,`Good`时，卡片的状态`State`将会变更为`Learning`
- 当评分`Rating`为`Easy`，卡片的状态`State`将会变更为`Review`，并根据fsrs参数来选定合适的复习时间

学习中`Learning`/重新学习`Relearning`：
- 当评分`Rating`为`Again`,`Hard`时，卡片状态仍然为`Learning/Relearning`
- 当评分`Rating`为`Good`时，卡片的状态`State`可能变更为`Learning/Relearning`，也可能会变更为`Review`，这取决于`ts-fsrs`调度决定的复习时间
- 当评分`Rating`为`Easy`时，卡片的状态`State`会变更为`Review`，并根据fsrs参数来选定合适的复习时间

复习`Review`:
- 当评分`Rating`为`Again`时，卡片状态将会回到`Relearning`
- 当评分`Rating`为`Hard`,`Good`,`Easy`时，卡片的状态`State`仍然为`Review`，并根据fsrs参数来选定合适的复习时间

> 当`enable_fuzz = true`时，`ts-fsrs`调度排期将会存在一定的随机性，避免卡片粘连

## 扩展
`ts-fsrs`提供2个逆向流程的方法：`rollback`,`forget`

- rollback：回滚本次复习操作，回到上次状态
- forget：手动操作，将卡片设置为新卡片，并设置评分`Rating=Manual`

以上2个方法均需要实现`ts-fsrs`的`Revlog`数据类型结构，以便能正常使用[2]
```typescript
type ReviewLog = {
    rating: Rating; // 复习的评级（手动变更，重来，困难，良好，容易）
    state: State; // 复习的状态（新卡片、学习中、复习中、重新学习中）
    due: Date;  // 上次的调度日期
    stability: number; // 复习前的记忆稳定性
    difficulty: number; // 复习前的卡片难度
    elapsed_days: number; // 自上次复习以来的天数
    last_elapsed_days: number; // 上次复习的间隔天数
    scheduled_days: number; // 下次复习的间隔天数
    review: Date; // 复习的日期
}
```


# TS-FSRS-DEMO 调度盒子

![[fsrs-TS-FSRS-BOXES.drawio.png]]

> `ts-fsr-demo`采用`next.js+prisma`进行设计，利用`react`的SSR特性，在服务器上加载完数据后反馈到客户端，保证了数据源不被恶意调用。

为了能够正常的进行学习/复习操作，采用了上述的三状态盒子，初始数据通过SSR进行获取，使用`React.useContext`实现数据共享（使用vue开发者可直接使用响应式数据）

- 新卡片盒子NewCardBox：
	- 读取每日最大新学习卡片限制，并设定
	- 今日已学习的新卡片数量
	- 卡片当前状态为`New`
	- 对获取的数据进行随机排序
- 学习中卡片盒子LearningCardBox：
	- 卡片当前状态为`Learning`，`Relearning`
	- 对获取的数据根据调度时间从近到远排序
- 复习卡片盒子ReviewCardBox：
	- 卡片当前状态为`Review`
	- 卡片的调度时间`>=`每日起点（ts-fsrs-demo设为每日的4点`GMT+0`）
	- 对获取的数据进行随机排序

其中还涉及复习页面记录操作的特殊栈：
- 操作回滚栈RollbackStack：
	- 记录每一次复习操作的卡片id`cid`和下一次调度状态`nextState`
		- 以便回滚操作执行后，三状态盒子的卡片量和状态正确

[ts-fsrs-demo](https://github.com/ishiko732/ts-fsrs-demo/blob/ea029343f24b1240d670b6db12c7ef41a4ff4217/src/context/CardContext.tsx)在每一次复习后：
- 会对数据进行随机排序，其中学习中卡片盒子则是会根据调度时间排序
- 记录当前复习操作到操作回滚栈
- 发送到服务器保存复习记录信息
- 请求获取下一张卡片调度情况


# 参考

[1]: AnKi SRS ALGORITHM https://www.juliensobczak.com/inspect/2022/05/30/anki-srs.html
[2]: 【FSRS】利用TS-FSRS进行调度卡片介绍 - 小石子的文章 - 知乎
https://zhuanlan.zhihu.com/p/672444457
