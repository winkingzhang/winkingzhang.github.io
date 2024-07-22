---
title:  "React 组件中避免业务逻辑示例"
header:
  teaser: "/assets/images/teaser.jpg"
categories:
  - React
tags:
  - React
  - Component
  - Refactor
excerpt: >
    通过对示例业务的一步步重构来演示如何避免将业务逻辑混入 React 组件。
---


>
> 文中所引用的源代码均托管在我的个人 GitHub 仓库里：<br/>
> [https://github.com/winkingzhang/react-business-refactor](https://github.com/winkingzhang/react-business-refactor)
>

## 用户故事

假设我们有一个卖书的业务，目前正在开发如下的功能：
> 作为一个购书用户，我希望能看到给定 ISBN 的书籍的详细信息，
> 通过检阅书籍标题、作者和以本地货币显示的折扣后价格（或者原价），以便于决定是否购买。

## 业务分析

站在 React 组件开发的视角，从以上的用户故事中，我们需要创建一个 React 组件来显示书籍的详细信息，主要包含以下几个部分：

- 输入是一个字符串，表示书籍的 ISBN 编号
- 应该有一个后台服务的调用，来检索带有 ISBN 的图书详细信息，其响应包括
  - 书籍标题，不为空的字符串
  - 书籍作者，不为空的字符串
  - 书籍描述，字符串，可以为空
  - 折扣价格或者原价，一个数字，表示货币单位的价格
- 由于 UI 外观不是本文的重点，我将省略所有样式部分，只关注业务逻辑的重构。

## 具体实现

> 所有这些演示代码都在 React 18+ 上运行，它也可以在 React 16+ 上运行，但可能需要多次渲染。

#### 第一版：先让业务跑起来

让我们简化所有样式，我们可以轻松地用 `Typescript` 制作如下所示的 `BookDetailView` 组件

```typescript
import React, { useState } from 'react';

interface BookDetailModel {
  name: String,
  author: String,
  discountPrice?: number,
  price?: number,
}

interface BookDetailViewProps {
  isbn: String,
}

enum AsyncLoadingState {
  Loading = 1,
  Loaded = 0,
  Error = -1,
}

const BookDetailView = ({isbn}: BookDetailViewProps) => {
  const [loadingState, setLoadingState] = 
          useState<AsyncLoadingState>(AsyncLoadingState.Loading);
  const [book, setBook] = useState<BookDetailViewModel | null>();
  
  useEffect(() => {
    const fetchBookDetailAsync = async () => {
      const response = await fetch(`http://localhost:3000/books/${isbn}.json`);
      return await response.json();
    };
    fetchBookDetailAsync()
            .then((book) => {
              setBook(book);
              setLoadingState(AsyncLoadingState.Loaded);
            })
            .catch(() => setLoadingState(AsyncLoadingState.Error));
  }, [isbn]);
  
  if (loadingState == AsyncLoadingState.Loading) {
    return <Text>Loading...</Text>
  } else if (loadingState == AsyncLoadingState.Error) {
    return <Text>Error</Text>
  }
  
  if (!book) {
    return <Text>Not Found</Text>
  }
  
  const onSubmit = () => {
    // ... submit logic here
  };
  
  return (
          <Container>
            <Name value={book.name}/>
            <Author value={book.author}/>
            <Price value={book.discountPrice || book.price}/>
            <Button onClick={onSubmit}>Buy</Button>
          </Container>
  );
}
```

`useState` 这里我们用 **hook** 设置了两个状态

- `loadingState`/`setLoadingState`，指示远程 API 调用的加载状态
  - 正在加载，初始状态，表示组件正在加载数据，会显示加载指示
  - Loaded，成功状态，表示加载数据成功
  - 错误，访问远程 API 时出现错误
- `book`/`setBook`，标明书籍详情
  - 当它为空或未定义时，显示错误消息 “未找到”
  - 否则，显示书籍详细信息

并且我们使用 `useEffect` 钩子来包装 http 请求并从远程 api 调用的响应中异步应用书籍详细信息。

看起来很简单，但是，这个 React UI 组件在第 25 到第 36 行之间混合了一些业务，
一般来说，视图不应该关心如何检索书籍详细信息，因此，这里需要小小重构一下。

> 顺便说一句，我确实包装了如下所示但上面从未提及过的常见组件。

```typescript
const globalNumberFormat = Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD'
});

// @ts-ignore
const Container = ({children}) => (
    <div>{children}</div>
);

// @ts-ignore
const Text = ({children, style}: { children: any, style?: any }) => (
    <div style={style}>{children}</div>
);

// @ts-ignore
const Button = ({children, onClick}: { children: any, onClick?: any }) => (
    <button onClick={onClick}>{children}</button>
);

const Name = ({value: name}: { value: String }) => (
    <Text style={ {fontSize: 36, marginTop: 10} }>{name}</Text>
);

const Author = ({value: author}: { value: String }) => (
    <Text style={ {fontSize: 20, marginTop: 15} }>{author}</Text>
);

const Price = ({value: price}: { value?: number }) => (
    <Text style={ {fontSize: 24, marginTop: 10} }>
        {globalNumberFormat.format(price || 0)}
    </Text>
);
```

#### 第二版，使用自定义钩子重构获取书籍详细信息

如题，这里让我们把获取书籍详细信息的逻辑从 `BookDetailView` 组件中提取出来，放到一个自定义钩子中，
让 `isbn` 作为唯一的参数，其他的加载状态和书籍详细模型放到内部。

正如你看到的，这里需要使用 `useEffect` 来封装 http 请求

```typescript
const useBookDetail = (isbn: String) => {
    const [loadingState, setLoadingState] = 
            useState<AsyncLoadingState>(AsyncLoadingState.Loading);
    const [book, setBook] = useState<BookDetailModel | null>();
    
    useEffect(() => {
        const fetchBookDetailAsync = async () => {
            const response = await fetch(`http://localhost:3000/books/${isbn}.json`);
            return await response.json();
        };
        fetchBookDetailAsync()
            .then((book) => {
                setBook(book);
                setLoadingState(AsyncLoadingState.Loaded);
            })
            .catch(() => setLoadingState(AsyncLoadingState.Error));
    }, [isbn]);
    
    return {loadingState, book};
};

export default useBookDetail;
```
这样在 `BookDetailView` 中就可以移除很多代码，最后看起来是这样的（这里省略了重复部分）：

```typescript
const BookDetailView = ({isbn}: BookDetailViewProps) => {
    const {loadingState, book} = useBookDetail(isbn);
    
    /* omit all the same lines */
    // ......
  
    return (
        <Container>
            <Name value={book.name}/>
            <Author value={book.author}/>
            <Price value={book.discountPrice || book.price}/>
```
看起来好了很多，但是如果聚焦到第 11 行，那么这里计算 `price` 的逻辑还是业务，
它不应该放在 React UI 组件里，接下来我们继续重构。

#### 第三版，彻底封装业务和远程调用

首先快速回顾第二版遇到的几处坏味道：
1. `price` 的计算逻辑应该放在业务逻辑中，而不是 React UI 组件中。
换言之， UI 只应该关心两件事 —— 显示界面元素，响应元素的交互并将控制权转交后台的业务逻辑。
2. `fetchBookDetailAsync` 是硬编码在 `useBookDetail` 钩子中的，实际项目不应该这么做，而是封装成一个 `api` 之类的对象。

这就是我们需要做的事情，让我们继续重构。

###### 分离 View Model 和 Data Model

如上，调用远程 api 返回书籍详情，这个模型可以称之为 Data Model（这里是相对前端而言，对后端 DTO 更为合适）。
要想在 UI 显示，则需要从 Data Model 转换一个纯粹的业务视图模型，它仅仅表示 UI 显示的数据。
这样，就有下面的这两个模型：

```typescript
// data model is placed at 'src/models/data.ts'
export interface BookDetailModel {
    name: String,
    author: String,
    discountPrice?: number,
    price: number,
}

// view model is placed at 'src/models/view.ts'
export interface BookDetailViewModel {
    name: String,
    author: String,
    price: number,
}
```

接下来是转换逻辑，就叫它 `bookDetailHandler` 吧，
它的作用是分析从 http 响应回来的 json 对象，即 Data Model，然后转换为 View Model（这里包含一个简单的单价计算）：

```typescript
const parseBookDetail = (bookDetail: BookDetailModel): BookDetailViewModel => {
    return {
        name: bookDetail.name,
        author: bookDetail.author,
        price: bookDetail.discountPrice || bookDetail.price,
    }
};
```

###### 封装远程调用

这里没有什么特别的，只是把 api 调用单独提取出来放到一个单独的文件里：

```typescript
const getBookDetailByIsbn = async (isbn: String): Promise<BookDetailModel> => {
    const response = await fetch(`http://localhost:3000/books/${isbn}.json`);
    const bookDetail = await response.json();
    return bookDetail as BookDetailModel;
};
```

###### 连接 UI, 业务和 api 调用

接下来修改 `useBookDetail` 钩子，这里仅仅调用业务逻辑，而不是直接调用远程 api：

```typescript
const useBookDetail = (isbn: String) => {
    const [loadingState, setLoadingState] = 
            useState<AsyncLoadingState>(AsyncLoadingState.Loading);
    const [book, setBook] = useState<BookDetailViewModel | null>();
    
    useEffect(() => {
         getBookDetailViewModelByIsbn(isbn)
            .then((book) => {
                setBook(book);
                setLoadingState(AsyncLoadingState.Loaded);
            })
            .catch(() => setLoadingState(AsyncLoadingState.Error));
    }, [isbn]);
    
    return {loadingState, book};
};
```

最后，在 `bookDetailHandler` 里定义 `getBookDetailViewModelByIsbn` 函数，实现如下：

```typescript
const parseBookDetail = (bookDetail: BookDetailModel): BookDetailViewModel => {
    // ... omit long line here ...
};

const getBookDetailViewModelByIsbn = async (
    isbn: String
): Promise<BookDetailViewModel> => {
    return parseBookDetail(await getBookDetailByIsbn(isbn));
}

export {
    getBookDetailViewModelByIsbn,
};
```
这里 `getBookDetailByIsbn` 是从 `api/bookDetail` 里导入的对远端 api 的调用 （这里我用的 mock 数据）。

这样，我们就完成了对业务逻辑的封装，现在 `BookDetailView` 只关心 UI 的显示和交互，而不再关心业务逻辑。

## 总结

就是这么简单，让我们总结一下我们所做的事情，以及为什么这么做：

- 正如标题所示，避免将业务逻辑与 React 组件（UI）混合在一起
- 在单独的模块中抽象外部交互（如果喜欢轻量级解决方案，子文件夹也能满足基本要求）
- 分别定义数据模型和视图模型，并在业务处理程序中将它们桥接起来 
- 与独立的业务处理程序协作，使其更加封装
