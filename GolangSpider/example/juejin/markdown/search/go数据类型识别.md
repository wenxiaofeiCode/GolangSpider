# go数据类型识别 #

## go数据类型 ##

go语言数据类型主要分为以下的四个大类:

* 基础类型(整数,浮点数,负数,布尔值等)
* 聚合类型(数组,结构体)
* 引用类型(slice,指针,map,函数,通道)
* 接口类型

go语言是拥有类型系统的语言,相对于笔者最熟悉的javascript这种动态且无类型的语言来说有着长远的好处.通过类型系统能在编译阶段减少一定的运行时错误.例如在go语言中不同类型之间必须通过显示转换来进行赋值等操作.本文主要从go语言中的基础类型开始,逐步的讲解go语言中几种基本的引用类型.

## 基础类型 ##

### 字符串 ###

字符串是不可改变的字节序列.可以通过[i:j]操作符截取对应字符串的子串.由于字符串不可改变的特点,子串和母串共用一端底层内存.

` s := "hello world" b := s[6:] // [i:j] 从i开始不包括j world 复制代码`

### 常量 ###

常量是一种表达式,可以在编译的阶段来确定相应的值.在声明常量的时候可以指定类型和值(如果没有指定类型会通过值来推断常量的类型).在连续声明多个常量的时候,主要有以下两种方式:

` const ( a = iota b c d ) // a b c d 0 1 2 3 const ( a = 1 b c = 2 d ) // a b c d 1 1 2 2 复制代码`

无类型常量(常量字面量)是还没有确定从属类型的常量值.无类型常量相对于同样的有类型的常量有更大的精度.例如0.0相对有浮点数拥有更大的精度.在将无类型常量复制给对应的变量的时候,赋值的变量会转换为无类型常量默认的类型.

` i := 0 // int(0) b := 0.0 // float64(0.0) 复制代码`

## 数组 ##

数组是具有固定长度的存储相同数据类型的序列.由于数组在声明的时候对长度有限制,当存储元素到达数组的容量的时候就需要申请新的内存空间来进行数据的存储.所以在存储数据上一般不会使用数组.数组元素的初始值是该类型的零值.在声明数组的时候需要显示的指定长度和类型.

### 数组操作 ###

可以通过下面这两种方式声明一个数组.

` test := [...]int{1,2,3} test := [3]int{1,2,3} a := [2]int{} // 定义长度为2的数组 a[0] // 0 通过下标访问 b := [2]int{} a == b // true c := [3]int{} a == c // false 这行代码在编译的时候会报错,因为长度也属于数组类型的一部分,长度2和长度3的数组无法比较 array1 = [3]int{1,2,3} array2 = [3]int{} array2 = array1 // 同类型的数组可以复制,对于指针数组只复制指向. 复制代码`

### 注意点 ###

go语言在传递函数的参数的时候,总是以值的方式进行传递,对于长数组可以考虑通过传递指针的方式来降低复制的性能

## slice ##

slice是用相同类型元素的可变长度序列.可以基于一个已有的数组来创建这个数组的slice.slice有三个属性:指针,容量,长度.可以在一个数组的基础上产生多个slice,它们共享内存空间.需要注意的是slice可以理解为对原数组的引用,通过对slice的修改是会影响到底层数组的.

### 声明和使用 ###

#### 声明切片 ####

主要可以通过切片字面量和内置的make函数来实现切片的声明.

` var test = make(int[], 3, 5) // make(type[], len, cap) 声明一个长度为3容量为5的切片 var test1 = []int{1,2} // 声明一个长度和容量都为2的切片 var test2 = []int{ 99: 1 } // 声明一个长度和容量为100的切片,初始化第100的元素为1 //声明数组与声明切片的对比 注意声明slice的时候不需要指定长度 test3 := [2]int{1,2} test4 := []int{1,2} 复制代码`

#### 操作切片 ####

` var num = [10]int{1,2,3,4,5,6,7,8,9,10} a := num[1:4] // 操作符[i:j]创建一个新的slice,引用原数组i到j-1个元素.slice的容量是slice起始元素到底层数组最后一个元素之间的个数,切片a只能看到底层数组i以及之后的元素 len(a) // 3 获取切片的长度 cap(a) // 9 获取切片的容量 a[0] = 100 b := a[:5] // 可以在一个已有的slice上扩充容量,产生新的slice len(b) // 4 cap(b) // 9 // 通过切片创建切片 slice := []int{1,2,3,4,5} newSlice := slice[1:3] 复制代码`

在上面最后的一个例子中,可以通过切片创建切片,slice能看到底层数组的所有元素,newSlice只能看到底层数组1(包括)之后的元素.

#### 切片增长 ####

可以通过内置的append函数来动态的增加元素到切片中.

` slice := []int{1,2,3,4,5}// 声明一个切片 newSlice := slice[1:2] // 在已有切片上创建一个切片 切片的长度是1 容量是4 newSlice = append(newSlice, 10) fmt.Println(slice[2]) // 10 复制代码`

在刚才的例子中,由于newSlice的容量是4,在进行append的时候没有达到容量的限制,所以底层数组的元素被修改了.对于这种情况在创建切片刀时候就限制切片的容量和长度一致.这样当进行元素的添加的时候,append函数会新创建底层数据.

` slice := []int{1,2,3,4,5}// 声明一个切片 newSlice := slice[1:2:2] // 在已有切片上创建一个切片 操作符[i:j:k] 长度 j-i 容量 k-i newSlice = append(newSlice, 10) fmt.Println(slice[2]) // 3 复制代码`

append还支持批量的增加元素.

` a := []int{1,2} b := []int{3,4} c := append(a, b...) // 迭代切片 for _, value := range c { fmt.Println(value) } 复制代码`

## map ##

在go中map是对散列表的引用.散列表是无序的键值的结合.可以通过如下的方式创建map:

` var test = make(map[string]int) // 声明map的键值的类型 map的值也可以是复合类型 例如 make(map[string]map[string]int) test["name"] = 100 // 赋值 var person = map[string]int{ "card": 1, id: "2" } //声明并初始map person["card"] // 1 delete(person, "card") // 删除对应的属性 if value, ok := test["haha"]; !ok { // 不存在对应的键值 } 复制代码`

可以声明一个nil映射,nil映射不能用于存储建值.

` var a map[string]int a["hahh"] = 1 // 报错 复制代码`

map有以下几点需要特别注意:

* 在赋值map类型的值的时候,需要对map进行初始化.初始化的map的值是对应类型的零值.
* 获取键值的操作返回值,来判断对应元素是否存在map中.
* map的键值是可以使用==来比较的(slice不可以).
* map是引用类型,在函数间传递的时候会对原值进行修改.

## 结构体 ##

结构体是将多个命名变量组合到一起的聚合数据类型.可以通过下面的方式声明一个结构体:

` type Person struct { name string id,age int baby []int } var haha Person // 下面这几种对结构体变量赋值的方式是等同的 haha.name = "haha" test := Person{ name: "100" } // 指定变量的值的方式初始化变量 name := &haha.name *name = "100" var copyPerson *Person = &haha copyPerson.name = "100" 复制代码`

在声明结构体的时候需要注意以下的几点:

* 结构体变量的大小写决定变量是否可以导出(可以被其他导出的包读写)
* 结构体变量不可以拥有自己本身的结构体类型,可以通过自身结构体类型的指针来实现递归结构
* 如果结构体所有的成员变量是可比较的,那么这个结构体是可比较的

### 结构体嵌套 ###

在定义结构体的成员的时候,go允许只指定成员的类型来实现成员的声明.通过这种方式定义的结构体成员成为匿名成员.匿名成员的类型必须是一种命名类型或者指向命名类型的指针.匿名成员可以为方便变量提供便捷的操作.

` type Circle struct { x int y int radius int } type Wheel struct { Circle color string } wheel := Wheel{Circle{ 10, 10, 19}, "red"} // wheel.x = 10 复制代码`

## 后记 ##

我自己在写一个公众号-前端小板凳,主要分享我学习前端和工作中的一些想法,欢迎大家关注.