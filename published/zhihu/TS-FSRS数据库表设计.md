ts-fsrs已实现卡片调度（含抖动）和回滚/重置操作，为了存储相关数据，你至少需要完成以下表设计：
![[fsrs.drawio.svg]]
利用ts-fsrs实现的程序的表名和字段不一定需要跟上图一致，但推荐字段保持一致(`Note`表除外)。

其中：
- Parameters：FSRS核心参数，当为多用户需要设计该表，单用户使用推荐使用`env`
- Note：笔记内容
- Card：笔记对应的卡片（可选1对1关系，上图表述为1对多）
- Revlog：卡片复习记录

> 提示：当存在牌组(deck)的情况下，可以创建牌组(`Deck`)与笔记(`Note`)的1对多关系

## FSRSParameters

ts-fsrs最新版本按照FSRS V4版本进行重构，需要以下核心参数：

- `request_retention`：记忆概率; 代表你想要的目标记忆的概率。
	- 默认值参照`ts-fsrs`的`default_request_retention=0.9`
	- 注意：在较高的保留率和较高的重复次数之间有一个权衡。建议你把这个值设置在0.8和0.9之间。
- `maximum_interval`:最大间隔天数;
	- 默认值参照`ts-fsrs`的`default_maximum_interval=36500`
	- 注意：复习卡片间隔的最大天数。 当复习卡片的间隔达到此天数时， 「困难」、「良好」和「简单」的间隔将会一致。 此间隔越短，工作量越多。
- `w`：FSRS优化器参数；
	- 默认值参照`ts-fsrs`的`default_w=[0.4, 0.6, 2.4, 5.8, 4.93, 0.94, 0.86, 0.01, 1.49, 0.14, 0.94, 2.18, 0.05, 0.34, 1.26, 0.29, 2.61]`
	- 通过运行FSRS优化器(目前有[fsrs-optimizer](https://github.com/open-spaced-repetition/fsrs-optimizer)，[fsrs4anki](https://github.com/open-spaced-repetition/fsrs4anki)，[fsrs-rs](https://github.com/open-spaced-repetition/fsrs-rs)可使用)创建的参数。默认情况下，这些是由样本数据集计算出来的权重。
- `enable_fuzz`：启用抖动
	- 默认值参照`ts-fsrs`的`default_enable_fuzz=false`
	- 注意：当启用时，这将为新的间隔时间增加一个小的随机延迟，以防止卡片粘在一起，总是在同一天被复习。

## Note表
Note表用于存储内容信息，取决于你怎样保存数据
- `nid`：笔记id
- `question`：问题本身
- `answer`：答案本身
- `extend`：扩展字段

## Card表
用于关联笔记Note与卡片的之间的关系，可以是1对1也可以是1对多关系，取决你程序如何设计
- `nid`：笔记id
- `cid`：卡片id
- `due`：调度日期；创建卡片时取当前时间
- `stability`：记忆稳定性；创建时取0
- `difficulty`：卡片难度系数；创建时取0
- `elapsed_days`：自上次复习以来的天数；创建时取0
- `scheduled_days`：下次复习的间隔天数；创建时取0
- `reps`：卡片被复习的总次数；创建时取0
- `lapses`：卡片被遗忘或错误记忆的次数；创建时取0
- `state`：卡片的当前状态（新卡片`0`、学习`1`、复习`2`、重新学习`3`）
- `last_review`：最近的复习日期（如果适用）；创建时未定义`Null`


>`ts-fsrs`允许`due`,`last_review`使用除了日期类型之外的`char,varchar,number,timestamp`，只要是符合日期表述的输入则可以内部方法`fixDate`修正为`date`类型。

```typescript
export function fixDate(value: unknown) {  
  if (typeof value === "object" && value instanceof Date) {  
    return value;  
  } else if (typeof value === "string") {  
    const timestamp = Date.parse(value);  
    if (!isNaN(timestamp)) {  
      return new Date(timestamp);  
    } else {  
      throw new Error(`Invalid date:[${value}]`);  
    }  
  } else if (typeof value === "number") {  
    return new Date(value);  
  }  
  throw new Error(`Invalid date:[${value}]`);  
}
```

>`ts-fsrs`允许`state`使用非枚举类型，使用限定的`number,int`($0\le state \le 3$)和特定的文本`New`,`Learning`,`Review`,`ReLarning`(不限制大小写)

```typescript
export function fixState(value: unknown): State {  
  if (typeof value === "string") {  
    const firstLetter = value.charAt(0).toUpperCase();  
    const restOfString = value.slice(1).toLowerCase();  
    const ret= State[`${firstLetter}${restOfString}` as keyof typeof State]  
    if(ret === undefined){  
      throw new Error(`Invalid state:[${value}]`);  
    }  
    return ret;  
  } else if (typeof value === "number") {  
    return value as State;  
  }  
  throw new Error(`Invalid state:[${value}]`);  
}
```

## Revlog表
- `lid`：复习记录id
- `cid`：卡片id
- `rating`：复习的评级（手动变更`0`，重来`1`，困难`2`，良好`3`，容易`4`）
- `state`：复习前的状态（新卡片`0`、学习`1`、复习`2`、重新学习`3`）
- `due`：上次的调度日期，回滚卡片使用
- `stability`：复习前的记忆稳定性，回滚卡片使用
- `difficulty`： 复习前的卡片难度，回滚卡片使用
- `elapsed_days`：自上次复习以来的天数
- `last_elapsed_days`：上次复习的间隔天数，回滚卡片使用
- `scheduled_days`：下次复习的间隔天数
- `review`：复习的日期
- `duration`：复习时长（可选）

>`ts-fsrs`允许`state`使用非枚举类型，使用限定的`number,int`($0\le state \le 3$)和特定的文本`New`,`Learning`,`Review`,`ReLarning`(不限制大小写)

>`ts-fsrs`允许`rating`使用非枚举类型，使用限定的`number,int`($0\le rating \le 4$)和特定的文本`Manual`,`Again`,`Hard`,`Good`,`Easy`(不限制大小写)

```typescript
export function fixRating(value: unknown): Rating {  
  if (typeof value === "string") {  
    const firstLetter = value.charAt(0).toUpperCase();  
    const restOfString = value.slice(1).toLowerCase();  
    const ret = Rating[`${firstLetter}${restOfString}` as keyof typeof Rating]  
    if(ret === undefined){  
      throw new Error(`Invalid rating:[${value}]`);  
    }  
    return ret;  
  } else if (typeof value === "number") {  
    return value as Rating;  
  }  
  throw new Error(`Invalid rating:[${value}]`);  
}
```

其中`Manual`在调用`fsrs.forget`产生：
```typescript
forget = (  
  card: CardInput,  
  now: DateInput,  
  reset_count: boolean = false,  
): RecordLogItem => {
	...
}

export type RecordLogItem = {  
  card: Card; log: ReviewLog  
}
```

其中`Again`,`Hard`,`Good`,`Easy`在调用`fsrs.repeat`产生：
```typescript
repeat = (card: CardInput, 
		   now: DateInput): RecordLog => {
	...
}
  
type ExcludeManual<T> = Exclude<T, Rating.Manual>;  
  
export type Grade = ExcludeManual<Rating>;

export type RecordLogItem = {  
  card: Card; log: ReviewLog  
}

export type RecordLog = {  
  [key in Grade]: RecordLogItem;  
};
```


# 使用Prisma实现表结构
目前[ts-fsrs-demo](https://github.com/ishiko732/ts-fsrs-demo/tree/v1.0.0)采用相似的数据库表结构的案例，`ts-fsrs-demo`采用[Prisma](https://github.com/ishiko732/ts-fsrs-demo/blob/main/src/prisma/schema.prisma)作为连接数据库，并采用prisma的关系依赖，不建立实际的数据库外键关系。

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider     = "mysql"
  url          = env("DATABASE_URL")
  relationMode = "prisma"
}

model Revlog {
  lid            String   @id @unique @default(cuid())
  cid            Int
  grade          Rating
  state          State
  due            DateTime
  stability      Float
  difficulty     Float
  elapsed_days   Int
  last_elapsed_days   Int
  scheduled_days Int
  review         DateTime
  card           Card     @relation(fields: [cid], references: [cid])

  @@index([cid])
}

model Card {
  cid            Int    @id @unique @default(autoincrement())
  due            DateTime  @default(now())
  stability      Float
  difficulty     Float
  elapsed_days   Int
  scheduled_days Int
  reps           Int
  lapses         Int
  state          State     @default(New)
  last_review    DateTime?
  nid            Int    @unique
  note           Note      @relation(fields: [nid], references: [nid])
  logs           Revlog[]
}

model Parameters {
  uid               Int     @id @unique @default(autoincrement())
  request_retention Float   @default(0.9)
  maximum_interval  Int     @default(36500)
  w                 Json
  enable_fuzz       Boolean @default(false)
}

model Note {
  nid      Int @id @unique @default(autoincrement())
  question String @default("")
  answer   String @default("")
  extend   Json
  card     Card?
}

enum State {
  New        @map("0")
  Learning   @map("1")
  Review     @map("2")
  Relearning @map("3")
}

enum Rating {
  Manual @map("0")
  Again @map("1")
  Hard  @map("2")
  Good  @map("3")
  Easy  @map("4")
}
```
