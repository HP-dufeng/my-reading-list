# Factory method
> A **factory** is an object that is used to create other objects.

再次强调一下：  
**Go isn't exactly object oriented language and promotes simplicity.**  
千万不要生搬硬套设计模式的概念，应该想想这个模式想要解决的问题是什么。

我看了好多文章，实现的代码，布拉布拉的一大堆，比如这里：[factory-pattern-in-golang](https://matthewbrown.io/2016/01/23/factory-pattern-in-golang/)

其实，不就是要解决对象创建的问题吗，go 的风格是 定义一个 New 方法来创建你需要的对象，go source code 里到处都是，比如，  
errors package：
```
func New(text string) error
```
比比皆是，当然你也可以把它理解为构造函数，反正就是创建对象嘛。

重点是：  
**factory method，类似于一个 helper method，用来去创建对象，而不需要知道 class 的实现细节。**

```
    err := errors.New("emit macho dwarf: elf header corrupted")
    if err != nil {
        fmt.Print(err)
    }
```
需要一个 error 对象的时候，直接调用 New 方法就好了，不用关心 New 方法里的具体实现方式，反正它就给我返回一个 error 对象。