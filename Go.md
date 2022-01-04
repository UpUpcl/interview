# 1.Go语言编译器

# 2.类型推断

# 3.常量与隐式转换

# 4.字符串

## 字符串的结构体

```go
type StringHeader struct{
	Data uintprt
	Len int
}
```

Data指向底层的字符数组，Len代表字符串的长度。**Go采用UTF-8编码，每个字符占据1个字节，但是汉字占据3个字节**

## rune类型

go语言中的字符有两种：
 	1. uint8类型，或者为byte类型代表了一个ASCII的一个字符
 	2. rune类型，代表一个UTF-8字符

用字符表示字符串的组成元素可能会产生歧义。在go语言中利用rune类型来表示和区分字符串重的“字符”
利用range轮询字符串是，轮询的不再是单字节，而是具体的rune类型
rune类型是int32的别称。

## 字符串底层原理

go语言中声明字符串的方式有两种：
1. var a string = `hello` （反引号）
2. var b string = "hello"
在解析字符串时，双引号调用stdstring( )，如果出现另一个双引号直接退出，如果出现\\，则转译字符；单撇号比较简单：一直循环向后读，直到匹配到配对的单撇号。

### 字符串拼接
1. 通过 + 
2. 用strings.join( )
3. fmt.Sprintf( )
原理：字符串拼接不是简单地将一个字符串合并到另一个字符串中，而是找到一个更大的空间。

# 指针

Go语言的指针与别与C/C++中的指针，Go语言总的指针不能进行偏移和运算，是安全指针

**3个概念** 指针地址、指针类型和指针取值

go语言中只支持&取地址和*取值操作。 
%v打印的是变量的真实值。%T答应变量的类型。

当传递参数修改某一个变量的时候，需要传递的指针。


# 5.数组

go语言中的数组和其他语言不一样，一旦定义好，不能进行扩容，在复制和传递时为值复制。

数组是内存中连续的一片区域

## 数组底层原理
```go
type Array struct {
	Elem *Type
	Bound int64
}
```

# 6.Slice
切片是长度可变的序列，序列中每个元素的类型都相同。

```go
type SliceHeader strcut{
	Data uintptr
	Len int
	Cap int
}
```
指针指向切片元素对应的底层数组元素的地址，长度对应切片中元素的数目，长度不能超过容量。
⚠️：当我们在没有触发扩容机制，开辟新的数组空间时，修改slice的值或者添加值，都是在原有的底层数组上修改。
⚠️：当我们传递slice时，函数栈帧会复制一个SliceHeader的结构体，Data指向原来的底层数组。因此，当我们在调用函数里append数据时，底层数组也会添加数据。此时，我们在调用函数外，可以访问到数字，但是SliceHeader的Len字段没有增加。

## 扩容原理
1. 如果新申请容量cap大于2倍的旧容量old.cap， 则newcap=cap
2. 如果旧切片的长度小于1024，则最终容量是旧容量的2倍 （newcap < 2*oldcap && len< 1024则 newcap=2*oldcap）
3. 如果旧切片的长度大于或者等于1024，则最终容量从旧容量开始循环增加原来的1/4，直到最终容量大于或等于新申请的容量
⚠️：最终会根据切片类型，分配不同大小的内存。为了内存对齐，申请的内存可能大于实际的类型大小*容量大小。

**切片的截取也共享原底层数组** 
**完全拷贝切片时，需要新建一个内存，使用copy函数复制**


# 7.Map

哈希表的原理是将多个key/value对分散存储在buckets(桶)中。

## 哈希冲突和解决方法

**哈希冲突**：不同的key，经过hash函数之后可能产生相同的hash值。
**解决冲突的方法** 拉链法和开放地址法

Go语言采用的是链地址法，在bucket后边连接一个溢出桶。

用var声明的slice，channel，map时，如果不声明字面量，则值为nil，访问会报panic，需要用make开辟内存空间；

## key的可比性
**可比类型**

1. bool, int, float, complex, string
2. 指针值时可比较的，如果两个指针值指向相同的变量，或者两个指针的值均为nil，则他们相等。
3. 通道值是可以比较的。如果两个通道值是有make创建的，或者两个值都为nil，则他们相等。
4. 接口值是可比较的。如果两个接口值具有相同的动态类型和相同的动态值，或者两个接口值都为nil，则他们相等。
5. 如果两个结构体的所有字段可比，那么他们的值可以比较
6. 如果数组的元素类型的值可比较，则数组可比较
7. **silce， map，function不可比较**

⚠️：map只支持并发读，不支持并发写。需要用sync.map

## map底层数据结构
```go
type hmap struct{
	count int
	flags uint8
	B uint8
	noverflow uint16
	hash0. uint32
	buckets unsafe.Pointer
	oldbuckts unsafe.Pointer
	nevacuate uintptr
	extra *mapextra
}
```
count 代表map中的元素
flags代表当前map的状态（是否处在写入的状态）
2的B次方表示当前map中桶的数量
noverflow表示溢出桶的数量，当溢出桶太多是，会进行等量扩容。
hash0代表生成hash的随机数种子
buckets是指向当前map对应的桶的指针
oldbuckets是在map扩容时存储旧桶的，当所有旧桶中的数据都已经转移到了新桶中时，则清空
nevacuate在扩容时使用，用于标记当前旧桶中小于nevacuate的数据都已经转移到新桶中
extra存储map中的溢出桶。当B>=4时，表示可能用到溢出桶的概率很大，所以会预先分配2^(B-4)个溢出桶。

低B为表示放在哪一个桶，代表桶的bmap结构只列出了首歌字段，即一个固定长度为8的数组，存储key的hash值的前8位
一个桶可以装8个key-value键值对。
`
type bmap struct{
	tophash [bucketCnt]uint8
	overflow *bmap
}
`
图解：
![图片](https://mmbiz.qpic.cn/mmbiz_png/mUp5wBPumxpNxs5s5dxcN0tOhOORuY3picCZUyeKXF3dsWF1fFDrkUaHN7prHSR8JuN7HZfGAOkdX1rDw6B7icCQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当bucket存满8个时，并不会开辟一个新桶，而是将数据放置到溢出桶中，每个桶的最后都存储了overflow，即溢出桶的指针。

当初始化map的数量比较大，则map会提前创建好一些溢出桶存储在extra字段,当出现溢出现象时，可以用提前创建好的桶而不用申请额外的内存空间。只有预分配的溢出桶使用完了，才会新建溢出桶。
```go
type mapextra struct{
	overflow *[]*bmap
	oldoverflow *[]*bmap
	nextOverflow *bmap
}
```

nextOverflow指向下一个空闲的溢出桶

![图片](https://mmbiz.qpic.cn/mmbiz_png/mUp5wBPumxpNxs5s5dxcN0tOhOORuY3pxkEy7qRlJEzlxyl8ll17icn5qc9hwp6cVVFtIndr4BsxbZibDH3LWCbg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## map重建
 发生重建的原因：
 1. map超过了负载因子大小
 2. 溢出桶的数量太多（B<15时，溢出桶的数量大于等于2^B；>=15时，溢出桶的数量等于2^B）

负载因子 = count / (2^B)    负载因子位6.5
当超过负载因子6.5， map会进行翻倍扩容。目的是减少冲突的发生。
当溢出桶使用太多时，会进行等量扩容。目的是防止溢出桶数量的增加，导致内存泄漏

⚠️：map的重建，会将桶的迁移均摊到多次io操作中，避免一次性迁移带来的瞬时抖动。

![图片](https://mmbiz.qpic.cn/mmbiz_png/mUp5wBPumxpNxs5s5dxcN0tOhOORuY3paFyMDL1bLGP51XLJB9LYPFNpibuYJl2Yh76f6Vl9hWqbdicqnwkick1iaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当进行map的delete操作时, 和赋值操作类似，会找到指定的桶，如果存在指定的key,那么就释放掉key与value引用的内存。同时tophash中指定位置会存储`emptyOne`,代表当前位置是空的。

同时在删除操作时，会探测到是否当前要删除的元素之后都是空的。如果是，tophash会存储为`emptyRest`. 这样做的好处是在做查找操作时，遇到emptyRest 可以直接退出，因为后面的元素都是空的。


# 8.函数与栈

# 9.defer

# 10. Panic和recover

## 发生panic的原因

1. 指针越界
2. 断言错误
3. 关闭一个已经关闭的channel
4. 向关闭的channel写数据

## panic和recover的使用

# 11. Interface

## 接口动态类型
**存储**在接口中的类型称为接口的动态类型
下列代码就是存储。定义一个接口，然后利用实现了该接口的结构体赋值。

```go
var s Shape
s = Rectangle{3, 4}
```

接口本身的类型称为静态类型

## 接口的动态调用
多个结构体实现了该接口，接口动态调用的过程实质上时调用当前接口动态类型中具体方法的过程；在对接口变量进行动态调用时，调用的方法只能是接口中具有的方法。

## 多接口
一个类型，可以实现多个接口
## 接口的组合
定义的接口可以是其他接口的组合。例如在Go语言中io的ReadWriter接口，组合了Reader和Writer接口。
## 接口类型断言
可以使用i.(Type)在运行时获取存储在接口中的类型，i表示接口，Type代表实现次接口的动态类型。其中Type一定是实现了接口i的类型，否则编译不通过；
## 接口的比较性
接口值是可比较的。如果两个接口值具有相同的动态类型和相同的动态值，或者两个接口值都为nil，则他们相等。
**规则**
1. 动态值为nil的接口变量总是相等的
2. 如果只有1个接口为nil，那么比较结果总是false
3. 如果接口存储的动态类型值是不可比较的，在运行时报错。 

## interfance的10个问题
1. **值接收者和指针接收者的区别方法**
方法能给用户自定义的类型添加新的行为。它和函数的区别在于方法有一个接收者，给一个函数添加一个接收者，那么它就变成了方法。接收者可以是值接收者，也可以是指针接收者。

在调用方法的时候，值类型既可以调用值接收者的方法，也可以调用指针接收者的方法；指针类型既可以调用指针接收者的方法，也可以调用值接收者的方法。

**也就是说，不管方法的接收者是什么类型，该类型的值和指针都可以调用，不必严格符合接收者的类型。**

![image-20211212200209179](/Users/chenlei/Library/Application Support/typora-user-images/image-20211212200209179.png)

**如果实现了接收者是值类型的方法，会隐含地也实现了接收者是指针类型的方法。**

**什么时候使用**：如果方法的接收者是值类型，无论调用者是对象还是对象指针，修改的都是对象的副本，不影响调用者；如果方法的接收者是指针类型，则调用者修改的是指针指向的对象本身。
**使用指针作为方法的接收者的理由**：

•方法能够修改接收者指向的值。

•避免在每次调用方法时复制该值，在值的类型为大型结构体时，这样做会更加高效。

2. **iface 和 eface 的区别是什么**
iface 和 eface 都是 Go 中描述接口的底层结构体，区别在于 iface 描述的接口包含方法，而 eface 则是不包含任何方法的空接口：interface{}。

**iface**接口包含方法； **eface**空接口

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type itab struct {
    inter  *interfacetype
    _type  *_type
    link   *itab
    hash   uint32 // copy of _type.hash. Used for type switches.
    bad    bool   // type does not implement interface
    inhash bool   // has this itab been added to hash?
    unused [2]byte
    fun    [1]uintptr // variable sized
}
```
iface 内部维护两个指针，tab 指向一个 itab 实体， 它表示接口的类型以及赋给这个接口的实体类型。data 则指向接口具体的值，一般而言是一个指向堆内存的指针。

再来仔细看一下 itab 结构体：_type 字段描述了实体的类型，包括内存对齐方式，大小等；inter 字段则描述了接口的类型。fun 字段放置和接口方法对应的具体数据类型的方法地址，实现接口调用方法的动态分派，一般在每次给接口赋值发生转换时会更新此表，或者直接拿缓存的 itab。
另外，你可能会觉得奇怪，为什么 fun 数组的大小为 1，要是接口定义了多个方法可怎么办？实际上，这里存储的是第一个方法的函数指针，如果有更多的方法，在它之后的内存空间里继续存储。

```go
type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}
```
可以看到，它包装了 _type 类型，_type 实际上是描述 Go 语言中各种数据类型的结构体。我们注意到，这里还包含一个 mhdr 字段，表示接口所定义的函数列表， pkgpath 记录定义了接口的包名。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx61tJTdWgO8Ngw5GcTPY5UOhJ9D0EyIib6mBLCUsy4vMBdXN0bAn6ibAbTUU5eLo7dyjhDh2EExARAJg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**eface**结构体

```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```

相比 `iface`，`eface` 就比较简单了。只维护了一个 `_type` 字段，表示空接口所承载的具体的实体类型。`data` 描述了具体的值。

```go
type _type struct {
    // 类型大小
    size       uintptr
    ptrdata    uintptr
    // 类型的 hash 值
    hash       uint32
    // 类型的 flag，和反射相关
    tflag      tflag
    // 内存对齐相关
    align      uint8
    fieldalign uint8
    // 类型的编号，有bool, slice, struct 等等等等
    kind       uint8
    alg        *typeAlg
    // gc 相关
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
```

3. **接口的动态类型和动态值**

从源码里可以看到：`iface`包含两个字段：`tab` 是接口表指针，指向类型信息；`data` 是数据指针，则指向具体的数据。它们分别被称为`动态类型`和`动态值`。而接口值包括`动态类型`和`动态值`。

4. **接口的构造过程是怎样的**

我们已经看过了 `iface` 和 `eface` 的源码，知道 `iface` 最重要的是 `itab`和 `_type`。

调用convT2I64函数，返回一个iface类型接口

```go
func convT2I64(tab *itab, elem unsafe.Pointer) (i iface) {
    // ……
}
```

5. **类型转换和断言的区别**

类型转换：<结果类型> := <目标类型> (<表达式>)
因为空接口 interface{} 没有定义任何函数，因此 Go 中所有类型都实现了空接口。当一个函数的形参是 interface{}，那么在函数中，需要对形参进行断言，从而得到它的真实类型
断言： // 安全类型断言
<目标类型的值>，<布尔参数> := <表达式>.( 目标类型 )  

//非安全类型断言
<目标类型的值> := <表达式>.( 目标类型 )
断言错误会直接报panic错误

线上断言一般采用安全类型，或者switch-case结构

6. **接口转换原理**
通过前面提到的 iface 的源码可以看到，实际上它包含接口的类型 interfacetype 和 实体类型的类型 _type，这两者都是 iface 的字段 itab 的成员。也就是说生成一个 itab 同时需要接口的类型和实体的类型。

调用了runtime.convI2I(SB)方法

```go
func convI2I(inter *interfacetype, i iface) (r iface) {
    tab := i.tab
    if tab == nil {
        return
    }
    if tab.inter == inter {
        r.tab = tab
        r.data = i.data
        return
    }
    r.tab = getitab(inter, tab._type, false)
    r.data = i.data
    return
}
```

代码比较简单，函数参数 `inter` 表示接口类型，`i` 表示绑定了实体类型的接口，`r` 则表示接口转换了之后的新的 `iface`。通过前面的分析，我们又知道， `iface` 是由 `tab` 和 `data` 两个字段组成。所以，实际上 `convI2I` 函数真正要做的事，找到新 `interface` 的 `tab` 和 `data`，就大功告成了。

我们还知道，`tab` 是由接口类型 `interfacetype` 和 实体类型 `_type`。所以最关键的语句是 `r.tab = getitab(inter, tab._type, false)`。

因此，重点来看下 `getitab` 函数的源码，只看关键的地方

```go
func getitab(inter *interfacetype, typ *_type, canfail bool) *itab {
    // ……

    // 根据 inter, typ 计算出 hash 值
    h := itabhash(inter, typ)

    // look twice - once without lock, once with.
    // common case will be no lock contention.
    var m *itab
    var locked int
    for locked = 0; locked < 2; locked++ {
        if locked != 0 {
            lock(&ifaceLock)
        }

        // 遍历哈希表的一个 slot
        for m = (*itab)(atomic.Loadp(unsafe.Pointer(&hash[h]))); m != nil; m = m.link {

            // 如果在 hash 表中已经找到了 itab（inter 和 typ 指针都相同）
            if m.inter == inter && m._type == typ {
                // ……

                if locked != 0 {
                    unlock(&ifaceLock)
                }
                return m
            }
        }
    }

    // 在 hash 表中没有找到 itab，那么新生成一个 itab
    m = (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*sys.PtrSize, 0, &memstats.other_sys))
    m.inter = inter
    m._type = typ

    // 添加到全局的 hash 表中
    additab(m, true, canfail)
    unlock(&ifaceLock)
    if m.bad {
        return nil
    }
    return m
}
```

简单总结一下：getitab 函数会根据 `interfacetype` 和 `_type` 去全局的 itab 哈希表中查找，如果能找到，则直接返回；否则，会根据给定的 `interfacetype` 和 `_type` 新生成一个 `itab`，并插入到 itab 哈希表，这样下一次就可以直接拿到 `itab`。

这里查找了两次，并且第二次上锁了，这是因为如果第一次没找到，在第二次仍然没有找到相应的 `itab` 的情况下，需要新生成一个，并且写入哈希表，因此需要加锁。这样，其他协程在查找相同的 `itab` 并且也没有找到时，第二次查找时，会被挂住，之后，就会查到第一个协程写入哈希表的 `itab`。

`additab` 会检查 `itab` 持有的 `interfacetype` 和 `_type` 是否符合，就是看 `_type` 是否完全实现了 `interfacetype` 的方法，也就是看两者的方法列表重叠的部分就是 `interfacetype` 所持有的方法列表。注意到其中有一个双层循环，乍一看，循环次数是 `ni * nt`，但由于两者的函数列表都按照函数名称进行了排序，因此最终只执行了 `ni + nt` 次，代码里通过一个小技巧来实现：第二层循环并没有从 0 开始计数，而是从上一次遍历到的位置开始。

7. 如何用 interface 实现多态

多态是一种运行期的行为，它有以下几个特点：

> 1.一种类型具有多种类型的能力
>
> 2.允许不同的对象对同一消息做出灵活的反应
>
> 3.以一种通用的方式对待个使用的对象
>
> 4.非动态语言必须通过继承和接口的方式来实现



# 12. Reflect编程
反射是指在程序运行期间对程序进行访问和修改的能力。程序在编译时，变量被转换为内存地址，变量名不会被编译器写入到可执行部分。在运行程序时，程序无法获取自身的信息。

## 为什么需要
在运行时，我们没有办法获取结构体变量内部的方法名以及属性名。对于函数或方法，没有办法动态地检查参数的个数和返回值的个数，更不能在运行时通过函数名动态调用函数。

例如要在运行时获取结构体的字段。

## 反射的基本使用方法
1. func ValueOf (i interfance{}) Value
2. func TypeOf (i interface{} Type)



reflect.Value看作反射的值，reflect.Type看作反射的实际类型。

reflect.Type是一个interfec，

```go
type Type interface {
	// Methods applicable to all types.

	// Align returns the alignment in bytes of a value of
	// this type when allocated in memory.
	Align() int

	// FieldAlign returns the alignment in bytes of a value of
	// this type when used as a field in a struct.
	FieldAlign() int

	// Method returns the i'th method in the type's method set.
	// It panics if i is not in the range [0, NumMethod()).
	//
	// For a non-interface type T or *T, the returned Method's Type and Func
	// fields describe a function whose first argument is the receiver,
	// and only exported methods are accessible.
	//
	// For an interface type, the returned Method's Type field gives the
	// method signature, without a receiver, and the Func field is nil.
	//
	// Methods are sorted in lexicographic order.
	Method(int) Method

	// MethodByName returns the method with that name in the type's
	// method set and a boolean indicating if the method was found.
	//
	// For a non-interface type T or *T, the returned Method's Type and Func
	// fields describe a function whose first argument is the receiver.
	//
	// For an interface type, the returned Method's Type field gives the
	// method signature, without a receiver, and the Func field is nil.
	MethodByName(string) (Method, bool)

	// NumMethod returns the number of methods accessible using Method.
	//
	// Note that NumMethod counts unexported methods only for interface types.
	NumMethod() int

	// Name returns the type's name within its package for a defined type.
	// For other (non-defined) types it returns the empty string.
	Name() string

	// PkgPath returns a defined type's package path, that is, the import path
	// that uniquely identifies the package, such as "encoding/base64".
	// If the type was predeclared (string, error) or not defined (*T, struct{},
	// []int, or A where A is an alias for a non-defined type), the package path
	// will be the empty string.
	PkgPath() string

	// Size returns the number of bytes needed to store
	// a value of the given type; it is analogous to unsafe.Sizeof.
	Size() uintptr

	// String returns a string representation of the type.
	// The string representation may use shortened package names
	// (e.g., base64 instead of "encoding/base64") and is not
	// guaranteed to be unique among types. To test for type identity,
	// compare the Types directly.
	String() string

	// Kind returns the specific kind of this type.
	Kind() Kind

	// Implements reports whether the type implements the interface type u.
	Implements(u Type) bool

	// AssignableTo reports whether a value of the type is assignable to type u.
	AssignableTo(u Type) bool

	// ConvertibleTo reports whether a value of the type is convertible to type u.
	ConvertibleTo(u Type) bool

	// Comparable reports whether values of this type are comparable.
	Comparable() bool

	// Methods applicable only to some types, depending on Kind.
	// The methods allowed for each kind are:
	//
	//	Int*, Uint*, Float*, Complex*: Bits
	//	Array: Elem, Len
	//	Chan: ChanDir, Elem
	//	Func: In, NumIn, Out, NumOut, IsVariadic.
	//	Map: Key, Elem
	//	Ptr: Elem
	//	Slice: Elem
	//	Struct: Field, FieldByIndex, FieldByName, FieldByNameFunc, NumField

	// Bits returns the size of the type in bits.
	// It panics if the type's Kind is not one of the
	// sized or unsized Int, Uint, Float, or Complex kinds.
	Bits() int

	// ChanDir returns a channel type's direction.
	// It panics if the type's Kind is not Chan.
	ChanDir() ChanDir

	// IsVariadic reports whether a function type's final input parameter
	// is a "..." parameter. If so, t.In(t.NumIn() - 1) returns the parameter's
	// implicit actual type []T.
	//
	// For concreteness, if t represents func(x int, y ... float64), then
	//
	//	t.NumIn() == 2
	//	t.In(0) is the reflect.Type for "int"
	//	t.In(1) is the reflect.Type for "[]float64"
	//	t.IsVariadic() == true
	//
	// IsVariadic panics if the type's Kind is not Func.
	IsVariadic() bool

	// Elem returns a type's element type.
	// It panics if the type's Kind is not Array, Chan, Map, Ptr, or Slice.
	Elem() Type

	// Field returns a struct type's i'th field.
	// It panics if the type's Kind is not Struct.
	// It panics if i is not in the range [0, NumField()).
	Field(i int) StructField

	// FieldByIndex returns the nested field corresponding
	// to the index sequence. It is equivalent to calling Field
	// successively for each index i.
	// It panics if the type's Kind is not Struct.
	FieldByIndex(index []int) StructField

	// FieldByName returns the struct field with the given name
	// and a boolean indicating if the field was found.
	FieldByName(name string) (StructField, bool)

	// FieldByNameFunc returns the struct field with a name
	// that satisfies the match function and a boolean indicating if
	// the field was found.
	//
	// FieldByNameFunc considers the fields in the struct itself
	// and then the fields in any embedded structs, in breadth first order,
	// stopping at the shallowest nesting depth containing one or more
	// fields satisfying the match function. If multiple fields at that depth
	// satisfy the match function, they cancel each other
	// and FieldByNameFunc returns no match.
	// This behavior mirrors Go's handling of name lookup in
	// structs containing embedded fields.
	FieldByNameFunc(match func(string) bool) (StructField, bool)

	// In returns the type of a function type's i'th input parameter.
	// It panics if the type's Kind is not Func.
	// It panics if i is not in the range [0, NumIn()).
	In(i int) Type

	// Key returns a map type's key type.
	// It panics if the type's Kind is not Map.
	Key() Type

	// Len returns an array type's length.
	// It panics if the type's Kind is not Array.
	Len() int

	// NumField returns a struct type's field count.
	// It panics if the type's Kind is not Struct.
	NumField() int

	// NumIn returns a function type's input parameter count.
	// It panics if the type's Kind is not Func.
	NumIn() int

	// NumOut returns a function type's output parameter count.
	// It panics if the type's Kind is not Func.
	NumOut() int

	// Out returns the type of a function type's i'th output parameter.
	// It panics if the type's Kind is not Func.
	// It panics if i is not in the range [0, NumOut()).
	Out(i int) Type

	common() *rtype
	uncommon() *uncommonType
}
```

reflect.Value是一个结构体, 有一个rtype的字段，相当于value也实现了type的接口

```go
type Value struct {
	// typ holds the type of the value represented by a Value.
	typ *rtype

	// Pointer-valued data or, if flagIndir is set, pointer to data.
	// Valid when either flagIndir is set or typ.pointers() is true.
	ptr unsafe.Pointer

	// flag holds metadata about the value.
	// The lowest bits are flag bits:
	//	- flagStickyRO: obtained via unexported not embedded field, so read-only
	//	- flagEmbedRO: obtained via unexported embedded field, so read-only
	//	- flagIndir: val holds a pointer to the data
	//	- flagAddr: v.CanAddr is true (implies flagIndir)
	//	- flagMethod: v is a method value.
	// The next five bits give the Kind of the value.
	// This repeats typ.Kind() except for method values.
	// The remaining 23+ bits give a method number for method values.
	// If flag.kind() != Func, code can assume that flagMethod is unset.
	// If ifaceIndir(typ), code can assume that flagIndir is set.
	flag

	// A method value represents a curried method invocation
	// like r.Read for some receiver r. The typ+val+flag bits describe
	// the receiver r, but the flag's Kind bits say Func (methods are
	// functions), and the top bits of the flag give the method number
	// in r's type's method table.
}
```

## Elem( )间接访问 （value结构体的方法）

如果反射中存储的是**指针或接口**，要访问指针指向的数据时，利用Elem（）方法返回指针或者接口的数据。

**当要修改反射的值时，Elem方法非常重要**。

**在reflect.Type也有Elem方法，但是该方法只用于获取类型**。该方法可以返回指针，接口指向的类型，数组，通道，切片，指针，哈希表。

## 修改反射的值

可以利用reflect.Value的Set方法

```go
func (v Value) Set (x Value)
```

Set方法的传入参数必须是一个reflect.Value类型，但是反射中的类型必须是指针。

例如：

```go
var num float64 = 1.23
point := reflect.Value(num)
point.Set(reflect.Value(789))
```

**程序会panic**

 只有反射中存放的是指针才可以修改反射的值

```go
var num float64 = 1.23
point := reflect.Value(&num)
newPoint := point.Elem()
fmt.Println(newPoint.CanSet()) // CanSet()方法检测是否可以重新赋值
// 重新赋值
newPoint.SetFloat(77)
```

## 结构体与反射

### 遍历结构体字段

```go
type User struct{
  Id int
  Name string
  Age int
}

func (u User) ReflectCallFunc(){
  fmt.Println("jonson ReflectCallFunc")
}
```

```go
user := User{1, "jonson", 25}
getType := reflect.TypeOf(user)
getValue := reflect.ValueOf(user)

for i:=0; i<getType.NumFiled();i++{
  field := getType.Field(i)
  value := getValue.Field(i).Interface() // value的interface以空接口的形式返回reflect.value中的值
}
```

通过reflect.Type类型的NumFiled函数获取结构体中字段的个数。reflect.Type与reflect.Value都有Filed方法，reflect.Type的Filed获取结构体的元信息，其返回StructField结构，该结构包含字段名、所在包名、Tag等基础信息

### 修改结构体字段

⚠️：反射修改值，必须传入**指针、指针、指针。**

⚠️：当修改结构体的值时，**传入指针还是不行，必须用Elem获取指针指向的结构体类型。**

⚠️：当结构体的字段名不是大写开头，不能被复制。

### 嵌套结构体的赋值

```go
type children struct{
  Age int
}
type Nested struct{
  X int
  Child children
}

vs := reflect.ValueOf(&Nested{}).Elem()
vz := vs.Filed(1)
vz.Set(reflect.Value(children{Age:18}))
```



### 结构体方法与动态调用

### 反射在运行时创建结构体


**TypeOf函数: 得到传入值的类型信息**
Name 和kind 函数；Name得到的是类型的具体类型名字； Kind得到的是类型的种类

```go
func reflectType(x interface{})  {
	// 别人不知道调用我这个函数的时候会传进来什么
	// 1. 类型断言 只能一个一个的去猜
	// 2. 反射
	v := reflect.TypeOf(x)
	fmt.Println(v, v.Name(), v.Kind())
	fmt.Printf("%T\n", v)
}

func main() {
	var a  float32 = 1.23
	reflectType(a)
	var b int8 = 12
	reflectType(b)
	var c Cat
	reflectType(c)
	var d Dog
	reflectType(d)
}
// float32 float32 float32
// *reflect.rtype
// int8 int8 int8
// *reflect.rtype
// main.Cat Cat struct
// *reflect.rtype
// main.Dog Dog struct
// *reflect.rtype
```

ValueOf函数: 得到传入值的反射值

```go
```



# 13. Goroutine

协程的栈大小在2kb， 线程的栈为2MB

## 上下文切换

在进程、线程、和协程切换时，会保存当前的运行状态，以便下次运行能够像没有发生调度一样继续运行。

进程：进程间的切换，最大问题在与内存地址空间的切换导致的缓存失效（例如虚拟地址空间和物理地址的映射）
线程：保存寄存器指针，进程状态信息等
协程：协程的切换速度要快于线程的，因为他是用户态线程，内核感受不到他的存在。并且gouroutine在切换时只需要保存很少的register值（sp，bp，pc）。而线程切换会保存额外的寄存器值，例如浮点寄存器。

## 调度策略

go语言的协程在一般情况下为协作式调度，当一个协程完成自己的任务，可以主动的释放cpu的使用权。当一个协程运行了过长时间，go的调度器会发生抢占式调度。



## 调度器
Go 语言的调度器通过使用与 CPU 数量相等的线程减少线程频繁切换的内存开销，同时在每一个线程上执行额外开销更低的 Goroutine 来降低操作系统和硬件的负载。

## go语言调度器的发展历程
1. 单线程调度器 · 0.x
    只包含 40 多行代码；
    程序中只能存在一个活跃线程，由 G-M 模型组成；

2. 多线程调度器 · 1.0
    允许运行多线程的程序；
    全局锁导致竞争严重；

3. 任务窃取调度器 · 1.1
    引入了处理器 P，构成了目前的 G-M-P 模型；
    在处理器 P 的基础上实现了基于工作窃取的调度器；
    **在某些情况下，Goroutine 不会让出线程，进而造成饥饿问题；**
    **时间过长的垃圾回收（Stop-the-world，STW）会导致程序长时间无法工作；**

4. 抢占式调度器 · 1.2 ~ 至今
	a. 基于协作的抢占式调度器 - 1.2 ~ 1.13
	通过编译器在函数调用时插入抢占检查指令，在函数调用时检查当前 Goroutine 是否发起了抢占	请求，实现基于协作的抢占式调度；
	Goroutine 可能会因为垃圾回收和循环长时间占用资源导致程序暂停；
	b. 基于信号的抢占式调度器 - 1.14 ~ 至今
	实现基于信号的真抢占式调度；
	垃圾回收在扫描栈时会触发抢占调度；
	抢占的时间点不够多，还不能覆盖全部的边缘情况；
	
	
	
	
	
	![golang-scheduler](https://img.draveness.me/2020-02-05-15808864354595-golang-scheduler.png)
	
	Go 语言调度器

G — 表示 Goroutine，它是一个待执行的任务；
M — 表示操作系统的线程，它由操作系统的调度器调度和管理；
P — 表示处理器，它可以被看做运行在线程上的本地调度器；

## G

Goroutine 只存在于 Go 语言的运行时，它是 Go 语言在用户态提供的线程，作为一种粒度更细的资源调度单元，如果使用得当能够在高并发的场景下更高效地利用机器的 CPU。

Goroutine 在 Go 语言运行时使用私有结构体 runtime.g 表示。这个私有结构体非常复杂，总共包含 40 多个用于表示各种状态的成员变量，这里也不会介绍所有的字段，仅会挑选其中的一部分，首先是与栈相关的两个字段：

```go
type g struct {
	stack       stack
	stackguard0 uintptr
}
```

其中 `stack` 字段描述了当前 Goroutine 的栈内存范围 [stack.lo, stack.hi)，另一个字段 `stackguard0` 可以用于调度器抢占式调度。除了 `stackguard0` 之外，Goroutine 中还包含另外三个与抢占密切相关的字段

```go
type g struct {
	preempt       bool // 抢占信号
	preemptStop   bool // 抢占时将状态修改成 `_Gpreempted`
	preemptShrink bool // 在同步安全点收缩栈
}
```

Goroutine 与我们在前面章节提到的 `defer` 和 `panic` 也有千丝万缕的联系，每一个 Goroutine 上都持有两个分别存储 `defer` 和 `panic` 对应结构体的链表：

```go
type g struct {
	_panic       *_panic // 最内侧的 panic 结构体
	_defer       *_defer // 最内侧的延迟函数结构体
}
```

其他重要字段

```go
type g struct {
	m              *m
	sched          gobuf
	atomicstatus   uint32
	goid           int64
}
```

- `m` — 当前 Goroutine 占用的线程，可能为空；
- `atomicstatus` — Goroutine 的状态；
- `sched` — 存储 Goroutine 的调度相关的数据；
- `goid` — Goroutine 的 ID，该字段对开发者不可见，Go 团队认为引入 ID 会让部分 Goroutine 变得更特殊，从而限制语言的并发能力

上述四个字段中，我们需要展开介绍 `sched` 字段的 [`runtime.gobuf`](https://draveness.me/golang/tree/runtime.gobuf) 结构体中包含哪些内容：

```go
type gobuf struct {
	sp   uintptr
	pc   uintptr
	g    guintptr
	ret  sys.Uintreg
	...
}
```

- `sp` — 栈指针；
- `pc` — 程序计数器；
- `g` — 持有 [`runtime.gobuf`](https://draveness.me/golang/tree/runtime.gobuf) 的 Goroutine；
- `ret` — 系统调用的返回值；

这些内容会在调度器保存或者恢复上下文的时候用到，其中的栈指针和程序计数器会用来存储或者恢复寄存器中的值，改变程序即将执行的代码。

结构体 [`runtime.g`](https://draveness.me/golang/tree/runtime.g) 的 `atomicstatus` 字段存储了当前 Goroutine 的状态。除了几个已经不被使用的以及与 GC 相关的状态之外，Goroutine 可能处于以下 9 种状态：

![image-20211215105815772](/Users/chenlei/Library/Application Support/typora-user-images/image-20211215105815772.png)

Gscan、Gscanrunnable， _Gscanrunning等是设计垃圾回收的

![golang-goroutine-state-transition](https://img.draveness.me/2020-02-05-15808864354615-golang-goroutine-state-transition.png)

## M

Go 语言并发模型中的 M 是操作系统线程。调度器最多可以创建 10000 个线程，但是其中大多数的线程都不会执行用户代码（可能陷入系统调用），最多只会有 `GOMAXPROCS` 个活跃线程能够正常运行。

Go 语言会使用私有结构体 [`runtime.m`](https://draveness.me/golang/tree/runtime.m) 表示操作系统线程，这个结构体也包含了几十个字段，这里先来了解几个与 Goroutine 相关的字段

```go
type m struct {
	g0   *g
	curg *g
	...
}
```

其中 g0 是持有调度栈的 Goroutine，`curg` 是在当前线程上运行的用户 Goroutine，这也是操作系统线程唯一关心的两个 Goroutine。

![g0-and-g](https://img.draveness.me/2020-02-05-15808864354644-g0-and-g.png)

g0 是一个运行时中比较特殊的 Goroutine，它会深度参与运行时的调度过程，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行。

[`runtime.m`](https://draveness.me/golang/tree/runtime.m) 结构体中还存在三个与处理器相关的字段，它们分别表示正在运行代码的处理器 `p`、暂存的处理器 `nextp` 和执行系统调用之前使用线程的处理器 `oldp`：

````go
type m struct {
	p             puintptr
	nextp         puintptr
	oldp          puintptr
	tls					  [6]uintprt
}
````

[`runtime.m`](https://draveness.me/golang/tree/runtime.m) 还包含大量与线程状态、锁、调度、系统调用有关的字段



## P

调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。

因为调度器在启动时就会创建 `GOMAXPROCS` 个处理器，所以 Go 语言程序的处理器数量一定会等于 `GOMAXPROCS`，这些处理器会绑定到不同的内核线程上。

[`runtime.p`](https://draveness.me/golang/tree/runtime.p) 是处理器的运行时表示，作为调度器的内部实现，它包含的字段也非常多，其中包括与性能追踪、垃圾回收和计时器相关的字段，这些字段也非常重要，但是在这里就不展示了，我们主要关注处理器中的线程和运行队列：

```go
type p struct {
	m           muintptr

	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	runnext guintptr
	...
}
```

![image-20211215111714821](/Users/chenlei/Library/Application Support/typora-user-images/image-20211215111714821.png)


## g0与普通协程的切换

一般的协程有main goroutine与子协程。main goroutine是程序启动时，创建的main goroutine来执行runtime.main的函数（包括main.main），

在每个M上都有一个g0，g0运行在操作系统线程上，其作用主要是执行协程调度的一系列代码。当用户协程退出或者被抢占时，意味着需要重新执行协程调度，这时需要从用户协程g切换到g0栈。g的结构体中sched字段保存了g在上下文切换时要保存的现场。

## 调度循环

调度循环的流程为：
执行schedule函数---->执行execute函数---->执行gogo函数----->切换到协程g执行----->当协程g主动让出、被抢占或者退出调用mcall---->切换到g0---->执行gosched_m函数、goexit函数等---->重新执行schedule函数。

其中schedule函数处理具体的调度策略，选择下一个要执行的协程
execute函数执行一些具体的状态转移、协程g与结构体m之间的绑定等操作
gogo函数是与操作系统有关的函数，用于完成栈的切换以及cpu寄存器的恢复
mcall函数主要是保存当前协程的执行现场，并切换到g0栈根据不同的原因执行不同的函数
例如 用户调用GoSched函数主动让出执行权，执行gosched_m函数；协程退出执行goexit函数

## 调度策略
调度策略的内容在schedule函数中，如何选择出一个可以运行的goroutine
在执行schedule函数时，首先会检测是否处于垃圾回收阶段，如果是，就要判断是否要执行后台标记协程。

调度器将运行队列分为全局队列和局部队列。局部队列是每个p只有一个长度为256的数组，是一个循环队列。runqhead指向runq的开头，runqtail指向runq的结尾。

```go
type g struct{
  runq   [256]guintprt
  runnext gunintprt  
}
```

**runqnext会指向下一个运行的goroutine。如果runnext不为空，则会直接执行当前runnext指向的协程，不去runq获取。**

**全局runq**

```go
type schedt struct{
  	runq  gQueue
}
```

一般的获取goroutine思路：先从本地runq中获取，再从全局获取。但是可能存在如果一直在本地runq中获取goroutine，那么全局runq的协程得不到调度。因此p中每次执行61次调度，需要优先从全局runq中获取一个G到当前p



### 获取本地runq

调度器首先查看runnext是否为空，如果不为空就返回对应的G，如果为空就从runq中寻找。当循环队列的runqhead和runqtail相等时，意味着runq为空。反之，存在可用的协程，从队列头部get一个协程返回。⚠️：虽然大部分情况只有当前G访问局部队列，但是可能存在其他P窃取任务造成同时访问的情况，因此，这里访问时需要加锁。

### 获取全局runq

当P每执行61次调度，或者局部runq没有可以运行的G，需要从全局runq中查找一批协程分配给本地runq。

⚠️：在从全局获取G时，为了保证公平，会根据P的数量平分全局运行队列中的G，同时，要转移的G的数量不能超过runq容量的一半。最后通过runqput函数循环将G放入P的局部runq中。

⚠️：如果P执行61次后，同时本地runq容量满了，此时需要将本地runq中等量的G放入全局队列，保证每个G都有机会执行。

### 获取准备就绪的网络协程
### steal 协程

如果本地runq和全局runq都没有，就会去其他P去偷取，其中函数是findrunnable。由于P都存在全局all [ ]*P中，当循环遍历allp找到可用的协程，但是缺少公平性。Go在这里用了一些数学方法，来保证每个p都有可能被窃取。找到可以窃取的p之后，利用函数runqgrap函数，将要窃取的P的一半放入自己的runq中。


## 调度时机


 ### 主动调度

 协程可以主动让出自己的CPU，利用runtime.Gosched函数。大多属情况下，用户不需要执行该函数，因为Go语言编译器会在调用函数之前插入检查代码，判断该协程是否需要被抢占。

 特殊情况：计算密集型应用，这种场景没有抢占的实际，在1.14之前无法被抢占。

 **主动调度的原理：**需要从当前协程切换到g0，取消G和M之间绑定的关系，**并将G放入全局runq**，并调用schedule函数开始下一轮循环

 ### 被动调度
 被动调度指协程在休眠、channel通道阻塞、网络I/O阻塞、执行垃圾回收而暂停时，被动的让渡自己的CPU使用权。被动调度可以最大化利用CPU资源。
 根据调度的原因不桶，调度器执行的操作也不同。

 被动调度和主动调度相似的是：需要改变G的状态和M的绑定关系并切换到g0
 被动调度和主动调度不相似的是：主动释放的G的状态为Grunnable，而被动释放的G状态为Gwaiting，所以被动调度需要一个唤醒机制。

 例如：当一个G从channel读取数据时，channel中没有可读取的数据，会调用gopark函数完成被动调度，gopark函数时被动调度的核心逻辑。

 gopark函数最后会调用park_m，该函数会接触G和M之间的关系，然后根据被动调度的原因不同，执行不同的waitunlockf函数，并开始新一轮调度。

 如果当前协程需要被唤醒，那么会从Gwaiting转换为Grunnable，并添加到当前P的局部运行队列中。

 ### 抢占式调度

 为了让每个协程都有机会执行，并且最大化CPU资源。
 在Go程序启动时会启动一个监控线程，系统监控在一个独立的M上运行，不用绑定逻辑处理器P，系统监控每隔10ms就会检测是否有准备就绪的网络协程，并放到全局runq。
 系统监控服务会判断当前协程是否运行时间过长，或者处于系统调用阶段。如果是，则会抢占当前G的执行。
 执行时间过长分为：执行超过10ms，或者一个协程系统调用超过了20微秒。

监控线程的抢占方式：1. 设置stackPreemp；2. 异步抢占

 #### 执行时间过长的抢占式

**协作式抢占**
当协程运行时间过程，其判断的是根据栈空间的大小。因此当一个协程运行时间过长，会将它结构体字段的stackguard0设置为stackPreemp字段。然后在函数调用期间，检测g.stackguard0是否等于stackPreemp，如果为true，则进行一次协程调度。

 **缺点：** 如果没有发生函数调用，那么这个协程就不会让出。
 因此咋go1.14之后引入开基于信号量的抢占式调度。

 #### 基于信号量的抢占式

 ![image-20211215193535347](/Users/chenlei/Library/Application Support/typora-user-images/image-20211215193535347.png)

Go语言借助用户态在处理信号时完成协程的上下文切换的操作，需要借助进程对特定的信号进行处理。

在抢占时，调度器向线程发送sigPreempt信号，触发信号处理。处理信号的函数为sighandler函数。当遇到sigPreempt抢占信号时，触发异步抢占机制。当信号为sigPreempt时，调用doSigPreempt函数。在执行完doSigPreempt函数后会执行asyncPreempt函数，保存当前register值，最后调用asyncPreempt2函数，调用schedule函数。

 ### 系统调用时间太久

当一个G发生系统调用时，当前正在工作的线程会陷入等待状态，等待内核完成系统调用并返回。

发生以下3中情况之一时，需要抢占调度：

1. 当前局部runq中有等待运行的G。这种情况下，抢占调度只是为了局部运行队列中的协程有执行的机会，因为其一般是当前P私有的。
2. 当前没有空闲的P和自旋的M。如果有空闲的P和自旋的M，说明当前比较空闲，那么释放当前的P没有太大意义。（自旋的M在获取G，如果没有自旋的M，说明M都能够获取到G执行，所以hand off当前的P并不会让P中的G得到执行。）
3. 当系统调用的时间超过了10ms，需要立即抢占

系统调用时的抢占原理主要是将P的状态转化为_Pidle，目的是找到一个M来执行这个P。主要的逻辑位于handoff函数中。handoff函数判断是否需要找到一个新的M来接管P。当发生以下条件之一，需要启动一个M来接管：

1. 本地runq中有等待运行的G
2. 需要处理GC
3. 没有自旋的M
4. 全局runq不为空
5. 需要处理网络socket读写事件

当这些条件都不满足，P被放入空闲队列。

⚠️：如果发生系统调用的协程从内核台返回，该协程可以继续执行时怎么办：

首先，当P被抢占时，P与M解绑，会在m的oldp字段保存该P。当被阻塞的协程返回时，优先绑定oldp。

寻找P的过程：

1. 尝试能否使用oldp，如果当前P处在Psyscall，则说明可以安全绑定
2. 当P不可使用时，加锁从全局空闲队列寻找空闲P
3. 如果没有空闲的P，则将该G放入全局runq



调度器要保证公平和效率。




# 14. Channel和select

## Channel

## Select
select是系统调用，经常使用select，poll，epoll等函数构建I/O多路复用

C语言的select系统调用可以同时坚挺多个文件描述符的可读或可写状态。Go语言可以让多个Goroutine同时等待多个Channel可读或者可写。

使用Select会出现两个现象
1. 能在channel上实现非阻塞的收发操作
	当select含有default时，如果case可以收发，那么执行对应的case，反之执行default代码
2. 在遇到多个channel同时响应是时，随机执行一个
	使用随机性为了避免讥饿问题的发生，防止后边的case永远不会得到执行
	

不同数量的case会造成select在编译时的不同
1. 不存在任何case
	会直接阻塞当前goroutine，无法唤醒
	
2. 只存在一个cese
	会改写成if语言
	
3. 存在两个case，其中一个时default
	非阻塞操作
	**发送** 会非阻塞的发送数据，所以不存在接收方或者缓存区空间不足时，当前goroutine都不会阻塞而直接返回。传入false
	**接收** 
	
4. 存在多个case
	1. 将所有case转换成包含channel以及类型的scase结构体；
	2. 调用函数从准备就绪的channel中选择一个可执行的scase
	3. 判断哪个case被选中
	
	其中最重要的就是怎么选中一个channel
	1. 确定case执行的处理顺序 （随机的轮询和地址顺序确定加锁顺序）
	
	2. 在循环中根据case的类型作出不同操作
		1. 查找是否已经存在准备就绪的channel	
		![image-20211214214934765](/Users/chenlei/Library/Application Support/typora-user-images/image-20211214214934765.png)
	
		2. 将当前goroutine加入channel对应的收发队列并等待其他goroutine被唤醒
		
		   第一阶段主要查找有没有可以立即执行的goroutine，如果不能立刻找到或缺的channel，就会进入第二阶段，将当前goroutine加入到sendq或者recvq队列
		
		3. 当前goroutine被唤醒之后找到满足条件的channel并进行处理 



![image-20211214215402768](/Users/chenlei/Library/Application Support/typora-user-images/image-20211214215402768.png)



# 15. Context
为什么需要context？**为了让协程优雅的退出。**
在没有context时，需要借助通道close的机制，该机制会唤醒所有坚挺该通道的协程。
使用context更重要的一点**协程之间通常存在级联关系，退出需要具有传递性**

## 如何使用context

```go
//net/http
func (r *Request) WithContext(ctx context.Context) *Request
```

context.Context底层是一个接口提供了四种方法：

```go
type Context interface{
  Deadline() (deadline time.Time, ok bool)
  Done() <-chan struct{}
  Err() error
  Value(key interface{}) interface{}
}
```

Deadline方法的第一个返回值表示还有多久到期，第二返回值表示是否到期。

Done是使用最频繁的，返回一个通道，一般的做法是监听该通道的信号，如果收到信号则表示通道已经关闭，需要执行退出。

如果通道已经关闭，则Err函数返回退出原因。

Value方法返回指定key对应的value。 value函数主要于安全凭证、分布式追踪ID、操作优先级等。

# 锁

## 原子锁
sync/atomic包提供了一系列原子操作函数
例如atmoic.AddInt64( )
atmoic.CompareAndSwap

```go
atmoic.CompareAndSwapInt64(&flag, 0, 1) //判断flag变量的值是否为0，如果是，则将flag的设置为1。这是一个原子操作
```



## 互斥锁

当某一个协程长时间霸占锁，其他协程得不到锁将会自旋毫无意义的消耗CPU。为了解决这种问题，操作系统的锁接口提供了终止与唤醒的机制，避免频繁自旋造成浪费。在操作系统内部构建起锁的等待队列。



互斥锁是一种混合锁，实现方式包含了自旋锁，同时参考了操作系统锁的实现。

```go
type Mutex struct{
  state int32 // 表示当前锁的状态
  sema uint32 
}
```

sema通过位图的形式存储了当前锁的状态。其中包含是否为锁定状态、正在正待被锁唤醒的协程数量、两个和饥饿模式有关的表示。

锁的饥饿模式为了解决某一个协程可能长时间无法获取锁的问题。在饥饿模式下，unlock会唤醒最先申请加速的协程，从而保证公平。sema是互斥锁中国实现的信号量。

### 加锁

互斥锁的**第一个阶段**是使用CAS操作快速抢占锁，如果成功就立即返回，反之就调用慢的方法 **m.lockSlow**函数。

**lockSlow方法**在正常情况下会自旋尝试抢占锁一段时间，而不会立即进入休眠状态。锁只有在**正常模式**下才能自旋获得锁。

下列四种情况之一，不会spaning：

1. 在单核CPU
2. P<=1
3. 当前协程所在P的runq有其他协程待运行
4. 自旋次数超过了设定的阈值

当长时间未获得锁，**就进入第二阶段**，（0为有锁，1无锁）使用信号量进行同步。如果加锁操作进入信号量同步阶段，则信号量值减1。如果减锁操作进入同步阶段，则信号量计数值加1。当信号量大于0时，以为着有其他goroutine执行了减锁操作，这是加锁协程可以直接退出。当信号量等于0，意味着加锁协程需要陷入休眠状态。

在互斥第三个阶段，所有锁的信息都会根据锁的地址存储在全局semtable哈希表中。

根据信号量地址简单取模，哈希结果相同的多个锁可能存储在同一个哈希桶中，哈希桶中通过一根双向链表解决哈希冲突问题。

⚠️：在访问hash表时，也需要加锁。但是此处的锁和互斥锁有所不同，其实现方式为先自旋一定次数，如果还没有获取到锁，则调用操作系统级别的锁。



⚠️：Go中的Mutex算一种混合锁，他结合了**原子操作、自旋、信号量、全局哈希表、等待队列、操作系统锁**。

锁被放置到全局的等待队列中并等待被唤醒，唤醒的顺序为从前往后，FIFO。当长时间无法获取锁，当前锁进入饥饿模式。在饥饿模式下，为了保证公平性，新申请锁的协程不回进入自旋状态，而是直接加入等待队列，让出自己的CPU。

### 解锁
1. 锁处于正常状态， 即没有进入饥饿状态和唤醒状态，也没有多个协程因为抢占锁陷入堵塞，则Unlock方法在修改mutexLocked立即退出，否则走unlockSlow
2. 判断锁是否重复释放。
3. 如果锁处于饥饿状态，则进入信号量同步阶段，到全局哈希表中寻找当前锁的等待队列，唤醒指定协程
4. 如果当前锁未处于饥饿状态且当前mutexWoken已设置，则表明有其他申请锁的协程准备从正常状态退出，这是锁释放后不用去当前锁的等待队列中唤醒其他协程，而是直接退出。如果唤醒了等待队列中的协程，则将唤醒的协程放入当前协程P的runnext字段。如果是饥饿状态，则当前G直接让渡自己的CPU，让被唤醒的协程直接运行。

## 读写锁

只有写锁加锁时，会获取互斥锁；读锁只会判断其他的指标有无写锁存在。

```go
type RWMutex struct{
  w 					Mutex   //互斥锁
  writerSem		uint32	//信号量，写锁等待读取完成
  readerSem		uint32	//信号量，读锁等待写入完成
  readerCount	int32		//当前正在执行读操作的数量
  readerWait	int32		//写操作被阻塞时等待的读取操作数量
}
```

1. 读锁操作先通过原子操作将readerCount加1，如果readerCount>=0就直接返回，所以如果只有获取读取锁的操作，操作成本只有一次原子操作atomic.AddInt32(&rw.readerCount, 1)。当readerCount<0时，说明当前有写锁，当前协程将借助信号量（readerSem）陷入等待状态。如果获取到信号量则立即退出，没有获取到信号时，逻辑与互斥锁一样。
2. 读锁解锁是，如果当前没有写锁，成本只有一次原子操作。如果有写锁等待，则调用rUnlockSlow函数判断当前是否为最后一个被释放的读锁，如果是则需要增加信号量并唤醒写锁。
3. 读写锁申请写锁时，要调用Lock方法先获取互斥锁。接着readerCount减去rwmutexMaxReaders阻止后续的读操作。获取到互斥锁但不一定能直接获取写锁，如果当前已经有其他goroutine持有互斥锁的读锁，那么当前协程会加入全局等待队列并进入休眠状态，当最后一个读锁被释放时，会唤醒该协程。
4. 写锁解锁时，调用Unlock方法。将readerCounter原子操作加上rwmutexMaxReaders，表示不回阻塞后续的读锁，依次唤醒所有等待中的读锁。当锁哟读锁唤醒完毕后会释放互斥锁。

⚠️：读写锁在写操作时的性能与互斥锁类似，但是只有读操作时效率要高很多。

# 17. 内存管理

## Go语言内存分配全局视野
Go语言将内存分成了67个级别的span，其中0代表特殊的大对象，大小不固定。在为具体的对象分配内存时，并不是直接分配span，而是分配不同级别的span中的元素。因此每个span的级别不是以每个span的大小为依据的，而是以span中的元素大小为依据的。
虽然每个span的大小不固定，但是都是8k或者更大的连续区域。

### 三级对象管理
为了管理span，加速span对象的访问和分配，Go语言采取了三级管理结构，分别为mcache、mcentral和mheap。
Go采用的是TCMalloc内存分配思想，每个逻辑处理器P都存储了一个本地span缓存，称作mcache。由于同一时间只有一个协程在逻辑处理器上，所以申请空间不需要加锁。并且mcache中每一种span只有一个。
1. mcentral被所有逻辑处理器P共享
2. mcentral对象收集所有给定规格大小的span。每个mcentral都包含两个mspan的链表：empty mspanList表示没有空闲对象或者span已经被mcache缓存。nonempty mspanList表示有空闲对象的span链表。
⚠️：mcache每一种span只有一个，但是mcentral每一种span都有两个链表。当mcache的span不够用时，想mcentral申请。

每个级别的span都有一个mcentral用于管理span链表，这些所有的mcentral都由mheap管理。mheap不只是管理mcentral，大对象也会直接通过mheap进行分配。

### 四级内存块管理
根据对象大小，Go语言将对内存分成了Area、chunk、span和page。在Unix系统下Area时64M、chunk是512k，page是8k。

## 对象分配
在分配内存的时候，将对象按照大小不同划分为微小对象、小对象、大对象。
其中微小对象的分配流程最长，最复杂。

### tiny对象分配
微小对象是小于16字节的对象。
1. 首先tiny对象放入class2的span中。将微小对象按照2、4、8的规则进行字节对齐。例如，字节为1的元素被分为2字节。
2. 查看之前分配的元素中是否有空余的空间。如果正在分配的元素可以容纳要分配的字节，返回tiny+offset。意味着当前地址往后的字节可以被分配。
3. 如果当前要分配的元素空间不够，将尝试从mcache中查找span中下一个可用的元素。因此tiny分配的第一步是尝试利用分配过的前一个元素的空间、达到节约内存的目的。

2.1. 在步骤2中要查找之前分配元素剩余的空间是否满足当前元素要的空间。可以利用对应级别的mspan中的allocCache字段，作为位图，用于标记span中的元素是否被分配。
allocCache是uint64，因此最多一次缓存64字节。allocCache采用小端模式标记span中的元素是否被分配。bit位为1时代表当前对应的span中的元素依据被分配。一个class2的span为8192字节。所以需要专门一个字段freeindex标识当前span中的元素被分配到了那里。span中小于freeindex序号的元素都已经被分配了，因此将从freeindex开始分配。
例如：从allocCache中找到一个0即可，假如X位为0，那么X+freeindex为当前span可用的元素序号。当allocCache都为1，需要移动freeindex并更新allocCache，一直到span中元素的末尾。

2.2 如果当前mcache中的span没有可以使用的元素，需要**加锁**从共享的mcentral中查找。

2.3 从mheap中查找。mheap位图查找、mheap基数树查找

2.4 向操作系统申请。调用mmap向操作系统申请，申请时必须为heapArea的倍数。也就是最少也要申请64MB。

### 小对象分配
小于32kb的对象。确定好属于哪一个等级的span，并在指定等级的span中查找。mcache-->mcentral-->mheap位图查找-->mheap基数树查找-->操作系统分配。

### 大对象分配
大于32kb的，因为class66的span为32kb。直接向mheap申请。

# 18. GC

## 常见垃圾回收算法
### 标记-清扫

### 标记-压缩

### 半空间复制

### 引用计数

### 分代GC


## Go的垃圾回收演进
Go语言采用并发三色标记算法进行垃圾回收

压缩GC缺点：压缩算法的主要优势是减少碎片并且快速分配。Go语言使用了TCMalloc虽然没有压缩算法那样极致，但是很好地解决了内存碎片的问题。并且需要加锁，压缩算法并不适合并发程序使用

分代GC：Go语言有内存逃逸，将继续使用的对象转移到了堆中，大部分生命周期很短的对象会在栈中分配，降低了分代的优势。

Go1.0采用单线程GC，在垃圾回收开始阶段需要停止所有用户协程，并且在垃圾回收阶段只有一个协程执行垃圾回收。

Go1.1 垃圾回收采用多协程并行执行，大大加快了垃圾回收的速度，但是还是不允许用户协程执行。

Go1.5进行了重大更新，允许用户协程与后台的垃圾回收同时执行，大大降低了用户协程暂停的事件 300ms--->40ms

Go1.6 大幅度减少了STW期间的任务，暂停时间从40ms-->5ms

Go1.8 采用混合写屏障技术小初了栈重新扫描长度时间，将用户协程暂停时间降低到0.5ms

垃圾收集是一门非常古老的技术，它的执行速度和利用率很大程度上决定了程序的运行速度，Go 语言为了实现高性能的并发垃圾收集器，使用**三色抽象、并发增量回收、混合写屏障、调步算法以及用户程序协助等机制将垃圾收集的暂停时间优化至毫秒级以下**，从早期的版本看到今天，我们能体会到其中的工程设计和演进，作者觉得研究垃圾收集的是实现原理还是非常值得的。



## 垃圾回收循环

Go语言垃圾回收大致会经历 
触发垃圾回收---->标记准备阶段---->并行标记阶段---->标记终止阶段---->垃圾清扫阶段

在并行标记阶段引入了辅助标记
在垃圾清扫阶段引入了辅助清扫和系统驻留内存清扫

触发垃圾回收的三种方式：

1. 手动触发 runtime.GC
2. 分配内存时 判断是否分配的内存是否达到了gc.tigger
3. 监控线程：超过一定时间

### 标记准备阶段

标记准备阶段最重要的任务就是清扫上一阶段GC一流的需要清扫的对象，因为使用了**懒清扫算法**，所以当执行下一次GC时，可能还有垃圾对象没有被清扫。在标记准备阶段会充值各种状态和统计指标，启动专门用于标记的协程、统计需要扫描的任务数量、开启写屏障、启动标记协程。需要STW。

标记准备阶段会为每个P启动一个标记协程，但是并不是所有的标记协程都有机会执行，因此用户协程宇标记协程需要并行，减少GC给用户带来的影响。

标记阶段关注的问题：1. 何如决定需要多少标记协程；2. 如何调度标记协程

#### 计算标记协程的数量
Go语言规定标记协程消耗的CPU应该接近25%

dedicatedMarkWorkersNeeded = gomaxproc * 0.25 + 0.5

utilError ：= dedicatedMarkWorkersNeeded/（gomaxproc * 0.25）
设置一个maxUtilError=0.3
如果utilError > maxUtilError,  dedicatedMarkWorkersNeeded减1
并计算 fractionUtilizationCoal = （ gomaxproc * 0.25 - dedicatedMarkWorkersNeeded）/ （gomaxproc）

fractionUtilization是为了P数量太少引入的。如果0.25P=0.5，如果开一个协程那么误差太大，所以franctionUtilizationCoal计算为0.25，代表每个P在标记阶段需要花25%时间执行后台标记。

#### 切换到后台标记协程
如何调度标记协程。
在标记准备阶段执行了STW，此时暂停了所有协程，当关闭STW再次启动时，每个逻辑处理器P都会进行新一轮的调度，在调度循环开始时，调度器会判断程序是否处于GC阶段，如果是则尝试判断当前P是否需要执行后台标记任务。

在findRunnableGCNeeded大于0，表示当前协程立即执行标记任务。如果参数franctionUtilization
goal大于0，并且当前P执行标记任务的时间小于franctionUtilizationGoal X 当前标记周期的时间，会继续执行后台标记任务，但是不会在整个标记周期一直执行。

### 并行标记阶段
后台标记有三种模式：
1. DedicatedMode：专门负责标记对象，不会被调度器抢占
2. FranctionMode：协助标记任务，在标记阶段达到时间后，会自动退出
3. IdleMode：代表当前没有处理器可以查找到可以执行的协程时，执行垃圾收集的标记任务，直到被抢占。

后台标记flag有四种：
1. gcDraibUntilPreempt：当前协程处于可以被抢占状态
2. **gcDrainFlushBgCredit**：标记会计算后台完成的标记任务量以减少并行标记期间用户协程执行辅助垃圾收集的工作量。
3. gcDrainIdle：对应IdleMode
4. gcDrainFraction：对应FranctionMode，执行到时间后退出

#### 根对象扫描

扫描的第一阶段时扫描根对象。在标记准备阶段会统计出这次GC一共要扫描多少对象，要扫描的对象要原子操作。因为并发标记对象可能有多个协程访问。根对象包括全局变量、span中finalizer的任务数量以及所有的协程栈

在.bss段.data段和协程栈上的对象称为根对象。

#### 全局变量扫描



#### 栈扫描
#### 栈对象
#### 扫描灰色对象

### 标记终止阶段

标记终止阶段主要完成一些指标，例如统计用时，统计强制开始GC的次数、更新下一次触发GC需要到达的对目标、关闭写屏障，并唤醒后台清扫的协程，开始下一阶段的清扫工作。标记终止阶段会再一次进入STW。

标记终止阶段最重要的任务是计算下一次触发GC时需要达到的堆目标，叫做垃圾回收的调步算法。

### 垃圾清扫阶段

#### 懒清扫逻辑

#### 辅助清扫

## 辅助标记

解决时GC的正常结束或者循环问题。

## 屏障技术

屏障技术解决的是标记的准确问题。

Go语言采用混合写屏障，利用插入写屏障和删除写屏障来实现强弱三色不变性。

强三色不变性：不允许出现黑色到白色对象的引用；强三色不变性强调白色对象对黑色对象的引用操作。而插入写屏障会将白色对象标记为灰色，或者黑色对象退为灰色。

弱三色不变性：如果出现黑色到白色对象的引用，如果有到该白色对象的可达灰色对象，也可以避免误判。弱三色不变性提示我们关注破坏灰色对象到白色对象路径的行为。而删除写屏障会将白色对象标记为灰色对象。

# 19. PProf
CPU
Memory
Goroutine

![image-20211217213425736](/Users/chenlei/Library/Application Support/typora-user-images/image-20211217213425736.png)



# 20 test

利用Test测试，和Benchmark进行性能基准分析，可以看到执行速度，利用的内存，内存分配次数等，-bench至少运行一秒钟。

