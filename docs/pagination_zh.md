# 分页

通常，应用程序中会有一些视图需要展示包含大量fetch或及时的数据的列表，分页是最常见得解决方案，值得欣慰的是Apollo Client内置的分页功能可以轻松完成分页。

获取分页有两种基本方式：编号页和游标标记。两种其他方式：离散页面和滚动加载。想深入了解它们之间差异以及何时应该合理使用上述方式，推荐阅读《Understanding Pagination》这篇博客，详情请戳这里：[Understanding Pagination](https://medium.com/apollo-stack/understanding-pagination-rest-graphql-and-relay-b10f835549e7%29)。

> 在本文中，我们将介绍使用Apollo实现分页基本方式和其他方式的技术细节。

## 使用 `fetchMore`
在Apollo中，使用`fetchMore`函数是实现分页最简单的方法，`fetchMore`由高阶组件`graphql`的`data`prop提供。 该方法可以自动将GraphQL新的查询结果合并到原始的查询结果中。

你可以指定要用于新查询的`query`和`variables`，以及如何将新查询结果与客户端上的现有数据合并。 具体怎么做将决定你实现什么样的分页。以下是几种具体的实现。

## 基于偏移量的分页（offet-based）

基于偏移量的分页（也称为编号页面）是一种非常常见的分页方式，由于在后端实现最容易，许多站点都在使用。例如，在SQL中，可以使用（[OFFSET and LIMIT](https://www.postgresql.org/docs/8.2/static/queries limit.html)轻松生成带编号的页面。

一个来自[GitHunt](https://github.com/apollographql/GitHunt-React)带编号页的例子:

```js
const FEED_QUERY = gql`
  query Feed($type: FeedType!, $offset: Int, $limit: Int) {
    currentUser {
      login
    }
    feed(type: $type, offset: $offset, limit: $limit) {
      id

    }
  }
`;
const FeedData = ({ match }) => (
  <Query
    query={FEED_QUERY}
    variables={{
      type: match.params.type.toUpperCase() || "TOP",
      offset: 0,
      limit: 10
    }}
    fetchPolicy="cache-and-network"
  >
    {({ data, fetchMore }) => (
      <Feed
        entries={data.feed || []}
        onLoadMore={() =>
          fetchMore({
            variables: {
              offset: data.feed.length
            },
            updateQuery: (prev, { fetchMoreResult }) => {
              if (!fetchMoreResult) return prev;
              return Object.assign({}, prev, {
                feed: [...prev.feed, ...fetchMoreResult.feed]
              });
            }
          })
        }
      />
    )}
  </Query>
);
```
本段代码的上下文[请戳这里](https://github.com/apollographql/GitHunt-React/blob/e5d5dc3abcee7352f5d2e981ee559343e361d2e3/src/routes/FeedPage.js#L26-L68)。

正如你所看到的那样，可以通过render prop函数访问`fetchMore`。 默认情况下，`fetchMore`将使用原始`query`，因此我们只传入新变量。 从服务器返回新数据后，`updateQuery`函数用于将其与现有数据合并，这将会使扩展列表组件重新渲染UI。

需要注意的是，为了让UI组件在调用`fetchMore`之后接收更新的`loading`prop，一定要在Query组件的props中将`notifyOnNetworkStatusChange`设置为`true`。

## 基于游标的分页（Cursor-based）
在基于游标的分页中，“cursor”用于跟踪数据集中应该从哪里获取下一个具体的分页项。光标有时在一些场景下可以非常简单，只需要引用最后一个对象的ID，但在某些情况下——例如根据具体条件排序的列表，除了提取的最后一个对象的ID之外，游标（cursor）还需要对排序规则进行进一步编码处理。

在客户端上实现基于游标的分页与基于偏移的分页并没有什么不同，但是我们的做法是保留对最后获取对象的引用，以及使用排序顺序的信息，而不是使用绝对偏移。

在下面的例子中，我们使用`fetchMore`查询来连续加载新的评论，这些评论将被添加到列表中。 在查询中`fetchMore`使用的游标由服务器初始响应提供，并且每当获取更多数据时都会更新。

```js
const MoreCommentsQuery = gql`
  query MoreComments($cursor: String) {
    moreComments(cursor: $cursor) {
      cursor
      comments {
        author
        text
      }
    }
  }
`;

const CommentsWithData = () => (
  <Query query={CommentsQuery}>
    {({ data: { comments, cursor }, loading, fetchMore }) => (
      <Comments
        entries={comments || []}
        onLoadMore={() =>
          fetchMore({
            //注意，这里与Query组件中使用的查询不同
            query: MoreCommentsQuery,
            variables: { cursor: cursor },
            updateQuery: (previousResult, { fetchMoreResult }) => {
              const previousEntry = previousResult.entry;
              const newComments = fetchMoreResult.moreComments.comments;
              const newCursor = fetchMoreResult.moreComments.cursor;

              return {
                // 通过返回 `cursor`, 将 `fetchMore` 更新为 cursor.
                cursor: newCursor,
                entry: {
                  // 将新的评论添加到先前的列表中
                  comments: [...newComments, ...previousEntry.comments]
                },
                __typename: previousEntry.__typename
              };
            }
          })
        }
      />
    )}
  </Query>
);

```

## 中继式游标分页（Relay-style cursor pagination）

另一个流行的GraphQL客户端Relay对分页查询的输入和输出有着自己的见解，因此开发者有时需要根据Relay的需求构建服务器的分页模型。如果你有一个根据[Relay Cursor Connections](https://facebook.github.io/relay/graphql/connections.htm)规范设计的服务器，你从Apollo客户端调用该服务器也是没有问题。

使用中继式游标分页与基于游标的基本分页非常相似。 主要区别在于查询响应的格式，它会影响获取游标的位置。

Relay在返回游标的连接上提供了一个`pageInfo`对象，它包含第一项和最后一项的游标，分别对应的属性是`startCursor`和`endCursor`。 该对象还包含一个布尔属性`hasNextPage`，可用于确定是否还有更多的数据。

下面的例子一次指定 10 个项的请求，请求应该在`cursor`之后开始计算返回结果。 如果传了`null`，游标中继将忽略它，并从数据集的开头计算返回结果，允许对初始和后续请求使用相同的查询。

```js
const CommentsQuery = gql`
  query Comments($cursor: String) {
    Comments(first: 10, after: $cursor) {
      edges {
        node {
          author
          text
        }
      }
      pageInfo {
        endCursor
        hasNextPage
      }
    }
  }
`;

const CommentsWithData = () => (
  <Query query={CommentsQuery}>
    {({ data: { Comments: comments }, loading, fetchMore }) => (
      <Comments
        entries={comments || []}
        onLoadMore={() =>
          fetchMore({
            variables: {
              cursor: comments.pageInfo.endCursor
            },
            updateQuery: (previousResult, { fetchMoreResult }) => {
              const newEdges = fetchMoreResult.comments.edges;
              const pageInfo = fetchMoreResult.comments.pageInfo;

              return newEdges.length
                ? {
                    // 将新的评论添加到列表尾部，更新`pageInfo`
                    // 获取新的`endCursor`和`hasNextPage`值
                    comments: {
                      __typename: previousResult.comments.__typename,
                      edges: [...previousResult.comments.edges, ...newEdges],
                      pageInfo
                    }
                  }
                : previousResult;
            }
          })
        }
      />
    )}
  </Query>
);

```

## `@connection`指令

使用分页查询时，在store中很难找到累积的查询结果，传递给查询的参数用于确定默认store的key值，但执行查询之外代码段通常是不知道这些参数的。 这对必要的store更新来说是有问题的，因为没有稳定的key值用来更新这些store。 要想Apollo Client的分页查询使用稳定的store key，可以使用可选指令`@connection`为部分查询指定store key。例如，如果我们希望为先前的feed查询提供稳定的store key，可以使用`@ connection`指令来调整查询：
```js
const FEED_QUERY = gql`
  query Feed($type: FeedType!, $offset: Int, $limit: Int) {
    currentUser {
      login
    }
    feed(type: $type, offset: $offset, limit: $limit) @connection(key: "feed", filter: ["type"]) {
      id
      # ...
    }
  }
`;
```
这会使每个查询中累积的feed或`fetchmore`放在store的`feed` key下，稍后我们可以使用它进行必要的store更新。在本例中，我们还使用了`@connection`指令的`filter`参数(可选)，它允许在store key中包含查询的一些参数。例如本例，我们希望在store key中包含`type`查询参数，这将出现多个store的值，这些值从不同类型的feed中叠加页数。
