# go-restful


## go-restful 是干什么的
kubernetes 里的 apiserver 使用这个框架向外提供 rest 的接口。

## Quick start
> go-restful 的使用方式  
> &emsp;&emsp;主要包含三种对象:
> - Container: 一个Container包含多个WebService  
> - WebService: 一个WebService包含多条route
> - Route: 一条route包含一个method(GET、POST、DELETE等)，一条具体的path(URL)以及一个响应的handler function

使用很简单，看看例子就可以了，去这里：[example](https://github.com/emicklei/go-restful/blob/master/examples/restful-user-resource.go)