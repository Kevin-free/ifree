---
Title: Go 的参数传递
Title: Explain parameter passing in Go
Created Date: 2023-02-07T10:24:36.983Z
Created Date: 2023-02-08T10:24:36.983Z
---



# Go 的参数传递|传值？传引用？

> 此处温馨复习一下 `&`和 `*` ：
>
> 1. `&` 是取地址符号 , 即取得某个变量的地址 , 如：&a
> 2.  `*` 是指针运算符 , 可以表示一个变量是指针类型 , 也可以表示一个指针变量所指向的存储单元 , 也就是这个地址所存储的值



## 传值

**传值，也叫做值传递（pass by value）。指的是在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。**

通俗的讲，值传递，所传递的是该参数的副本，本质上不能认为是一个东西，因为**指向的不是同一个内存地址**。

我们可以写个示例清楚的演示：

```go
func TestPassValue(t *testing.T) {
	s := "Kevin爱学习"
	fmt.Printf("TestPassValue 中s的内存地址是：%p\n", &s)
	modify(s)
}
func modify(s string) {
	fmt.Printf("modify 中s的内存地址是：%p\n", &s)
}
```

输出结果如下：

```go
TestPassValue 中s的内存地址是：0xc00004e620
modify 中s的内存地址是：0xc00004e630
```

对于这类基础类型我们很好理解，它们就是一个拷贝，但是指针呢？我们稍微改一下这个例子：

```go
func TestPassValue(t *testing.T) {
	s := "Kevin爱学习"
	sp := &s
	fmt.Printf("TestPassValue 指针sp的内存地址：%p\n", &sp)
	modify(sp)
}
func modify(sp *string) {
	fmt.Printf("modify 指针sp的内存地址：%p\n", &sp)
}
```

输出结果如下：

```
TestPassValue 指针sp的内存地址：0xc00004e620
modify 指针sp的内存地址：0xc000006038
```

我们可以看到在 TestPassValue 函数中的变量 s 所指向的内存地址是 `0xc00004e620`。在经过 modify函数的参数传递后，其在内部所输出的内存地址是 `0xc000006038`，两者发生了改变。



![](http://wesub.ifree258.top/passByValue_001.png)



由此我们可以得出结论，**在 Go 语言中确实都是值传递**。那是不是在函数内修改值，就不会影响到实际参数呢？

我们在上一个案例中修改便知：

```go
func TestPassValue(t *testing.T) {
	s := "Kevin爱学习"
	fmt.Printf("TestPassValue s的内存地址：%p\n", &s)
	sp := &s
	fmt.Printf("TestPassValue 指针sp的内存地址：%p\n", &sp)
	modify(sp)
	fmt.Println(s)
}
func modify(sp *string) {
	fmt.Printf("modify 指针sp的内存地址：%p\n", &sp)
	*sp = "Kevin爱健身"
}
```

我们在 modify 函数中修改了变量 s 的值，那么最终在 TestPassValue 函数中我们输出的变量 s 值是 "Kevin爱学习" 还是 "Kevin爱健身" ？

来看结果：

```
TestPassValue s的内存地址：0xc000088610
TestPassValue 指针sp的内存地址：0xc0000c0028
modify 指针sp的内存地址：0xc0000c0030
Kevin爱健身
```

输出的是 "Kevin爱健身" ，这个时候你可能迷糊了，前面明明说了“**Go 语言只有值传递，这样在函数中如果对参数进行修改，将不会影响到实际参数**”，也验证了两者的内存地址确实不一样，为什么实际参数随之改变了呢？

因为“**如果传过去的值是指向内存空间的地址，那么是可以对这块内存空间做修改的**”。

也就是说，这两个内存地址，其实是指针的指针，其根源都指向同一个指针，也就是指着变量 s。因此修改变量 s，实际参数也变成了相应的值。



![](http://wesub.ifree258.top/passByValue_002.png)



通过上面的图，可以更好的理解。首先我们声明了一个变量`s`，值为`"Kevin爱学习"`，它的内存存放的地址为`0xc000088610`，通过这个内存地址，我们可以找到变量`s`，这个内存地址也就是变量`s`的指针`sp`。

指针`sp`也是一个指针类型的变量，它也需要内存存放，它的内存地址是`0xc0000c0028`。在我们传递指针变量`sp`给`modify`函数的时候，是该指针变量的拷贝，所以新拷贝的指针变量`sp`，它的内存地址已经变了，是新的`0xc0000c0030`。

不管是`0xc0000c0028`还是`0xc0000c0030`，我们都可以称之为指针的指针，它们指向同一个指针`0xc000088610`，这个`0xc000088610`又指向变量`s`，这也就是为什么我们可以修改变量`s`的值。



## 传引用

传引用，也叫作引用传递（pass by reference），**指在调用函数时将实际参数的地址直接传递到函数中**，那么在函数中对参数所进行的修改，将影响到实际参数。

**在 Go 语言中是没有引用传递的**，这里我不能使用Go举例子，但是可以通过说明描述。

以上面的例子为例，如果在`modify`函数里打印出来的内存地址是不变的，也是`0xc0000c0028`，那么就是引用传递。



## 引用类型 map

了解传值和传引用，你可能还是会有困惑：Go 语言中的 map 和 slice 类型，能直接修改，难道不是同个内存地址？不是引用？

比如下面这个例子：

```go
func TestTransMap(t *testing.T) {
	persons := make(map[string]string)
	persons["kevin"] = "这次一定！"
	fmt.Printf("TestTransMap map的内存地址是：%p\n", &persons)
	modifyMap(persons)
	fmt.Println(persons)
}
func modifyMap(p map[string]string) {
	fmt.Printf("modifyMap map的内存地址是：%p\n", &p)
	p["kevin"] = "记得点赞！"
}
```

输出结果：

```
TestTransMap map的内存地址是：0xc000006038
modifyMap map的内存地址是：0xc00000604
```

确实是值传递，那修改后的 map 的结果应该是什么。既然是值传递，那肯定就是 "这次一定！"，对吗？

输出结果：

```
map[kevin:记得点赞！]
```

结果是“记得点赞！“

两个内存地址是不一样的，所以这又是一个值传递（值的拷贝），那么为什么我们可以修改Map的内容呢？先不急，我们先看一个自己实现的`struct`。

```go
func TestTransStruct(t *testing.T) {
	p := Activity{"摸鱼"}
	fmt.Printf("TestTransStruct Activity的内存地址是：%p\n", &p)
	modifyStruct(p)
	fmt.Println(p)
}
type Activity struct {
	Name string
}
func modifyStruct(activity Activity) {
	fmt.Printf("modifyStruct Activity的内存地址是：%p\n", &activity)
	activity.Name = "学习"
}
```

输出结果：

```
TestTransStruct Activity的内存地址是：0xc00004e640
modifyStruct Activity的内存地址是：0xc00004e650
```

可以看到确实也是值传递，那你觉得结果是“摸鱼”还是“学习”呢？

输出结果：

```
{摸鱼}
```

结果还是“摸鱼”，这是为什么呢？

我们自己定义的`Activity`类型，在函数传参的时候也是值传递，但是它的值(`Name`字段)并没有被修改，我们想改成`学习`，发现最后的结果还是`摸鱼`。

这也就是说，`map`类型和我们自己定义的`struct`类型是不一样的。我们尝试把`modifyStruct`函数的接收参数改为`Activity`的指针。

```go
func TestTransStruct(t *testing.T) {
	p := Activity{"摸鱼"}
	fmt.Printf("TestTransStruct Activity的内存地址是：%p\n", &p)
	modifyStruct(&p)
	fmt.Println(p)
}
type Activity struct {
	Name string
}
func modifyStruct(activity *Activity) {
	fmt.Printf("modifyStruct Activity的内存地址是：%p\n", activity)
	activity.Name = "学习"
}
```

输出结果：

```
TestTransStruct Activity的内存地址是：0xc00004e640
modifyStruct Activity的内存地址是：0xc00004e640
{学习}
```

可以看到这次被修改了。 指针类型可以修改，非指针类型不行，那么我们可以大胆的猜测，我们使用`make`函数创建的`map`是不是一个指针类型呢？看一下源代码:

```go
//src\runtime\map.go

// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
    //省略无关代码
}
```

我使用的是 Go 1.14，通过查看`src\runtime\map.go` 源码发现，`make`函数返回的确实是一个 `hmap`类型的指针 `*hmap`。也就是说 `map`即 `*hmap`。所以`func modifyMap(p map)` 这个函数其实就等于 `func modifyMap(p *hmap)`，这就和我们前面`func modify(sp *string)` 的例子一样可以理解了。

所以，map 的例子可以如下理解：

```go
func TestTransMap(t *testing.T) {
	persons := make(map[string]string) // map === *hmap
	fmt.Printf("TestTransMap map的内存地址是：%p\n", persons) // * 指针即地址，不需要 &
	persons["kevin"] = "这次一定！"
	fmt.Printf("TestTransMap 指向map的指针地址是：%p\n", &persons)
	modifyMap(persons)
	fmt.Println(persons)
}
func modifyMap(p map[string]string) {
	fmt.Printf("modifyMap 指向map的指针地址是：%p\n", &p)
	p["kevin"] = "记得点赞！"
}
```

输出结果：

```
TestTransMap map的内存地址是：0xc000066540
TestTransMap 指向map的指针地址是：0xc000006038
modifyMap 指向map的指针地址是：0xc000006040
map[kevin:记得点赞！]
```

其内存关系图如下：



![](http://wesub.ifree258.top/passByValue_003.png)



所以，Go语言通过`make`函数，字面量的包装，为我们省去了指针的操作，让我们可以更容易的使用map。这里的`map`可以理解为引用类型，但是记住**引用类型不是传引用。**



在 Go 语言中，与 `map` 类型类似的还有 `chan` 类型：

```go
func makechan(t *chantype, size int64) *hchan {
    //省略无关代码
}
```

`chan` 也是一个引用类型，和 `map` 相差无几，这里就不多介绍了。



## 和map、chan都不一样的slice

`slice`和`map`、`chan`都不太一样的，一样的是，它也是引用类型，它也可以在函数中修改对应的内容。

```go
func TestTransSlice(t *testing.T) {
	ages := []int{6, 6, 6}
	fmt.Printf("原始slice的内存地址是：%p\n", ages)
	modifySlice(ages)
	fmt.Println(ages)
}
func modifySlice(ages []int) {
	fmt.Printf("函数里接收到slice的内存地址是：%p\n", ages)
	ages[0] = 1
}
```

输出结果：

```
原始slice的内存地址是：0xc0000921a0
函数里接收到slice的内存地址是：0xc0000921a0
[1 6 6]
```

从结果来看，两者的内存地址一样，也成功的变更到了变量`ages` 的值。这难道不是引用传递吗？

注意这两个细节：

- 没有用 `&` 来取地址。
- 可以直接用 `%p` 来打印。

之所以可以同时做到上面这两件事，是因为标准库 `fmt` 针对在这一块做了优化：

```go
// src\fmt\print.go

func (p *pp) fmtPointer(value reflect.Value, verb rune) {
	var u uintptr
	switch value.Kind() {
	case reflect.Chan, reflect.Func, reflect.Map, reflect.Ptr, reflect.Slice, reflect.UnsafePointer:
		u = value.Pointer()
	default:
		p.badVerb(verb)
		return
	}
	//省略部分代码
}
```

通过源代码发现，对于`chan`、`map`、`slice`等被当成指针处理，通过`value.Pointer()`直接对应的值的指针地址，当然就不需要取地址符了。

```go
// If v's Kind is Slice, the returned pointer is to the first
// element of the slice. If the slice is nil the returned value
// is 0.  If the slice is empty but non-nil the return value is non-zero.
func (v Value) Pointer() uintptr {
	// TODO: deprecate
	k := v.kind()
	switch k {
	//省略无关代码
	case Slice:
		return (*SliceHeader)(v.ptr).Data
	}
}
```

很明显了，当是`slice`类型的时候，返回是`slice`这个结构体里，字段Data第一个元素的地址。

```go
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

所以我们通过`%p`打印的`slice`变量`ages`的地址其实就是内部存储数组元素的地址，`slice`是一种结构体+元素指针的混合类型，通过元素`array`(`Data`)的指针，可以达到修改`slice`里存储元素的目的。

所以修改类型的内容的办法有很多种，类型本身作为指针可以，类型里有指针类型的字段也可以。

单纯的从`slice`这个结构体看，我们可以通过`modifySlice`修改存储元素的内容，但是永远修改不了`len`和`cap`，**因为他们只是一个拷贝，如果要修改，那就要传递`*slice`作为参数才可以。**



## 避坑

最后分享一个开发过程中发现的别人的小错误：

```go
type StInfo struct {
	name string
}

func TestChangeInfo(t *testing.T) {
	info := &StInfo{}
	changeInfo(info)
	fmt.Printf("TestChangeInfo info:%v \n", info)
}

func changeInfo(info *StInfo) {
	newInfo := &StInfo{
		name: "newInfo",
	}
	info = newInfo
}
```

你觉得最后的输出结果是什么呢？

```
TestChangeInfo info:&{} 
```

那你知道是为什么吗？

是的，`info = newInfo` 这里写错了，这样写是 info 指针的修改，只是此方法中指向了新值，原info不会变。

正确写法应该是 `*info = *newInf ` 这样就是 info 指针对应的值修改，改变 info 指针中的值，原info会变。

细节决定成败！

同样可以打印指针地址，画出图方便理解：



![](http://wesub.ifree258.top/passByValue_004.png)

## 小结

最终我们可以确定，Go 语言中所有的传参都是传值（值传递），都是一个拷贝的副本。因为拷贝的内容有时候是非引用类型（int，string，struct等这些），这样在函数中就无法修改原内容数据；有的是引用类型（指针，map，slice，chan等这些），这样在函数中就可以修改原内容数据。

是否可以修改原内容数据，和传值、传引用没有必然的关系。在C++中，传引用肯定是可以修改原内容数据的，在Go语言里，虽然只有传值，但是我们也可以修改原内容数据，因为参数是引用类型。

这里也要记住，**引用类型和传引用是两个概念。**

结论：Go 中只有传值！Go 中只有传值！Go 中只有传值！



## Reference

[又吵起来了，Go 是传值还是传引用？](https://mp.weixin.qq.com/s/qsxvfiyZfRCtgTymO9LBZQ)

[Go语言参数传递是传值还是传引用](https://www.flysnow.org/2018/02/24/golang-function-parameters-passed-by-value.html)

