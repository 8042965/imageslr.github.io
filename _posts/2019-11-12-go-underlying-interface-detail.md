---
layout: post
title: 📗【Go 原理】详解 interface
date: 2019/11/12 19:00
permalink: 2019/11/12/go-underlying-interface-detail.html
---

## 接口类型
Go 语言有两种接口类型，一种是带有方法的接口，通过 `type Name interface {}` 声明，表示为 `iface` 结构体；另一种是不带有任何方法的 `interface{}` 类型，表示为 `eface` 结构体。

## 接口类型的底层实现
[【Go 原理】详解 nil]({% post_url 2019-11-12-go-underlying-nil-detail %}) 中说道：
> 在底层，interface 作为两个成员来实现：一个类型和一个值 `(type, value)`。比如对于 int 值 3， 一个接口值示意性地包含 `(int, 3)`。
> 
> 接口的零值是 `(nil, nil)`，只有该接口内部的值和类型都是 `nil` 时它才等于 `nil`。

这其实是一种比较笼统的说法。对于带有方法的接口类型和不带任何方法的 `interface{}` 类型，底层实现有所区别。

不带任何方法的 `interface{}` 类型底层实现为 `eface` 结构体：
```go
type eface struct { // 16 bytes
    _type *_type
    data  unsafe.Pointer
}
```
其中第一个字段指向底层类型，第二个字段指向底层数据(具体值）。

而带有方法的接口类型底层实现为 `iface` 结构体：
```go
ype iface struct { // 16 bytes
    tab  *itab
    data unsafe.Pointer
}
```
其中第二个字段同样指向底层数据(具体值），第一个字段除了包含底层类型外，还包含更丰富的信息。以下是 `itab` 结构体的定义：
```go
type itab struct { // 32 bytes
    inter *interfacetype
    _type *_type
    hash  uint32 // copy of _type.hash. Used for type switches.
    _     [4]byte
    fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```
* `_type` 就是接口的底层类型
* `hash` 字段是 `_type.hash` 的拷贝，用于将 interface 转换为具体类型时快速判断目标类型和接口的底层类型是否一致
* `inter` 描述接口类型
* `fun` 是一个指针数组，每个指针指向具体类型 `_type` 实现的具体方法，在通过一个接口去调用某个方法的时候，会在这个表格中查找，然后使用具体值去调用正确的方法。如果该数组为空表示类型 `_type` 没有实现 `inter` 接口

再回头说一下接口类型的第二个字段 `data`。如果接口类型的具体值是一个值类型，那么 data 指向的是具体值的拷贝；否则，data 指向具体值的地址。以 file 类型和 Reader 接口为例：
```go
f := file.Open('...')
var reader Reader = f.reader()
```
`reader` 变量底层的 `data` 字段实际指向的是 f 的一个副本。

---
参考资料：
* [浅谈 Go 语言实现原理 · 2.2 接口 - draveness](https://draveness.me/golang/basic/golang-interface.html)