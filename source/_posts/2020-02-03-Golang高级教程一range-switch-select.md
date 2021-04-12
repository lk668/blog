---
title: Golang高级教程(一)for-range, switch, select使用
date: 2020-02-03
tags:
    - golang
---

本文简单介绍一下golang中，for-range，switch，select的使用。

<!-- more -->

# 1. for-range使用

## 1.1 for-range地址的坑

for-range的坑
```go
package main

import "fmt"

type Test struct {
	Name string
	Age  int
}

func main() {
	testList := []Test{
		{"test1", 10},
		{"test2", 11},
		{"test3", 12},
		{"test4", 13},
	}

	var newTestList []*Test

	for _, v := range testList {
		newTestList = append(newTestList, &v)
	}

	for _, v := range newTestList {
		fmt.Println(v)
	}
}
```
上述代码是想将一个数组元素的地址，存到另外一个数组。然而实际的结果输出如下所示，
```bash
➜  ~ go run main.go
&{test4 13}
&{test4 13}
&{test4 13}
&{test4 13}
```
从结果看，与我们预期的不一样，这是什么原因呢？稍等，我稍微改一下代码：

```go
package main

import "fmt"

type Test struct {
	Name string
	Age  int
}

func main() {
	testList := []Test{
		{"test1", 10},
		{"test2", 11},
		{"test3", 12},
		{"test4", 13},
	}

	var newTestList []*Test

	for _, v := range testList {
        tmpVal := v //添加这一行code
		newTestList = append(newTestList, &tmpVal)
	}

	for _, v := range newTestList {
		fmt.Println(v)
	}
}
```
结果如下：
```bash
➜  ~ go run main.go
&{test1 10}
&{test2 11}
&{test3 12}
&{test4 13}
```
这次发现结果如预期了。
现在来解释一下原因：在for range中，变量v只是被声明了一次，每次迭代的值，都赋值给了v，而该变量的地址始终不变，所以他所保存的是新赋予的数值。而在改进的代码中，我们将v赋予一个新的变量tmpVal，这样通过拷贝数据，每次都能把对应的数值地址记录。

上面的问题还有一种解决方法，直接引用数据的内存，这个方法比较好，不需要开辟新的内存空间，看代码：
```go
for k,_ := range u{
  n = append(n, &u[k])
}
```

## 1.2 for-range修改变量

```go
package main

import "fmt"

type Test struct {
	Name string
	Age  int
}

func main() {
	testList := []Test{
		{"test1", 10},
		{"test2", 11},
		{"test3", 12},
		{"test4", 13},
	}

	for _, v := range testList {
        v.Age = 10
	}
    for _, v := range testList {
        fmt.Println(v)
    }
}
```

输出结果如下：
```go
➜  ~ go run main.go
&{test1 10}
&{test2 11}
&{test3 12}
&{test4 13}
```
你会发现，数值并没有改变，正确的代码如下：
```go
for k, _ := range testList{
    testList[k].age = 10
}
```

## 1.3 for-range是否会有死循环

如下代码，在for-range遍历的过程中添加元素，是否会造成死循环呢？
```go
func main() {
 v := []int{1, 2, 3}
 for i := range v {
  v = append(v, i)
 }
}
```
答案是不会，range会对切片做拷贝，新增的数据并不在拷贝内容中，并不会发生死循环。

## 1.4 for-range map删除元素
在range迭代时，可以删除map中的数据，第一次
```go
for key := range m {
    if key.expired() {
        delete(m, key)
    }
}
```

官方的解释如下：

> The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next. If map entries that have not yet been reached are removed during iteration, the corresponding iteration values will not be produced. If map entries are created during iteration, that entry may be produced during the iteration or may be skipped. The choice may vary for each entry created and from one iteration to the next. If the map is nil, the number of iterations is 0.

> `map`的迭代顺序是不确定的，并且不能保证每次迭代之间都相同。 如果在迭代过程中删除了尚未到达的映射条目，则不会生成相应的迭代值。 如果映射条目是在迭代过程中创建的，则该条目可能在迭代过程中产生或可以被跳过。 对于创建的每个条目以及从一个迭代到下一个迭代，选择可能有所不同。 如果映射为nil，则迭代次数为0

如果在迭代过程中添加了，以为map存储元素是无序的，所以，有可能存储在已经遍历的位置，此时接下来的迭代，不会打印出新数据。如果存储在还没有遍历的位置，那么接下来的迭代，会打印出新数据。

# 2. switch的使用

用法如下，switch是按顺序进行匹配，如果都匹配不上的时候，执行default。
```go

switch a {
	case 10:
		fmt.Println("Input is 10")
	default:
		fmt.Println("input is not matched")
}
```

# 3. select的使用

select是用在channel中，select如果匹配上多个，随机选择执行一个，举例分析如下：
```go
select {
	case <- channel1:
		fmt.Println("Receive data from channel1")
	case <- channel2:
		fmt.Println("Receive data from channel2")
}
```

上述代码没有default，所以当没有数据发送给channel的时候，就会block在那，如果不想block，需要加default
```go
select {
	case <- channel1:
		fmt.Println("Receive data from channel1")
	case <- channel2:
		fmt.Println("Receive data from channel2")
	default:
		fmt.Println("Don't receive any data")
}
```

`参考`
- https://mp.weixin.qq.com/s/G7z80u83LTgLyfHgzgrd9g
