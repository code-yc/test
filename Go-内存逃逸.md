### 一、什么是内存逃逸？

#### 1、内存管理

在Go语言中，内存分配有两种方式：`栈分配`和`堆分配`

- `栈分配`在函数调用时为局部变量分配内存，当函数返回时，这些内存会自动释放。
- `堆分配`通过 `new` 或者 `make` 函数动态分配内存，需要`GC`释放。

#### 2、原理

> **定义：本应在栈上分配的内存被分配到堆上**

- 在函数内部创建的变量/对象，在函数结束后仍然被其他部分引用或持有
- 这意味着即使函数返回后，这部分内存也不会被自动释放，需要等待垃圾回收器`GC`来回收。

#### 3、逃逸的影响

- 内存占用增加
- 性能下降
- 内存泄漏



### 二、内存逃逸：指针逃逸

> **当函数的局部变量/对象，被函数外部所引用时，会逃逸到堆上**

#### 1、案例：返回指针

- 当带有指针的返回值，赋值给外部变量或作为参数传递给其他函数时
- 编译器无法确定该变量何时停止使用，因此必须将该数据分配到堆上，使其逃离当前函数作用域
- `slice`和`map`在底层实现上与指针相同

```
// 指针
func f1() (*int){	
	i := 0
	return &i
}

// slice
func f2() ([]int){	
	s := []int{1,2,3}
	return s
}

// map
func f3() (map[int]int){	
	m := map[int]int{1:1, 2:2}
	return m
}
```

#### 2、案例：向`channel`发送指针

- 向`channel`中发送数据的指针或包含指针的值 ，当其他协程或方法引用`channel`里的值时
- 编译器不知道该值什么时候会被接收，因此只能放入堆中

```
func f1(){
	i := 2
	ch := make(chan *int, 2)
	ch <- &i
	<- ch
}
```



#### 3、案例：在slice或map中存储指针

- 在`slice`或`map`中存储指针或包含指针的值
- 编译器无法确定该指针引用的数据是否在函数返回后还会使用，因此将会将指针指向的值分配到堆上，以便函数返回后继续存在

```
func f1() {
	i := 1
	list := make([]*int,2)
	list[0] = &i
}
```



### 三、内存逃逸：闭包

> **如果一个函数返回一个闭包，并且该闭包引用了函数的局部变量，那么这些变量会逃逸到堆上。**

- 在下例中，返回的函数用到了`i`，说明`i`超出`f1()`的生命周期

```
func f1() func() {
	i := 1
	return func() {
		fmt.Println(i)
	}
}
```



### 四、内存逃逸：接口动态分配

> 当一个具体类型的变量被赋值给接口类型时，由于接口的动态特性，具体的值可能会发生逃逸。

- 由于接口可以持有任意实现了该接口的类型，编译器在编译时无法确定具体的动态类型
- 因此，在运行时，将对象分配到堆上

```
type Person struct {
    Name string
}

func createPerson() interface{} {
    p := Person{Name: "Alice"}  // p 是一个局部变量
    return p  // p 被赋值给接口类型，发生了内存逃逸
}

func main() {
    result := createPerson()  // result 是接口类型，内部包含 Person 类型的值
    fmt.Println(result)
}
```



### 五、内存逃逸：栈空间不足

> **编译时无法确定类型或大小，会逃逸到堆上**

- 举例：定义一个`interface`但未赋值、
- 举例：对切片进行扩展时，不知道切片的大小

> **占用空间过大，会逃逸到堆上**

- 举例：函数内部定义一个过大的切片



### 六、优化策略

- 当不必要的时候，尽量使用值传递而不是指针传递。
- 严格限制变量的作用域。如果一个变量只在函数内部使用，就不要将其返回或赋值给外部变量。
- 避免在频繁调用的函数中使用闭包。
- 预分配切片和 map 的容量。



### 七、逃逸分析

使用`-gcflags`进行逃逸分析

```
go build -gcflags main.go
```

你会观察到以下意味着发生了逃逸

```
move to heap : i
[]int{...} escapes to heap
```



### 补充

#### 1、go语言内存分配的基本原则

- 指向栈上对象的指针，不能存储到堆上（如果指针存储到堆上，就会将对象也一起挪到堆上）
- 指向栈上对象的指针不能超过该栈对象的生命周期（如果栈对象被回收了，指针一定要被回收，如果不能，会挪到堆上）