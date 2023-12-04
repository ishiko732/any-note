title:  【Next.js】动态加载组件

---

# 需求
最近在做demo项目[ts-fsrs-demo](https://github.com/ishiko732/ts-fsrs-demo)，遇到了想扩展菜单组件的时候需要每次在`Page.tsx`添加新的ReactNode的情况。如果只有1，2个菜单组件的话还好，但对于项目的扩展性有比较大的限制。
![[Pasted image 20231204110944.png]]

# 想法
- 根据Next.js 13的路由设计思想[1] 和Server Component可以访问后端服务（数据库，文件等），那么我们是不是可以通过动态读取文件加载对应的JSX组件呢？
![[Pasted image 20231204110434.png]]
- 能否基于`next/dynamic`[2] 实现加载某一个目录下的全部组件（Server/Client Component）？


# 实现

## 1.创建通用Menu组件
由于需要访问文件资源，所以我们先要创建一个[Menu](https://github.com/ishiko732/ts-fsrs-demo/blob/31fb6c4f61b38c650f301943d9e808f27cb64d05/src/components/menu/index.tsx)Server Component（服务器组件），让他能够读取文件夹`src/components/menu/items`下的全部tsx组件，并且排除`index.tsx`：

```typescript
async function dynamicReactNodes() {
  // 读取组件
  const itemsDir = path.join(process.cwd(), "src/components/menu/items");
  const filenames = fs.readdirSync(itemsDir);
  return filenames
    .filter(
      (filename) =>
        path.extname(filename) === ".tsx" && filename !== "index.tsx"
    )
    .map((filename) => {
      const Item = dynamic(() => import(`./items/${path.basename(filename)}`), {
        loading: () => (
          <li>
            <span className="loading loading-spinner loading-md"></span>
          </li>
        ),
      });
      return <Item key={filename} />;
    });
}

export default async function Menu() {
  const menuItems = await dynamicReactNodes();
  return (
    <div className="sm:fixed bottom-[3rem] right-[16%] m-4 p-4">
      <ul className="menu bg-base-200 rounded-box border-2  border-stone-500 dark:border-gray-600">
        {menuItems.map((item) => item)}
      </ul>
    </div>
  );
}
```
这样，我们在[Page.tsx](https://github.com/ishiko732/ts-fsrs-demo/blob/31fb6c4f61b38c650f301943d9e808f27cb64d05/src/app/note/page.tsx)下只需要导入这个`Menu`组件便可以渲染全部菜单组件：

```typescript
import Menu from "@/components/menu";

export default async function Page({...}){
	...
	return (
			...
	        <Menu/>
	      </div>
	    </>
}
```

> 该组件会在服务器上渲染完后才到视图层

文件：
https://github.com/ishiko732/ts-fsrs-demo/blob/237ff583d62735e7211173decfed5df956981ae8/src/components/menu/index.tsx
## 2.创建通用MenuItem组件
之前我们排除了`src/components/menu/items/index.tsx`组件，目的就是为了创建通用的[MenuItem](https://github.com/ishiko732/ts-fsrs-demo/blob/31fb6c4f61b38c650f301943d9e808f27cb64d05/src/components/menu/items/index.tsx)组件：
```typescript
...

export default async function MenuItem({
  tip,
  className,
  children,
  onClick,
  formAction,
  dialog,
  disable,
}: Props) {
  if (disable !== undefined) {
    return null;
  }
  return formAction && !onClick ? (
    <form action={formAction}>
      <MenuItemContent
        tip={tip}
        className={className}
        onClick={onClick}
        dialog={dialog}
      >
        {children}
      </MenuItemContent>
    </form>
  ) : (
    <MenuItemContent
      tip={tip}
      className={className}
      onClick={onClick}
      dialog={dialog}
    >
      {children}
    </MenuItemContent>
  );
}

async function MenuItemContent({
  tip,
  className,
  children,
  onClick,
  dialog,
}: Props) {
  return (
    <>
      <li onClick={onClick} className="max-w-[54px] max-h-10">
        <a className={clsx("tooltip tooltip-right", className)} data-tip={tip}>
          {children}
        </a>
      </li>
      {dialog || null}
    </>
  );
}
```
- 为了能够适配Server Actions[3]同时避免混用服务器组件和客户端组件（Client Compoent）用`return formAction && !onClick ? `来决定如何进行渲染。
- 通过`disable`参数可以保证该组件是否进行渲染操作
- 通过`dialog`参数可以扩展对话框组件

> 该组件会在服务器上渲染完后才到视图层

文件：
https://github.com/ishiko732/ts-fsrs-demo/blob/237ff583d62735e7211173decfed5df956981ae8/src/components/menu/items/index.tsx

## 3.创建客户端菜单组件例子
如果需要使用`state`，`context`等资源的话，需要创建客户端组件，在文件首行添加`'use client'`来进行声明。

```typescript
'use client'
import { useState } from "react";
import MenuItem from ".";

function ClientTest() {
  const [cnt,setCnt] = useState(0)
  const handleClick =()=>{
    setCnt(pre=>pre+1)
  }

  return (
    <MenuItem tip="Client Test" onClick={handleClick}>
      <button className="btn btn-xs btn-square" type="submit">
        {cnt}
      </button>
    </MenuItem>
  );
}

export default ClientTest;
```
文件：
https://github.com/ishiko732/ts-fsrs-demo/blob/237ff583d62735e7211173decfed5df956981ae8/src/components/menu/items/client-test_4.tsx

## 4.创建服务器菜单组件例子

```typescript
import MenuItem from ".";

async function ServerTest() {
  const submit = async (formData: FormData) => {
    "use server";
    console.log(formData);
    console.log("hello");
  };

  return (
    <MenuItem tip="Server Test" formAction={submit}>
      <button className="btn btn-square btn-xs" type="submit">
        TEST
      </button>
    </MenuItem>
  );
}

export default ServerTest;
```
文件：
https://github.com/ishiko732/ts-fsrs-demo/blob/237ff583d62735e7211173decfed5df956981ae8/src/components/menu/items/server-test_3.tsx
# 构建结果
通过创建以下文件后，便可以在页面上显示出菜单组件的列表
![[Pasted image 20231204124342.png]]

> TIP: 文件上的_数字.tsx表示排序，目前没有找到更好的办法，除非加多一个配置文件进行配置顺序，但会失去一定的扩展性
> 在[dynamicReactNodes](https://github.com/ishiko732/ts-fsrs-demo/blob/31fb6c4f61b38c650f301943d9e808f27cb64d05/src/components/menu/index.tsx#L34)添加了sort进行扩展排序操作，否则将会按照文件名顺序进行渲染



# 参考
[1]:Routing Fundamentals:https://nextjs.org/docs/app/building-your-application/routing
[2]: Lazy Loading:https://nextjs.org/docs/app/building-your-application/optimizing/lazy-loading#nextdynamic
[3]:【Next.js】在实际工作中使用Server Actions的技巧 https://zhuanlan.zhihu.com/p/670134897
[4]: next-contentlayer: https://vercel.com/templates/next.js/nextjs-contentlayer

