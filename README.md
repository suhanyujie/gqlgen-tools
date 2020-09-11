# graphql learning
graphql 学习

## note
* 在 schema.graphqls 文件中定义接口和类型
* 在 xx.model 文件中定义实际的类型，或者由 `go run github.com/99designs/gqlgen` 命令生成好类型（结构体）

## 新增接口实例
在按照 graphql 官方 [demo](https://gqlgen.com/getting-started/) 示例中，我们可以初步体会 graphql（后续简称 gql）的使用方式。

我们尝试在这个基础之上，新增一个新的接口 `queryCondition`。用于条件筛选/查询。

### 定义 schema
先在 graph/schema.graphqls 文件中定义好 schema，由于已经有了查询类型的 Query 声明，所以我们只是在 Query 内部声明一个 `queryCondition`：

```
type Query {
  todos: [Todo!]!
  "通过筛选条件查询"
  queryCondition(input: QParam!): [Todo!]!
}
```

位于双引号内部的 `"通过筛选条件查询"` 内容可以视为注释。`queryCondition` 表示方法名，方法签名的入参的参数名是 `input`紧跟其后的 `QParam!` 表示类型。
它是一个新的类型，因此我们需要定义它：

```
input QParam {
  userId: String!
}
```

`QParam` 类型的数据是用于查询的条件参数。这里简单起见，我们只实现按照 `userId` 进行精确查询，例如：查询 userId 为 `3`（注意此处的 userId 是 string 类型）的用户的 todo 列表。对应的参数就是：

```
{
    userId: "3"
}
```

回到 `queryCondition` 的声明，函数小括号后的 `[Todo!]!` 表示，返回的是 todo 列表。有没有发现，只要是类型，都要加一个 `!` 后缀。

### 查询的实现
schema 声明完后，我们运行 `go run github.com/99designs/gqlgen` 命令进行代码生成。
生成的代码主要体现在以下两个文件：
* `graph/model/models_gen.go` 文件中，会生成我们上面声明的 `QParam` 类型
* `graph/schema.resolvers.go` 文件中，会有我们声明的 `queryCondition` 占位代码

其中会多出一个尚未实现的 `QueryCondition` 方法。去掉其中的 `panic(fmt.Errorf("not implemented"))` 占位代码。接下来让我们具体实现条件查询。

```go
func (r *queryResolver) QueryCondition(ctx context.Context, input model.QParam) ([]*model.Todo, error) {
	needle := make([]*model.Todo, 0)
	for _, todo := range r.todos {
		if todo.UserID == input.UserID {
			needle = append(needle, todo)
		}
	}
	return needle, nil
}
```

我们知道，在官方的 demo 中，当我们发起一个 createTodo 请求（也可以通过 GraphQL playground 界面操作发起请求）：

```curl
curl 'http://localhost:8080/query' -H 'Accept-Encoding: gzip, deflate, br' -H 'Content-Type: application/json' -H 'Accept: application/json' -H 'Connection: keep-alive' -H 'DNT: 1' -H 'Origin: http://localhost:8080' --data-binary '{"query":"mutation createTodo {\n  createTodo(input:{text:\"todo\", userId:\"testId1\"}) {\n    user {\n      id\n    }\n    text\n    done\n  }\n}\n"}' --compressed
```

createTodo 请求发起后，后端产生一个新的 todo 实例（参考 CreateTodo 的具体实现），并且这个实例存放在 Resolver 对象（位于文件 graph/resolver.go）中，
并且 queryResolver 类型是继承于 `*Resolver`，我们的 `QueryCondition` 方法恰好是挂在 `*queryResolver` 上，所以我们可以直接获取 `queryResolver` 对象的属性：`r.todos`，
它是存放所有 todo 列表的切片。我们只需遍历它，查找符合 QParam 的筛选条件的数据：

```go
for _, todo := range r.todos {
    if todo.UserID == input.UserID {
        needle = append(needle, todo)
    }
}
```

最终将 needle 返回即可。

最后，我们从前端发起请求，查看响应值是否符合预期：

```curl
curl 'http://localhost:8080/query' -H 'Accept-Encoding: gzip, deflate, br' -H 'Content-Type: application/json' -H 'Accept: application/json' -H 'Connection: keep-alive' -H 'DNT: 1' -H 'Origin: http://localhost:8080' --data-binary '{"query":"query queryParam{\n  queryCondition(input: {userId: \"3\"}){\n    text\n    done\n    user {\n      name\n      id\n    }\n  }\n}\n"}' --compressed
```

## reference
* 官方教程 https://gqlgen.com/
* 官方仓库 https://github.com/99designs/gqlgen
