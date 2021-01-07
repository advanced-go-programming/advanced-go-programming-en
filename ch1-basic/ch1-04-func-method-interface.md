# 1.4 Functions, Methods and Interfaces

Functions correspond to sequences of operations and are the basic building blocks of a program. functions in Go are named and anonymous: named functions generally correspond to package-level functions and are a special case of anonymous functions. Methods are special functions bound to a specific type. Methods in Go are type-dependent and must be statically bound at compile time. Interfaces define collections of methods that depend on runtime interface objects, so the methods corresponding to interfaces are dynamically bound at runtime. the Go language implements the duck object-oriented model through an implicit interface mechanism.

The initialization and execution of a Go language program always starts with the `main.main` function. However, if the `main` package imports other packages, they are included in the `main` package in order (the order of import here depends on the specific implementation, and may generally be in the order of a string of file names or package pathnames). If a package is imported more than once, it will only be imported once at execution time. When a package is imported, if it also imports other packages, the other packages are included first, then the constants and variables of the package are created and initialized, and then the `init` functions of the package are called. are called sequentially (`init` is not a normal function and can be defined with more than one, so it cannot be called by other functions either). Finally, when all the package constants and variables in the `main` package have been created and initialized, and the `init` function has been executed, then the `main.main` function is entered and the program starts executing normally. The following diagram shows the sequence in which the Go program functions are started.

![](../images/ch1-11-init.ditaa.png)

*Figure 1-11 Package initialization flow*

Note that before the `main.main` function is executed, all code runs in the same goroutine, which is the main system thread of the program. Therefore, if a new goroutine is started inside the `init` function with the go keyword, the new goroutine may only be executed after entering the `main.main` function.

## 1.4.1 Functions

In Go, functions are the first class of objects that we can keep in variables. There are mainly named and anonymous functions, and package level functions are generally named functions, and named functions are a special case of anonymous functions. Of course, each type in Go can also have its own method, which is actually a kind of function.

```go
// 具名函数
func Add(a, b int) int {
	return a+b
}

// 匿名函数
var Add = func(a, b int) int {
	return a+b
}
```

Functions in Go can have multiple arguments and multiple return values, and both arguments and return values exchange data with the callee in the form of passed values. Syntactically, functions also support a variable number of arguments. A variable number of arguments must be the last argument to appear, and a variable number of arguments is actually a slice type argument.

```go
// 多个参数和多个返回值
func Swap(a, b int) (int, int) {
	return b, a
}

// 可变数量的参数
// more 对应 []int 切片类型
func Sum(a int, more ...int) int {
	for _, v := range more {
		a += v
	}
	return a
}
```

When the variable parameter is an empty interface type, whether the caller unwraps the variable parameter or not can lead to different results.

```go
func main() {
	var a = []interface{}{123, "abc"}

	Print(a...) // 123 abc
	Print(a)    // [123 abc]
}

func Print(a ...interface{}) {
	fmt.Println(a...)
}
```

The first `Print` call is passed with the parameter `a... `, which is equivalent to a direct call to `Print(123, "abc")`. The second `Print` call is passed with an unpacked `a`, which is equivalent to a direct call to `Print([]interface{}{123, "abc"})`.

Not only can the parameters of a function have names, but also the return value of the function can be named.

```go
func Find(m map[int]int, key int) (value int, ok bool) {
	value, ok = m[key]
	return
}
```

If the return value is named, the return value can be modified by name, or by the `defer` statement after the `return` statement.

```go
func Inc() (v int) {
	defer func(){ v++ } ()
	return 42
}
```

The `defer` statement delays the execution of an anonymous function because it captures the local variable `v` of an external function, which we generally call a closure. The closure does not access the captured external variable by passing it a value, but by reference.

This referential access to external variables by closures may lead to some implicit problems.

```go
func main() {
	for i := 0; i < 3; i++ {
		defer func(){ println(i) } ()
	}
}
// Output:
// 3
// 3
// 3
```

Because it is a closure, in the `for` iteration statement, each `defer` statement delayed execution of the function references the same `i` iteration variable, which has a value of 3 at the end of the loop, so the final output is all 3.

The idea to fix this is to generate unique variables for each `defer` function in each round of iteration. This can be done in the following two ways.

```go
func main() {
	for i := 0; i < 3; i++ {
		i := i // 定义一个循环体内局部变量i
		defer func(){ println(i) } ()
	}
}

func main() {
	for i := 0; i < 3; i++ {
		// 通过函数传入i
		// defer 语句会马上对调用参数求值
		defer func(i int){ println(i) } (i)
	}
}
```

The first way is to define another local variable inside the loop body, so that each iteration of the `defer` statement's closure function captures different variables whose values correspond to the values at the time of the iteration. The second way is to pass the iteration variables in through the arguments of the closure function, and the `defer` statement immediately evaluates the calling arguments. Both ways work. However, in general, it is not a good habit to execute `defer` statements inside `for` loops, and this is only an example and not recommended.

In Go, when a function is called with a slice as an argument, it sometimes gives the illusion that the argument is passed by reference: because the elements of the passed slice can be modified inside the called function. In fact, any situation where the calling argument can be modified by a function argument is because a pointer argument is passed explicitly or implicitly in the function argument. The specification of function parameter passing is more precisely to pass values only for fixed parts of the data structure, such as strings or slices corresponding to pointers in structures and string length structures, but not what the pointers indirectly point to. The meaning of slice-passing can be well understood by replacing the argument of the slice type with a structure like `reflect.SliceHeader`.

```go
func twice(x []int) {
	for i := range x {
		x[i] *= 2
	}
}

type IntSliceHeader struct {
	Data []int
	Len  int
	Cap  int
}

func twice(x IntSliceHeader) {
	for i := 0; i < x.Len; i++ {
		x.Data[i] *= 2
	}
}
```

Because the underlying array part of the slice is passed by an implicit pointer (the pointer itself is still passed, but the pointer points to the same data), the called function can modify the data in the slice of the calling argument by means of the pointer. In addition to data, the slice structure also contains slice length and slice capacity information, which are also passed as values. If the `Len` or `Cap` information is modified in the called function, it will not be reflected in the slices of the calling argument, and we usually update the previous slices by returning the modified slices. This is why the built-in `append` must return a slice.

There is no logical limit to the depth of recursive calls to Go functions, and the stack of function calls is not subject to overflow errors because Go dynamically adjusts the size of the function stack as needed. Each goroutine allocates a very small stack (4 or 8KB, depending on the implementation) when it is first started, and can be dynamically sized up to GB (depending on the implementation, 250MB for 32-bit architectures and 1GB for 64-bit architectures in current implementations). Prior to Go 1.4, Go's dynamic stack used a segmented dynamic stack, which in layman's terms means that a chain table is used to implement the dynamic stack, and the memory location of each node in the chain table does not change. However, the dynamic stack implemented in a linked table has a large impact on the performance of certain hot calls that result in different nodes across the chain, because the adjacent chain nodes are generally not adjacent to each other in memory, which increases the chance of CPU cache hit failure. In order to solve the CPU cache hit rate problem of hot calls, Go1.4 has since switched to a continuous dynamic stack implementation, which uses a dynamic array-like structure to represent the stack. However, the continuous dynamic stack also introduces a new problem: when the continuous stack grows dynamically, the previous data needs to be moved to a new memory space, which causes the addresses of all variables in the previous stack to change. Although Go automatically updates pointers to stack variables that have changed addresses at runtime, it is important to understand that pointers are no longer fixed in Go (so you can't just keep pointers to numeric variables, and Go addresses can't just be saved to environments that are not under GC control, so you can't hold addresses of Go objects in C for long periods of time when using CGO). .

Because the stack of Go language functions is automatically resized, the average Go programmer has little need to care about the mechanics of running the stack. The concepts of stack and heap are not even intentionally spoken about in the Go language specification. We have no way of knowing whether function arguments or local variables are stored on the stack or in the heap, we just need to know that they work properly. Take a look at the following example.

```go
func f(x int) *int {
	return &x
}

func g() int {
	x = new(int)
	return *x
}
```

The first function returns the address of the function's argument variable directly - which seems to be a no-no, because if the argument variable is on the stack, the stack variable is invalidated after the function returns, and the returned address should naturally be invalidated as well. But the Go compiler and runtime are much smarter than we are, and will make sure that the variable pointed to by the pointer is in the right place. In the second function, although the `new` function is called internally to create a pointer object of type `*int`, it still doesn't know exactly where it is stored. For programmers with C/C++ programming experience, it is important to emphasize that you don't have to care about the function stack and heap in Go, the compiler and runtime will take care of that for us; also don't assume that the location of the variable in memory is fixed, the pointer can change at any time, especially if you don't expect it to.

## 1.4.2 Methods

Methods are generally a feature of object-oriented programming (OOP). In C++, methods correspond to member functions of a class object and are associated with a virtual table on the concrete object. In Go, however, methods are associated with types, so that static binding of methods can be done at the compilation stage. An object-oriented program will use methods to express the operations corresponding to its properties, so that the user using the object does not need to manipulate the object directly, but does so with the help of methods. Object-oriented programming (OOP) entered the mainstream development field is generally considered to have started with C++, which is a compatible C language based on the support of object-oriented features such as class. Then Java programming is claimed to be a pure object-oriented language, because Java functions cannot exist independently, each function must belong to a class.

The ancestor of the Go language, C, is not an object-oriented language, but the File-related functions in the C standard library also use the idea of object-oriented programming. Here we implement a set of C-style File functions.

```go
// 文件对象
type File struct {
	fd int
}

// 打开文件
func OpenFile(name string) (f *File, err error) {
	// ...
}

// 关闭文件
func CloseFile(f *File) error {
	// ...
}

// 读文件数据
func ReadFile(f *File, offset int64, data []byte) int {
	// ...
}
```

Among them, `OpenFile` is similar to a constructor for opening a file object, `CloseFile` is similar to a destructor for closing a file object, and `ReadFile` is similar to a normal member function. `CloseFile` and `ReadFile`, as ordinary functions, need to occupy name resources in the package level space. However, the `CloseFile` and `ReadFile` functions only operate on objects of type `File`, and this is when we would prefer such functions to be tightly bound to the type of the object they operate on.

This is done in Go by moving the first argument of the `CloseFile` and `ReadFile` functions to the beginning of the function name.

```go
// 关闭文件
func (f *File) CloseFile() error {
	// ...
}

// 读文件数据
func (f *File) ReadFile(offset int64, data []byte) int {
	// ...
}
```

In this way, the `CloseFile` and `ReadFile` functions become methods unique to the `File` type (rather than `File` object methods). They also no longer take up name resources in the package level space, and the `File` type already specifies the object they operate on, so the method names are generally simplified to `Close` and `Read`.

```go
// 关闭文件
func (f *File) Close() error {
	// ...
}

// 读文件数据
func (f *File) Read(offset int64, data []byte) int {
	// ...
}
```

Moving the first function argument to the front of the function is a small change from a code perspective, but from a programming philosophy perspective, the Go language is well into the ranks of object-oriented languages. We can add one or more methods to any custom type. The method corresponding to each type must be in the same package as the type definition, so it is not possible to add a method to a built-in type like `int` (because the method definition and the type definition are not in the same package). The name of each method must be unique for a given type, and methods, like functions, do not support overloading.

Methods are evolved from functions, but the first object parameter of the function has been moved to the front of the function name. So we can still use methods according to the original procedural thinking. Methods can be reduced to ordinary types of functions by calling them by the properties of method expressions.

```go
// 不依赖具体的文件对象
// func CloseFile(f *File) error
var CloseFile = (*File).Close

// 不依赖具体的文件对象
// func ReadFile(f *File, offset int64, data []byte) int
var ReadFile = (*File).Read

// 文件处理
f, _ := OpenFile("foo.dat")
ReadFile(f, 0, data)
CloseFile(f)
```

In some scenarios are more concerned with a set of similar operations: for example, `Read` reads some array and then calls `Close` to close it. In this environment, the user doesn't care about the type of the operation object, as long as the generic `Read` and `Close` behavior can be satisfied. However, in the method expressions, because the `ReadFile` and `CloseFile` function parameters contain `File` as a specific type parameter, this makes it impossible to seamlessly adapt `File`-related methods to other objects that are not of type `File` but have the same `Read` and `Close` methods. We Go coders can eliminate the difference in the type of the first argument in method expressions by combining the closure feature with.

```go
// 先打开文件对象
f, _ := OpenFile("foo.dat")

// 绑定到了 f 对象
// func Close() error
var Close = func() error {
	return (*File).Close(f)
}

// 绑定到了 f 对象
// func Read(offset int64, data []byte) int
var Read = func(offset int64, data []byte) int {
	return (*File).Read(f, offset, data)
}

// 文件处理
Read(0, data)
Close()
```

This happens to be the problem that method-values also have to solve. We can simplify the implementation with the method-value feature.

```go
// 先打开文件对象
f, _ := OpenFile("foo.dat")

// 方法值: 绑定到了 f 对象
// func Close() error
var Close = f.Close

// 方法值: 绑定到了 f 对象
// func Read(offset int64, data []byte) int
var Read = f.Read

// 文件处理
Read(0, data)
Close()
```

Go does not support the inheritance features of traditional object-oriented languages, but rather supports method inheritance in its own combinatorial way, which is implemented in Go by placing anonymous members within structures.

```go
import "image/color"

type Point struct{ X, Y float64 }

type ColoredPoint struct {
	Point
	Color color.RGBA
}
```

Although we could define `ColoredPoint` as a flat structure with three fields, we embed `Point` here to provide the two fields `X` and `Y`.

```go
var cp ColoredPoint
cp.X = 1
fmt.Println(cp.Point.X) // "1"
cp.Point.Y = 2
fmt.Println(cp.Y)       // "2"
```

By embedding anonymous members, we can inherit not only the internal members of the anonymous members, but also the methods corresponding to the anonymous member types. We generally think of Point as the base class and ColoredPoint as its inherited class or subclass. However, methods inherited in this way do not implement the polymorphic nature of virtual functions in C++. The receiver parameter of all inherited methods is still that anonymous member itself, not the current variable.

```go
type Cache struct {
	m map[string]string
	sync.Mutex
}

func (p *Cache) Lookup(key string) string {
	p.Lock()
	defer p.Unlock()

	return p.m[key]
}
```

The `Cache` structure type inherits its `Lock` and `Unlock` methods by embedding an anonymous `sync.Mutex`. But when calling `p.Lock()` and `p.Unlock()`, `p` is not the real recipient of the `Lock` and `Unlock` methods, but expands them into `p.Mutex.Lock()` and `p.Mutex.Unlock()` calls. This expansion is done at compile time, and has no runtime cost.

In traditional object-oriented languages (eg. C++ or Java), methods of subclasses are dynamically bound to objects at runtime, so some methods implemented in the base class may see `this` as an object other than the one corresponding to the base class type, a feature that causes uncertainty in the operation of base class methods. In Go, the base class method is "inherited" by embedding an anonymous member, `this` is the object of the type that implements the method, and the method is statically bound at compile time in Go. If we need the polymorphic nature of virtual functions, we need to use Go interfaces to implement them

## 1.4.3 Interfaces


Rob Pike, the father of the Go language, famously said that languages that try to disallow idiocy eventually become idiotic themselves. Static programming languages in general have a strict type system, which allows the compiler to check in depth that the programmer has not done anything out of the ordinary. The Go language tries to give programmers a balance between safe and flexible programming. It makes it relatively easy to program safely and dynamically by providing strict type checking while implementing support for duck types through interface types.

Go's interface types are an abstraction and generalization of the behavior of other types; because interface types are not tied to specific implementation details, by abstracting in this way we can make objects more flexible and adaptable. Many object-oriented languages have a similar concept of interfaces, but what is unique about interface types in Go is that they are duck types that satisfy an implicit implementation. A duck type means that if it walks like a duck and quacks like a duck, then it can be used as a duck, and this is the case with object-oriented Go, where if an object looks like an implementation of an interface type, then it can be used as that interface type. This design allows you to create a new interface type that satisfies existing concrete types without breaking their original definitions; it is especially flexible and useful when we are using types from packages that are not under our control. interface types in the Go language are deferred bindings that enable polymorphic functionality like virtual functions.

In the "Hello world" example, the `fmt.Printf` function is designed to be completely interface-based, and its real function is performed by the `fmt.Fprintf` function. The `error` type, which is used to represent errors, is even a built-in interface type. In C, `printf` can only print a limited number of basic data types to a file object. Fprintf` can print to any custom output stream object, either to a file or to standard output, to the network, or even to a compressed file; at the same time, the data printed is not limited to the language's built-in base types, but can be printed to any object that implicitly satisfies the `fmt. Stringer` interface can be printed, and objects that do not satisfy the `fmt. The signature of the `fmt.Fprintf` function is as follows.

```go
func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)
```

`io.Writer` is the interface used for output and `error` is the built-in error interface, which are defined as follows.

```go
type io.Writer interface {
	Write(p []byte) (n int, err error)
}

type error interface {
	Error() string
}
```

We can output each character after converting it to uppercase by customizing our own output object as follows.

```go
type UpperWriter struct {
	io.Writer
}

func (p *UpperWriter) Write(data []byte) (n int, err error) {
	return p.Writer.Write(bytes.ToUpper(data))
}

func main() {
	fmt.Fprintln(&UpperWriter{os.Stdout}, "hello, world")
}
```

Of course, we can also define our own print format to achieve the effect of converting each character to uppercase and then outputting it. For each object to be printed, if the `fmt.Stringer` interface is satisfied, the result returned by the object's `String` method is printed by default using.

```go
type UpperString string

func (s UpperString) String() string {
	return strings.ToUpper(string(s))
}

type fmt.Stringer interface {
	String() string
}

func main() {
	fmt.Fprintln(os.Stdout, UpperString("hello, world"))
}
```

Go does not support implicit conversions for base types (non-interface types), and we cannot assign a value of type `int` directly to a variable of type `int64`, nor can we assign a value of type `int` to a variable of a newly defined named type whose underlying type is `int`. language is very flexible with respect to the conversion of interface types. Conversions between objects and interfaces, and conversions between interfaces and interfaces may be implicit conversions. You can see the following example.

```go
var (
	a io.ReadCloser = (*os.File)(f) // 隐式转换, *os.File 满足 io.ReadCloser 接口
	b io.Reader     = a             // 隐式转换, io.ReadCloser 满足 io.Reader 接口
	c io.Closer     = a             // 隐式转换, io.ReadCloser 满足 io.Closer 接口
	d io.Reader     = c.(io.Reader) // 显式转换, io.Closer 不满足 io.Reader 接口
)
```

Sometimes there is too much flexibility between objects and interfaces, leading us to artificially restrict this unintentional adaptation. A common approach is to define a special method with special methods to distinguish between interfaces. For example, the `Error` interface in the `runtime` package defines a special `RuntimeError` method to avoid inadvertent adaptation of the interface by other types.

```go
type runtime.Error interface {
	error

	// RuntimeError is a no-op function but
	// serves to distinguish types that are run time
	// errors from ordinary errors: a type is a
	// run time error if it has a RuntimeError method.
	RuntimeError()
}
```

In protobuf, the `Message` interface takes a similar approach, also defining a specific `ProtoMessage` to avoid inadvertent adaptation of the interface by other types.

```go
type proto.Message interface {
	Reset()
	String() string
	ProtoMessage()
}
```

But this approach is only a gentleman's agreement, and it would be easy for someone to deliberately forge a `proto.Message` interface. A stricter approach would be to define a private method for the interface. Only objects that satisfy this private method can satisfy the interface, and the name of the private method contains the absolute path name of the package, so the private method can only be implemented inside the package to satisfy the interface. The `testing.TB` interface in the test package uses a similar technique.

```go
type testing.TB interface {
	Error(args ...interface{})
	Errorf(format string, args ...interface{})
	...

	// A private method to prevent users implementing the
	// interface and so future additions to it will not
	// violate Go 1 compatibility.
	private()
}
```

However, this approach of prohibiting external objects from implementing the interface through private methods comes at a cost: first, the interface can only be used internally by the package, and external packages cannot normally create objects that satisfy the interface directly; second, this protection is not absolute, and malicious users can still bypass this protection mechanism.

In the previous section on methods we talked about the possibility of inheriting anonymously typed methods by embedding an anonymously typed member in a structure. In fact, this embedded anonymous member does not have to be a normal type, but can also be an interface type. We can fake private `private` methods by embedding an anonymous `testing.TB` interface, because interface methods are delayed binding and it does not matter if the `private` method really exists at compile time.

```go
package main

import (
	"fmt"
	"testing"
)

type TB struct {
	testing.TB
}

func (p *TB) Fatal(args ...interface{}) {
	fmt.Println("TB.Fatal disabled!")
}

func main() {
	var tb testing.TB = new(TB)
	tb.Fatal("Hello, playground")
}
```

We re-implement the `Fatal` method in our own `TB` structure type, then call our own `Fatal` method through the `testing.TB` interface by implicitly converting the object to the `testing.TB` interface type (which is satisfied by the anonymous `testing.TB` object embedded in it), and then call our own TB` interface to call our own `Fatal` method.

This kind of inheritance by embedding anonymous interfaces or embedding anonymous pointer objects is in fact a pure virtual inheritance, where we inherit only the specification specified by the interface, and the real implementation is injected at runtime. For example, we can simulate the implementation of a gRPC plugin.

```go
type grpcPlugin struct {
	*generator.Generator
}

func (p *grpcPlugin) Name() string { return "grpc" }

func (p *grpcPlugin) Init(g *generator.Generator) {
	p.Generator = g
}

func (p *grpcPlugin) GenerateImports(file *generator.FileDescriptor) {
	if len(file.Service) == 0 {
		return
	}

	p.P(`import "google.golang.org/grpc"`)
	// ...
}
```

The constructed `grpcPlugin` type object must satisfy the `generate.Plugin` interface (in the package "github.com/golang/protobuf/protoc-gen-go/generator").

```go
type Plugin interface {
	// Name identifies the plugin.
	Name() string
	// Init is called once after data structures are built but before
	// code generation begins.
	Init(g *Generator)
	// Generate produces the code generated by the plugin for this file,
	// except for the imports, by calling the generator's methods
	// P, In, and Out.
	Generate(file *FileDescriptor)
	// GenerateImports produces the import declarations for this file.
	// It is called after Generate.
	GenerateImports(file *FileDescriptor)
}
```

The `p.P(...)' function used in the `GenerateImports` method of the `grpcPlugin` type corresponding to the `generate.Plugin` interface is implemented through the `Generator. ` function is implemented through the `generator.Generator` object injected by the `Init` function. Generator` corresponds to a concrete type, but if `generator.Generator` is an interface type we can even pass in a direct implementation.

It's really incredible how easily Go implements advanced features like duck object-oriented and virtual inheritance through a combination of a few simple features.
