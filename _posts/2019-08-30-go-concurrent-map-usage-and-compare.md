---
layout: post
title: 谈Go语言中并发Map的使用
date: 2019-06-19T11:32:41+00:00
author: mingkai
permalink: /2019/08/go-concurrent-map-usage-and-compare
categories:
  - golang
---


#### 谈Go语言中并发Map的使用

------

最近开发Go语言总是遇到哈希表的使用，在高并发下如何保证读写的安全性尤为重要，假如不了解的情况下，使用原生map的话，性能倒是很高，但在多个goroutine操作下就会遇到并发读写的错误出现。为了并发安全，修改读写访问，每次都写都加入读写锁，又会导致性能的大幅度下降，安全和性能实在是难以同时兼得。

这里我们梳理下Go当前访问Map的几种方式，并给出实际的测试实例和性能表现。



#### 1. 标准库map结构

map是go语言标准库中的哈希表实现，不是并发安全的，也就是在多个goroutine同时访问的时候存在数据并发访问冲突，导致程序异常。适合单一的goroutine中使用，性能也是所有所有哈希表实现中最好的，

同时在多个goroutine读取或者使用for循环遍历相同的map也是并发安全的，只要不涉及更新即可高效的访问map结构。

设计的测试实例如下面所示：

```go
const (
	MapSize     = 10000
	LoopCounter = 1000
)

func BenchmarkMap(b *testing.B) {
  // 初始化数据结构
	m := map[string]int{}
	keys := []string{}
	for i := 0; i < MapSize; i++ {
		keys = append(keys, strconv.Itoa(i))
		m[keys[i]] = i
	}
  // 重置测试计时器
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		for i := 0; i < LoopCounter; i++ {
			if _, ok := m[keys[i]]; ok {
			}
		}
	}
}
func BenchmarkMapWrite(b *testing.B) {
	m :=...
  ...
	for n := 0; n < b.N; n++ {
		for i := 0; i < LoopCounter; i++ {
			m[keys[i]] = i
		}
	}
}

func BenchmarkMapReadAndWrite(b *testing.B) {
	m :=...
  ...
	for n := 0; n < b.N; n++ {
		for i := 0; i < LoopCounter; i++ {
			if d, ok := m[keys[i]]; ok == true {
				m[keys[i]] = d + 1
			}
		}
	}
}
```

实际执行的情况如下面所示, 可以看到实际读取和写入的时间差不多，我们这里以读写1000次为一个单元，因此单一读取的操作的时间是25ns左右，也就是每秒可以执行4000万次的操作，写入操作为30ns左右，每秒执行3000多万次。测试中读写操作包含了1000次读取和1000次写入操作。

```
BenchmarkMapRead-4           	   50000	     25804 ns/op	       0 B/op	       0 allocs/op
BenchmarkMapWrite-4          	   50000	     31337 ns/op	       0 B/op	       0 allocs/op
BenchmarkMapReadAndWrite-4   	   30000	     47668 ns/op	       0 B/op	       0 allocs/op
```

假如我们测试并发的读取和写入操作，可以通过启动多个goroutine来测试, 下面的环境下我们启动8个goroutine同时去读取和写入这个map对象，实际执行情况肯定是会直接报错退出。

```go
func BenchmarkMapReadAndWriteWithMutilGoroutine(b *testing.B) {
	m :=...
  ...
	wg := sync.WaitGroup{}
	for i := 0; i < 8; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for n := 0; n < b.N; n++ {
				for i := 0; i < LoopCounter; i++ {
					if d, ok := m[keys[i]]; ok == true {
						m[keys[i]] = d + 1
					}
				}
			}
		}()
	}
	wg.Wait()
}
// 执行出错的原生map的并发读写
// fatal error: concurrent map read and map write
```

#### 2.自定义加锁的map结构

我们可以简单的封装下map结构使其实现安全的并发访问，并实现一些基本的读取，写入和删除接口操作。使用一个全局的读写锁，当数据读取和写入的时候分别执行锁操作，保证不会出现读写冲突的问题。缺点就是，当我们的map数据写操作较多的情况，会导致效率较低。一个写操作将导致全局map的后续读取和写入都需要等待，直到释放该锁。

实现和测试的代码如下：

```go
type CustomIntMap struct {
	sync.RWMutex
	internal map[string]int
}

func NewCustomIntMap() *CustomIntMap {
	return &CustomIntMap{
		internal: make(map[string]int),
	}
}

func (rm *CustomIntMap) Load(key string) (value int, ok bool) {
	rm.RLock()
	result, ok := rm.internal[key]
	rm.RUnlock()
	return result, ok
}

func (rm *CustomIntMap) Delete(key string) {
	rm.Lock()
	delete(rm.internal, key)
	rm.Unlock()
}

func (rm *CustomIntMap) Store(key string, value int) {
	rm.Lock()
	rm.internal[key] = value
	rm.Unlock()
}
```

上面的读写锁，用于不同的接口使用，当我们仅仅读取的时候，仅需要执行读取锁，不影响其他goroutine的读取执行，性能损耗不大，全局锁则会阻止其他goroutine读写的执行

```go
func BenchmarkCustomMapRead(b *testing.B) {
	m := NewCustomIntMap()
	keys := []string{}
	for i := 0; i < MapSize; i++ {
		keys = append(keys, strconv.Itoa(i))
		m.Store(keys[i], i)
	}
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		for i := 0; i < LoopCounter; i++ {
			if _, ok := m.Load(keys[i]); ok {

			}
		}
	}
}

func BenchmarkCustomMapWrite(b *testing.B) {
	m := NewCustomIntMap()
	keys := []string{}
	for i := 0; i < MapSize; i++ {
		keys = append(keys, strconv.Itoa(i))
		m.Store(keys[i], i)
	}
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		for i := 0; i < LoopCounter; i++ {
			m.Store(keys[i], i+1)
		}
	}
}

func BenchmarkCustomMapReadAndWrite(b *testing.B) {
	m := NewCustomIntMap()
	keys := []string{}
	for i := 0; i < MapSize; i++ {
		keys = append(keys, strconv.Itoa(i))
		m.Store(keys[i], i)
	}
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		for i := 0; i < LoopCounter; i++ {
			if _, ok := m.Load(keys[i]); ok {
				m.Store(keys[i], i+1)
			}
		}
	}
}
```

执行测试的结果如下所示, 可以看到相对于使用第三方库，这种方式明显可以改善程序的运行效率，相对于concurrent-map的性能提升约1/3左右

```
BenchmarkCustomMapRead-4   	           30000	     53241 ns/op	       0 B/op	       0 allocs/op
BenchmarkCustomMapWrite-4   	         20000	     70543 ns/op	       0 B/op	       0 allocs/op
BenchmarkCustomMapReadAndWrite-4   	   20000	     96799 ns/op	       0 B/op	       0 allocs/op

```

执行多个goroutine的时候，读取和写入操作的时间进行测试。但是对于多个goroutine下的读写操作，自定义加锁map的性能却不如concurrent-map的效率。

```go
func BenchmarkCustomMapReadAndWriteWithMutilGoroutine(b *testing.B) {
	m := NewCustomIntMap()
	keys := []string{}
	for i := 0; i < MapSize; i++ {
		keys = append(keys, strconv.Itoa(i))
		m.Store(keys[i], i)
	}
	b.ResetTimer()
	wg := sync.WaitGroup{}
	for i := 0; i < 2; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for n := 0; n < b.N; n++ {
				for i := 0; i < LoopCounter; i++ {
					if _, ok := m.Load(keys[i]); ok {
						m.Store(keys[i], i+1)
					}
				}
			}
		}()
	}
	wg.Wait()

}

```

执行发现自定义的安全锁map在多个goroutine下的读取操作和

```
BenchmarkCustomMapReadAndWriteWithMutilGoroutine-4   	    5000	    338891 ns/op	       0 B/op	       0 allocs/op
PASS
```



####  3. 第三方库concurrent-map

1.9版本之前，在官方还未推出自己的sync.Map并发哈希表结构之前，concurrent-map被很多人使用作为可选的并发安全的读写map库。与直接加锁相比，concurrent-map通过使用分片的方式将，存储空间分为几个片段，每个片段包含一部分数据，这样我们在写的时候不需要讲整个的结构都进行加锁，阻塞新的写入或者读取操作。

问题在于每次读取和写入都需要额外的计算分片信息。同时并发写相同分片的key会导致效率更低。同时每次读写内部还需要进行类型的转换，因为concurrent-map存储的类型为interface结构类型。

提供基本的读取和设置以及删除操作。这里我们使用同样的方式对于读写性能进行测试, 这里只贴出读写的测试代码：

```go
func BenchmarkConcurrentMapReadWrite(b *testing.B) {
	m := cmap.New()
	keys := []string{}
	for i := 0; i < MapSize; i++ {
		keys = append(keys, strconv.Itoa(i))
		m.Set(keys[i], i)
	}

	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		for i := 0; i < LoopCounter; i++ {
			if _, ok := m.Get(keys[i]); ok {
				m.Set(keys[i], i+1)
			}

		}
	}

```

测试可以看出来，实际执行的读写操作时间性能相对于原生的map基本上将近3x的性能下降。 Concurrent-map的实现上面也是采用了锁的方式，作为第三方开源产品，很多情况下为了兼容不同的存取实现，内部会有一些额外的操作比如类型转换，而这些都可能导致程序性能的下降。

```
BenchmarkConcurrentMapRead-4   	         20000	     63583 ns/op	       0 B/op	       0 allocs/op
BenchmarkConcurrentMapWrite-4   	       10000	    121971 ns/op	    8000 B/op	    1000 allocs/op
BenchmarkConcurrentMapReadWrite-4   	   10000	    160574 ns/op	    8000 B/op	    1000 allocs/op
```

针对并发操作的测试如下面所示，启动两个goroutine来同时执行读写操作，

```go
func BenchmarkConcurrentMapReadAndWriteWithMutilGoroutine(b *testing.B) {
	m := cmap.New()
	keys := []string{}
	for i := 0; i < MapSize; i++ {
		keys = append(keys, strconv.Itoa(i))
		m.Set(keys[i], i)
	}
	b.ResetTimer()
	wg := sync.WaitGroup{}
	for i := 0; i < 2; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for n := 0; n < b.N; n++ {
				for i := 0; i < LoopCounter; i++ {
					if _, ok := m.Get(keys[i]); ok {
						m.Set(keys[i], i+1)
					}
				}
			}
		}()
	}
	wg.Wait()
}
```



测试结果如下面所示, 两个goroutine,执行读写操作实际执行结果每个内部的操作读写约为280ns/2 = 140ns基本上在多个goroutine上与上面的基本一致。

```go
BenchmarkConcurrentMapReadAndWriteWithMutilGoroutine-4   	    5000	    280993 ns/op	   16000 B/op	    2000 allocs/op
```



#### 4. sync.Map

sync.Map结构是在1.9版本后加入到标准库中，提供给用户使用的并发安全锁机制，实现上通过两个map结构，其中一个只读的map，另外一个通过RWLocker锁保护的读写map组成。Go开发人员在github上提到过sync.Map的实现和解决的问题，提到sync.Map主要用于解决服务器上由于CPU核心过多导致的缓存争夺（多CPU更新同一个缓存变量的情况下，导致效率降低）的问题，这个问题可能导致对于一个O(1)操作，在多个核心（N核）同时操作的时候，可能导致变为O(N)的操作。

![image-20190902093538979](/Users/minkaizhang/Library/Application Support/typora-user-images/image-20190902093538979.png)



适合仅新增key的情况，且读取数据的比例远大于更新数据的总量。

同时也提到了类似于concurrent-map中多个分片锁的实现，对于并发读相同分片下可能导致对于分片锁资源的争夺问题。另外分片结构和锁策略也可能导致程序运行的[其他问题](https://github.com/golang/go/issues/17953)。

我们使用通用的测试方法测试, 如下所示为程序的读写测试，和并发读写操作测试代码。

```go
func BenchmarkSyncMapReadAndWrite(b *testing.B) {
	var m sync.Map

	keys := []string{}
	for i := 0; i < MapSize; i++ {
		keys = append(keys, strconv.Itoa(i))
		m.Store(keys[i], i)
	}

	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		for i := 0; i < LoopCounter; i++ {
			if _, ok := m.Load(keys[i]); ok {
				m.Store(keys[i], i+1)
			}
		}
	}
}

func BenchmarkSyncMapReadAndWriteWithMutilGoroutine(b *testing.B) {
	var m sync.Map

	keys := []string{}
	for i := 0; i < MapSize; i++ {
		keys = append(keys, strconv.Itoa(i))
		m.Store(keys[i], i)
	}

	b.ResetTimer()
	wg := sync.WaitGroup{}
	for i := 0; i < MaxGoRoutineNumber; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for n := 0; n < b.N; n++ {
				for i := 0; i < LoopCounter; i++ {
					if _, ok := m.Load(keys[i]); ok {
						m.Store(keys[i], i+1)
					}
				}
			}
		}()
	}
	wg.Wait()
}

```

测试的结果如下面所示：

```
BenchmarkSyncMapRead-4                                   	   30000	     61689 ns/op	       0 B/op	       0 allocs/op
BenchmarkSyncMapWrite-4                                  	   10000	    247273 ns/op	   40000 B/op	    3000 allocs/op
BenchmarkSyncMapReadAndWrite-4                           	    5000	    271561 ns/op	   40000 B/op	    3000 allocs/op
BenchmarkSyncMapReadAndWriteWithMutliGoroutine-4         	    5000	    331194 ns/op	   80000 B/op	    6000 allocs/op
PASS
```



#### 总结

通过对比各种不同的实现，可以看出：

- 读写方面原生map性能最好，其他的加锁的版本和sync.Map均会有2-3倍左右的性能下降 
  - 根据测试的性能读取排序大概是：map>自定义加锁map>concurrent-map>sync.Map, 总体上可以性能损耗相对map有2-3倍的下降
  - 根据测试的性能写入排序大概是：map >自定义加锁map>concurrent-map>sync.Map,其中sync.Map的写操作下降严重约5-6倍
  - 同时读写操作下，基本上和上面的顺序一致，自定义的锁约原生map的两倍左右，concurrent-map2-3倍，sync.Map基本上达到了4倍的性能下降。
- 并发方面：对于相同的操作下，并发读写(各一半)操作则出现自定义加锁map变慢的情况
  - 2个goroutine下，concurrent-map > 自定义加锁map>sync.Map
  - 4个goroutine下，concurrent-map >自定义加锁map> sync.Map
  - 8个goroutine下，concurrent-map >自定义加锁map> sync.Map
  - 16个goroutine下，concurrent-map >自定义加锁map> sync.Map
  - 32个goroutine下，concurrent-map > sync.Map >自定义加锁map
  - 64个goroutine下，concurrent-map > sync.Map >自定义加锁map

- 并发方面：对于相同的操作下，并发读操作则出现自定义加锁map变慢的情况
  - 2个goroutine下，sync.Map > concurrent-map >自定义加锁map
  - 4个goroutine下，sync.Map > concurrent-map >自定义加锁map
  - 8个goroutine下，sync.Map > concurrent-map >自定义加锁map
  - 16个goroutine下，sync.Map > concurrent-map >自定义加锁map
  - 32个goroutine下，sync.Map > concurrent-map >自定义加锁map
  - 64个goroutine下， sync.Map > concurrent-map >自定义加锁map



可以看出，当我们不需要并发操作的时候，直接使用map更快（显而易见），但是对于并发操作的时候，则要根据实际的业务需求进行判断，如果读取操作更多，且key比较稳定的话，在多个CPU核心的条件下，而对于读写各一半的情况下concurrent-map则会更好一些。

最后附上此次测试的系统信息

```
MacOSX: 10.14.6 Darwin Kernel Version 18.7.0
CPU:    2.5 GHz Intel Core i5
Memory: 16 GB 1600 MHz DDR3
```

