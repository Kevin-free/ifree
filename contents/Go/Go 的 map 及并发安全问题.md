---
Title: Go 的 map 及并发安全问题
Title En: Go map and concurrency security issues
Link: https://ifree.love/go-map-and-concurrency-security-issues/
Created Date: 2023-02-08T17:44:36.983Z
Updated Date: 2023-02-09T10:24:36.983Z
---

## map 是并发安全的吗？

[官方](https://golang.org/doc/faq#atomic_maps)已经提到内建的 `map` **不**是并发安全的。

在同一时间段内，让不同 goroutine 中的代码，对同一个`map`进行读写操作是不安全的。`map`值本身可能会因这些操作而产生混乱，相关的程序也可能会因此发生不可预知的问题。

我们可以用如下代码展示 `map` 的并发安全问题。

```Go
package main

func main() {
	m := make(map[int]int)

	go func() {
		for {
			_ = m[1]
		}
	}()

	go func() {
		for {
			m[2] = 2
		}
	}()

	select {}
}
```

错误信息是: `fatal error: concurrent map read and map write。`

如果你查看 Go 的源代码: [hashmap_fast.go#L118](https://github.com/golang/go/blob/master/src/runtime/hashmap_fast.go#L118),会看到读的时候会检查 `hashWriting` 标志， 如果有这个标志，就会报并发错误。

写的时候会设置这个标志: [hashmap.go#L542](https://github.com/golang/go/blob/master/src/runtime/hashmap.go#L542)

`h.flags |= hashWriting`

[hashmap.go#L628](https://github.com/golang/go/blob/master/src/runtime/hashmap.go#L628) 设置完之后会取消这个标记。

当然，代码中还有好几处并发读写的检查， 比如写的时候也会检查是不是有并发的写，删除键的时候类似写，遍历的时候并发读写问题等。

有时候，`map` 的并发问题不是那么容易被发现, 你可以利用`-race`参数来检查。

那么我们如何保证 `map` 的并发安全呢？

## 并发安全的 map 实现的三种方式

在 go 语言中实现多个 goroutine 并发安全访问修改的 `map` 的方式，主要有如下三种：

| 实现方式      | 原理                                                                                      | 使用场景                                                                 |
| ------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| map + Mutex   | 通过 Mutex 互斥锁来实现多个 goroutine 对 map 的串行访问。                                 | 读写度需要通过 Mutex 加锁和解锁，适用于读写比接近的场景                  |
| map + RWMutex | 通过 RWMutex 读写锁来实现对 map 的读写分离，从而实现读的并发性能提高。                    | 适用于读多写少的场景                                                     |
| sync.Map      | 底层通过分离读写 map 和原子指令来实现读的近似无锁，并通过延迟更新的方式来保证读的无锁化。 | 读多修改少，元素增加删除频率不高的情况，在大多数情况下替换上述两种实现。 |

## 原生 map + 锁

```Go
type counter struct {
	sync.RWMutex // 读写锁
	m map[string]int
}
```

在日常开发中， 上述这种数据结构肯定不少见，因为 golang 的原生 map 是非并发安全的，所以为了保证 map 的并发安全，最简单的方式就是给 map 加锁。

读数据的时候很方便的加锁：

```Go
counter.RLock()
n := counter.m["some_key"]
counter.RUnlock()
fmt.Println("some_key:", n)
```

写数据的时候:

```Go
counter.Lock()
counter.m["some_key"]++
counter.Unlock()
```

但是当 map 中的值越来越多，访问 map 的请求越来越多，大家都竞争这一把锁显得并发访问控制变重。

## 分段锁

我们可以将锁优化为分段锁，顾名思义就是将锁分段，将锁的粒度变小，将存储的对象分散到各个分片中，每个分片由一把锁控制，这样使得当需要对 A 分片上的数据进行读写时不会影响 B 分片的读写。

举个例子，数据存储在一栋房子里，原先只有大门有一把锁，我们要拿数据的话只能竞争大门这把锁，现在我们不对大门加锁，而对每个房间加锁，这样拿数据的时候各个房间互不影响。

### 分段锁的实现

```Go
// Map 分片
type ConcurrentMap []*ConcurrentMapShared

// 每一个Map 是一个加锁的并发安全Map
type ConcurrentMapShared struct {
	items        map[string]interface{}
	sync.RWMutex // 各个分片Map各自的锁
}
```

## sync.Map

可以说，上面的解决方案相当简洁，并且利用读写锁而不是 Mutex 可以进一步减少读写的时候因为锁带来的性能。

但是，它在一些场景下也有问题，如果熟悉 Java 的同学，可以对比一下 java 的 `ConcurrentHashMap` 的实现，在 map 的数据非常大的情况下，一把锁会导致大并发的客户端共争一把锁，Java 的解决方案是 shard, 内部使用多个锁，每个区间共享一把锁，这样减少了数据共享一把锁带来的性能影响，[orcaman](https://github.com/orcaman)提供了这个思路的一个实现： [concurrent-map](https://github.com/orcaman/concurrent-map)，他也询问了 Go 相关的开发人员是否在 Go 中也实现这种[方案](https://github.com/golang/go/issues/20360)，由于实现的复杂性，答案是 Yes, we considered it.,但是除非有特别的性能提升和应用场景，否则没有进一步的开发消息。

GO 1.9 正式加入了并发安全的字典类型 [sync.Map](https://github.com/golang/go/blob/master/src/sync/map.go)。那么，`sync.Map` 是怎么实现的呢？它是如何解决并发提升性能的呢？

`sync.Map` 的实现有几个优化点，这里先列出来，我们后面慢慢分析。

1. 空间换时间。通过冗余的两个数据结构（`read`、`dirty`），实现加锁对性能的影响。
2. 使用只读数据（`read`），避免读写冲突。
3. 动态调整，miss 次数多了之后，将 `dirty` 数据提升为 `read` 。
4. double-checking 双检查机制。
5. 延迟删除。删除一个键值只是打标记，只有在提升 `dirty` 的时候才清理删除的数据。
6. 优先从 `read` 读取、更新、删除，因为对 `read` 的读取不需要锁。

下面我们介绍 `sync.Map` 的重点代码，以便了解它的实现思想。

首先，我们看一下 `sync.Map` 的数据结构：

- Map

```Go
// Map类似于Go Map [interface{}]interface{}
// 但对于并发使用是安全的，通过多个 goroutine 不需额外的锁或同步
// 加载、存储和删除在常数时间内运行。
//
// 这个Map类型是独特的。大多数代码应该使用普通的map，
// 用单独的锁或同步，为了更好的类型安全
// 并且使他更容易维护映射内容的其他不变量。
//
// 这个Map用与优化这两种常见情况：
// 1、当一个给定的键只写一次，读多次，就像在只增长的缓存中一样；
// 2、当多个goroutines读取、写入和覆盖不相交的键时。
// 这两种情况，使用此Map相较于用Mutex或RWMutex，可以显著减少锁的争用。
//
// 初次使用之后不能复制。
// （笔者注：因为当sync.Map被拷贝之后，Map类型的dirty还是那个map，但是read和锁却不是之前的（不在同一世界你拿什么保护我），所以必然导致并发不安全）
type Map struct {
	// 当涉及到dirty数据的操作的时候，需要使用这个锁
	mu Mutex
	// 一个只读的数据结构，因为只读，所以不会有读写冲突。
	// 所以从这个数据中读取总是安全的。
	// 实际上，实际也会更新这个数据的entries,如果entry是未删除的(unexpunged), 并不需要加锁。如果entry已经被删除了，需要加锁，以便更新dirty数据。
	read atomic.Value // readOnly
	// dirty数据包含当前的map包含的entries,它包含最新的entries(包括read中未删除的数据,虽有冗余，但是提升dirty字段为read的时候非常快，不用一个一个的复制，而是直接将这个数据结构作为read字段的一部分),有些数据还可能没有移动到read字段中。
	// 对于dirty的操作需要加锁，因为对它的操作可能会有读写竞争。
	// 当dirty为空的时候， 比如初始化或者刚提升完，下一次的写操作会复制read字段中未删除的数据到这个数据中。
	dirty map[interface{}]*entry
	// 当从Map中读取entry的时候，如果read中不包含这个entry,会尝试从dirty中读取，这个时候会将misses加一，
	// 当misses累积到 dirty的长度的时候， 就会将dirty提升为read,避免从dirty中miss太多次。因为操作dirty需要加锁。
	misses int
}
```

它的数据结构很简单，值包含四个字段：`read`、`mu`、`dirty`、`misses`。

它使用了冗余的数据结构 `read` 和 `dirty`。`dirty` 中回包含 `read` 中未删除的 `entires`，新增加的 `entires` 会加入 `dirty` 中。

## 参考

[Go sync.Map](https://github.com/golang/go/blob/master/src/sync/map.go)

[golang-map 是线程安全的吗？](https://studygolang.com/articles/23184)

[golang 并发安全 Map 以及分段锁的实现](https://segmentfault.com/a/1190000018448064)

[Go 1.9 sync.Map 揭秘](https://colobu.com/2017/07/11/dive-into-sync-Map/#sync-Map)

[图解 Go 里面的 sync.Map 了解编程语言核心实现源码](https://www.codenong.com/j5e08df68f265da33a41/)
