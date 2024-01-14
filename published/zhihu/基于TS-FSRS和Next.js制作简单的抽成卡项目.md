## 介绍
为了能够抵消遗忘曲线的影响，在有了FSRS算法以后，还需要一个能来根据FSRS调度的时间来展示笔记数据，并能够可视化的进行下一次调度。
`ts-fsrs-demo`是一个简单的demo，最初的初衷是为了学习`プログラミング必須英単語600+`的单词并且修复`ts-fsrs`在实际项目中存在的问题而制作的。`ts-fsrs-demo`能够让开发者做出类似于背单词那样的抽成卡web项目，利用`vercel`和`planetscale`进行部署和存储数据实现多用户登录和多设备使用功能。

# 基础条件
`ts-fsrs-demo`尽量少用依赖，减少门槛，但不可避免的使用了以下依赖：
```bash
prisma (global) npm install -g prisma # 与数据库交互的ORM框架
dotenv (global) npm install -g dotenv # 加载环境参数
next.js (>= 14.2.0) # React的框架
next-auth（>= 4.24.5） # 鉴权框架
ts-fsrs (>= 3.2.1) # FSRS调度算法实现（TypeScript实现的ESM/CJS版）
tailwindcss (>= 3)
daisyui (>= 4.4.22) # 最流行Tailwind CSS的组件库
```
因此本文章希望读者具备以下的基础条件：
- 掌握React的hooks和Context的基本知识
- 了解Next.js的App Router构建项目和~~Server Actions(非必要)~~
- 了解prisma的基本知识
- 了解一定的MySQL的基本知识
- 了解基本的next-auth的oauth知识（不构建多用户登录项目则非必要）

## 安装demo项目

为了更好的学习该项目，请先将项目clone到本地后尝试运行
```bash
git clone https://github.com/ishiko732/ts-fsrs-demo.git
```
下载完后请使用npm或pnpm或yarn进行安装依赖(文章中全部采用npm)
```
npm install -g prisma
npm install -g dotenv
npm install
```
其中`-g`表示全局安装，由于demo的prisma文件不是放在默认位置上，所以需要使用`dotenv`来加载环境变量的参数。
![[Pasted image 20240114160412.png]]

安装完成后新建一个`.env.local`，内容需要包含：
```bash
DATABASE_URL="mysql://{dbUserName}:{dbPassword}:3306/{dbSchema}" # init database

NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=xxxxxxx # openssl rand -base64 32

GITHUB_ID=xxxx # github clientId
GITHUB_SECRET=xxxxxxx # github clientSecret
GITHUB_ADMIN_ID=xxxx #github user id
```

### 初始化数据库

初始化数据库表结构（将会根据`src/prisma/schema.prisma`来初始化）：
```bash
npm run dbpush
```

![[Pasted image 20240114160749.png]]
当看到类似的消息则表示初始化成功了

### 初始化next-auth
从`ts-fsrs-demo`的v2.0.0版本开始，默认使用了`next-auth`作为划分用户信息并使用了`GitHub Oauth`作为识别用户身份。
所以初始化Next-auth，我们需要在Github上申请一个`Oauth app`：
- https://github.com/settings/developers (`Github Developer Settings`)
- 选择`New Oauth App`

填写以下内容，完成创建操作：
![[Pasted image 20240114162334.png]]
```
Homepage URL: http://localhost:3000/
Authorization callback URL: http://localhost:3000/api/auth/callback/github
```
点击`Generate a new client secret`完成创建`clientSecret`
![[Pasted image 20240114162501.png]]
现在我们知道了`clientId`和`clientSecret`后，可以在`.env.local`填写好信息：
```
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=xxxxxxx # openssl rand -base64 32

GITHUB_ID=dd81b0fb27ce977bdcbd # github clientId 文章发布后已删除，请勿使用该Id
GITHUB_SECRET=ffaffd296afcc0d46b28447655b2a9ac84508263 # github clientSecret 文章发布后已删除，请勿使用该Secret
```

然后利用`openssl rand -base64 32`生成`NEXTAUTH_SECRET`,然后也是填写到`.env.local`
![[Pasted image 20240114162817.png]]

> `GITHUB_ADMIN_ID`在demo项目中并没有实际用到，所以这里不展开，这个id是将你的github设置为管理员

> 在首次登录时会自动注册信息，并将`プログラミング必須英単語600+`的笔记内容导入。


# 抽成卡实现
## 1.扩展TS-FSRS的类型

在[src/types.d.ts](https://github.com/ishiko732/ts-fsrs-demo/blob/main/src/types.d.ts)中：
- 根据[基于TS-FSRS的数据库表设计](https://zhuanlan.zhihu.com/p/672558313),为了能够匹配`Prisma`我们对`ts-fsrs`模块的类型进行了扩展`CardPrisma`，`RevlogPrisma`
![[Pasted image 20240114164846.png]]
- 根据[TS-FSRS的工作流](https://zhuanlan.zhihu.com/p/673902928)，新增了`StateBox类型
![[Pasted image 20240114164214.png]]

## 2.FSRS类型与Prisma类型互换
由于TS-FSRS的类型存在不匹配情况，所以需要进行封装一次类型转换，在[src/vendor/fsrsToPrisma/index.ts](https://github.com/ishiko732/ts-fsrs-demo/blob/main/src/vendor/fsrsToPrisma/index.ts)中，实现了FSRS类型与Prisma类型互换：
- createEmptyCardByPrisma：创建新的卡片
- transferPrismaCardToCard：Prisma类型转回FSRS类型
- stateFSRSStateToPrisma：FSRS的状态类型转为Prisma的状态类型
- stateFSRSRatingToPrisma：FSRS的评分类型转为Prisma的评分类型

## 3.读取FSRS参数

![[Pasted image 20240114171714.png]]
为了减少不必要的数据读取，我们使用`queryRaw`执行自己写的SQL语句来获取FSRS的参数：
```sql
select * from Parameters
where uid=(select uid from Note
		   where nid in (select nid from Card 
						 where cid=${Number(cid)}))
```
并且在参数存在的时候调用`processArrayParameters`完成类型转换。

> 注意：queryRaw 返回的均为`T[]`


##  4. 实现卡片基本操作（数据库与TS-FSRS交互）

在[src/lib/card.ts](https://github.com/ishiko732/ts-fsrs-demo/blob/main/src/lib/card.ts)中，我们实现了卡片相关的查找，调度，忘记，回滚等操作。
### 查找卡片
![[Pasted image 20240114170750.png]]
我们通过Prisma根据`cid`来查找卡片，并返回包含笔记信息，并将返回类型设为`CardPrisma`

### 调度卡片

要完成调度卡片操作，我们需要读取卡片cid或者笔记nid，并根据当前时间来进行调度。
![[Pasted image 20240114171021.png]]
在第49，50行中：
- 我们读取了该用户的FSRS参数
- 利用了`next-auth`来判断该uid是否有权限访问该卡片
```typescript
import { options } from "@/auth/api/auth/[...nextauth]/options";
import { getServerSession } from "next-auth/next";
import type { User } from "next-auth";

type SessionProps = {
	expires: string;
	user?: User;
};
export async function getAuthSession() {
	const session = await getServerSession(options);
	return session as SessionProps | null;
}

export async function isAdminOrSelf(uid: number) {
	const session = await getAuthSession();
	return session?.user?.role === "admin" || session?.user?.id === String(uid);
}
```

当所有条件判断完以后，我们调用fsrs，并将用户的FSRS参数传入，调用repeat进行调度，并返回调度的结果。

### 忘记卡片
![[Pasted image 20240114174151.png]]
通过调用fsrs.forget可以实现忘记卡片，需要传递已复习的卡片信息和当前时间，还有可选项是否重置已复习的次数

![[Pasted image 20240114173019.png]]
跟调度卡片一样，都是读取卡片，判断权限，再传入用户的FSRS参数。不同的是第168行使用的方法为forget，返回的结果是`recordItem`。
```typescript
type RecordLogItem = {
	card: Card;
	log: ReviewLog;
};
```

### 回滚卡片
![[Pasted image 20240114174308.png]]
通过调用fsrs.rollback可以实现回滚上一次操作，需要传递已复习的卡片信息和最新的复习记录。


![[Pasted image 20240114173535.png]]
![[Pasted image 20240114173852.png]]
跟调度卡片一样，都是读取卡片，判断权限，再传入用户的FSRS参数。
不同的是
- 第117行需要读取最后一次`Revlog`记录
- 第132行使用的方法为rollback

## 4.实现卡片交互操作 （Next.js服务端与页面交互）
`ts-fsrs-demo`会在服务端上完成数据初始化读取后，在客户端组件上进行水合操作，所以需要使用状态管理。`ts-fsrs-demo`采用`React.createContext`来创建状态管理(有兴趣的读者可以采用Mobx，Redux来进行状态管理)。

### FSRS参数交互
在登录以后允许用户自定义自己的FSRS参数。
![[Pasted image 20240114203227.png]]

设计思路：
- 读取next-auth的userId信息，读取用户当前的FSRS参数,并通过`defaultValue`回显数据
- 在点击`Save`调用`Server Actions`或请求API保存参数

```jsx
<input name="request_retention" className="input input-bordered w-full"
	type="number" max={0.99} min={0.7} step={0.01}
	defaultValue={params.params.request_retention} />
```

> 采用defaultValue而不采用State，Ref是保证该组件不是客户端组件。

以下是`src/components/settings/FSRSConfig.tsx`采用Server Actions实现的：
![[Pasted image 20240114204233.png]]

> 如果是请求API，则修改submit方法，在内部使用`fetch("/api/xxx")`来实现保存参数操作。

### 初始化卡片数据
根据[TS-FSRS的工作流](https://zhuanlan.zhihu.com/p/673902928)，采用三盒子模型。

设计思路：
- 读取next-auth的userId信息
- 获取当前时间
	- 如果当前时间<4:00(`UTC+0`)，则取昨天
	- 如果当前时间>=4:00(`UTC+0`)，则取今天
- 读取当天已学习的新卡片数量和新卡片当天最大限制(用户设定的限制)
- 读取笔记数据（根据状态设置筛选条件）
- 求和计算笔记集合，判断是否已结束

[src/app/card/page.tsx](https://github.com/ishiko732/ts-fsrs-demo/blob/main/src/app/card/page.tsx)：
![[Pasted image 20240114204857.png]]
![[Pasted image 20240114205818.png]]


### 初始化卡片状态管理和操作

在`CardClient`的最外层使用了`CardProvider`来保证内层组件都可以访问`CardContext`的值。
![[Pasted image 20240114210219.png]]


首先需要将服务器上读取到的数据进行转换为三盒子模型，并设定：
- `open`当前状态为不显示答案信息
- `schedule`当前卡片调度信息为未定义`undefined`
- `currentType`读取指向的盒子是哪个
![[Pasted image 20240114211005.png]]


`CardProvider` 点击评分按钮：
- 请求获取下一张卡片调度情况
- 对数据进行随机排序，其中学习中卡片盒子则是会根据调度时间排序
- 记录当前复习操作到操作回滚栈
- 发送到服务器保存复习记录信息
- 检查是否已结束当日的复习

#### 请求获取下一张卡片调度情况
首次加载，点击评分按钮，回滚卡片都会执行读取调度情况操作。

![[Pasted image 20240114211510.png]]
将采用请求API的方式获取调度信息。
![[Pasted image 20240114211649.png]]

> 在服务器上执行的原因是：
> 1. 采用不信任式模式，即客户端不直接调度卡片
> 2. 保证所有数据的时区均是服务器的时区


#### 对数据进行随机排序
在每次点击完评分按钮后将会执行数据排序
![[Pasted image 20240114212055.png]]
![[Pasted image 20240114211946.png]]

#### 记录当前复习操作到操作回滚栈

完成回滚栈初始化状态：
![[Pasted image 20240114212432.png]]

在每次点击完评分按钮后将会执行入栈操作：
![[Pasted image 20240114212851.png]]

在按下快捷键`Ctrl+Z`或者`⌘+Z`时触发回滚操作
![[Pasted image 20240114213250.png]]

将采用请求API的方式获取回滚后的信息。
![[Pasted image 20240114213331.png]]

#### 发送到服务器保存复习记录信息
在[src/components/schedule/ShowAnswerButton.tsx](https://github.com/ishiko732/ts-fsrs-demo/blob/3d35060aed21edc3ab214db7c78e4e0ecfa805fa/src/components/schedule/ShowAnswerButton.tsx#L22-L28)中调用，并将获取到的数据执行`CardProvider`的`handleChange`方法。
![[Pasted image 20240114213657.png]]

> 这里存在设计的失误，可以将handleClick方法修改后放到`CardProvider`里面


#### 检查是否已完成当日的复习
通过checkFinished方法来判断当前是否已完成当日复习，当未结束时将`currentType`指向非空的盒子
![[Pasted image 20240114214027.png]]
![[Pasted image 20240114214042.png]]