---
layout: post
title: go语言interface类型
---

内容
-------

- [go的interface类型](#interface变量)
- [interface实现](#方法)
- [type assertion](#类型与实例)
- [总结](#总结)


go的interface类型
---------
interface是go语言里面的一种数据类型，不过它与一般的数据类型有点不一样。我们理解的传统意义上的数据类型表示的是数据的存储解析形式，但是interface并没有涉及数据，它不能揭示任何的数据细节，也就是说它不是从数据的层面来定义一个变量。它是从方法层面来定义一个变量。

和传统变量通过判断变量的数据格式是否一致来判断变量的类型是否相同一样，interface通过判断两个变量实现的方法是否一致(或具有包含关系)来判断这两个变量是否是属于相同的interface。

比如我们定义一个interface变量类型：

```go
type myinterface interface {
	hello()
	world() string
}
```
我们可以看到该类型定义了两个方法。

当然，这些方法只是原型，没有实现。myinterface声明的作用主要是为所有实现了的数据类型提供一种类似泛型的作用，或者我们类比C++，认为所有实现了这两个方法的数据类型都是myinterface的子类，而相比myinterface，这些子类，不仅提供了方法的具体实现，还提供了数据成员，可以看到myinterface是没有数据成员的。当然，go的interface类型里面只能声明方法，不能声明数据成员：
```go
type myinterface interface {
	s int
	hello()
	world() string
}
```
这样的定义是错误的。

一般我们这样使用interface：

```go
//定义一个新的类型 wen
type wen struct {
    w int
    m string
}

func (w wen) hello(){
    //do something
}

func (w wen)world() string{
    //do something
    return ""
}

//定义一个新的类型 mike
type mike struct{
    d string
    f int
}

func (m mike)hello(){
    //do something
}
func (m mike)world()string{
    //do something
    return ""
}

//定义一个使用myinterface的函数：
func test(p myinterface){
    p.hello()
    p.world()
}

func main(){
    var a wen
    var b mike

    test(a)
    test(b)
}
```
如上面的例子利用interface实现了C++中多态的效果。

interface的实现
---------
前面的文章我们曾经介绍过空interface的实现，本节介绍带有方法的interface的实现。go源码中非空interface的实现结构体为：
```go
type iface struct {
	tab  *itab //第一个字段也是_type
	data unsafe.Pointer
}
```
可以看到相比于eface，iface的类型字段要复杂一些 itab:
```go
type itab struct {
	inter *interfacetype
	_type *_type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```
看到了一个func指针数组，这就是golang中实现类型方法的原理了，和C++的虚函数表很像。可以看到fun是一个固定的数组，也就是说它里面包含的方法数量是一致的，赋值类型的多余方法将不会被看到。

从注释可以发现，inter表示定义的接口类型，_type表示赋值的数据类型。比如对于：
```go
var b mike
var j myinterface = b
```
变量j的inter就是myinterface，_type就是mike。

**method list的构造**
在_type和inter的基础上，interface就可以构造它的实现method list。本质上是求两个类型的方法的交集，但是由于go预先对方法进行了排序，因此整个算法复杂度为O(N+M)，N，M分别是inter和_type实现的方法数目，我们可以看一下这个函数的实现：
```go
func (m *itab) init() string {
	inter := m.inter
	typ := m._type
	x := typ.uncommon()

	// both inter and typ have method sorted by name,
	// and interface names are unique,
	// so can iterate over both in lock step;
	// the loop is O(ni+nt) not O(ni*nt).
	ni := len(inter.mhdr)
	nt := int(x.mcount)
	xmhdr := (*[1 << 16]method)(add(unsafe.Pointer(x), uintptr(x.moff)))[:nt:nt]
	j := 0
imethods:
	for k := 0; k < ni; k++ {
		i := &inter.mhdr[k]
		itype := inter.typ.typeOff(i.ityp)
		name := inter.typ.nameOff(i.name)
		iname := name.name()
		ipkg := name.pkgPath()
		if ipkg == "" {
			ipkg = inter.pkgpath.name()
		}
		for ; j < nt; j++ {
			t := &xmhdr[j]
			tname := typ.nameOff(t.name)
			if typ.typeOff(t.mtyp) == itype && tname.name() == iname {
				pkgPath := tname.pkgPath()
				if pkgPath == "" {
					pkgPath = typ.nameOff(x.pkgpath).name()
				}
				if tname.isExported() || pkgPath == ipkg {
					if m != nil {
						ifn := typ.textOff(t.ifn)
						*(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*sys.PtrSize)) = ifn
					}
					continue imethods
				}
			}
		}
		// didn't find method
		m.fun[0] = 0
		return iname
	}
	m.hash = typ.hash
	return ""
}
```
该函数的目的是填充itab.fun数组。

**inter声明的方法原型都放在inter.mhdr中。**
**_type实现的方法信息放在typ.uncommon()中。**

这个函数通过依次从变量的_type中找到inter中声明的函数的实现指针来填充fun数组，来完成该变量的itab初始化。


type assertion
---------
类型断言是针对接口类型的一种运算，它可以从interface type中抽取出该interface type代表的具体类型。比如上面的例子，可以通过类型断言，从myinterface中抽取出mike类型，它的操作形式如下：

```go
t := inter.(T)
```
显然，它是interface泛型化的一个反过程。

每次对一个interface变量进行type assertion操作golang都会检查interfac中的_type是是否和操作符中的T是属于同一个类型，如果不是的话则会panic。

为了避免panic，可以这样写：
```go
t,ok := inter.(T)
```
通过ok可以判断是否可以被转换，如果类型不匹配也不会发生panic。

总结
---------
这篇文章简单介绍了golang interface中method的实现。通过本文的介绍，我们了解了go中method信息在interface type中的存储位置，同时也了解自定义类型的method信息的存储位置。我们还通过源码解析了golang构造method list的过程。和C++比较类似，go也是通过一个函数表来实现类似多态的行为，最后我们还涉及了interface  type assertion操作，它是一种从interface中提取该interface持有的具体类型的手段。
总的来说，interface是一种通过方法来区分类型的手段。实现相同方法的类型可以通过相同的interface联系起来，使得程序在运行时可以实现多态。也就是对外暴露的接口只需要表明参数接收这个interface，则实现这个interface方法的所有类型都可以作为这个接口的参数传递进去。
