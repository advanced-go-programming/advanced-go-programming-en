# Appendix A: Common Pitfalls in the Go Language

The common Go language traps listed here all conform to the Go language syntax and can be compiled properly, but may run with incorrect results or risk resource leaks.

## Variable parameters are empty interface types

When the variable parameter of the argument is the empty interface type, attention needs to be paid to parameter expansion when passing in the slices of the empty interface.

```go
func main() {
	var a = []interface{}{1, 2, 3}

	fmt.Println(a)
	fmt.Println(a...)
}
```

The compiler cannot find the error, whether it is expanded or not, but the output is different: the

```
[1 2 3]
1 2 3
```

## Arrays are passed by value

In function call parameters, arrays are value-passed and it is not possible to return the result by modifying the parameters of the array type.

```go
func main() {
	x := [3]int{1, 2, 3}

	func(arr [3]int) {
		arr[0] = 7
		fmt.Println(arr)
	}(x)

	fmt.Println(x)
}
```

The use of slices is required when necessary.

## map traversal is not sequential

map is a hash table implementation, and the order of each traversal may be different.

```go
func main() {
	m := map[string]string{
		"1": "1",
		"2": "2",
		"3": "3",
	}

	for k, v := range m {
		println(k, v)
	}
}
```

## The return value is shadowed

Masking of local variables with the same name within a named return value in a local scope.

```go
func Foo() (err error) {
	if err := Bar(); err != nil {
		return
	}
	return
}
```

## recover must be run in the defer function

The recover catches exceptions on grandfathered calls and is not valid on direct calls.

```go
func main() {
	recover()
	panic(1)
}
```

Direct defer calls are also invalid: 

```go
func main() {
	defer recover()
	panic(1)
}
```

Multiple layers of nesting are still invalid when defer is called.

```go
func main() {
	defer func() {
		func() { recover() }()
	}()
	panic(1)
}
```

Must be called directly in the defer function to be valid.

```go
func main() {
	defer func() {
		recover()
	}()
	panic(1)
}
```

## The main function retires early

Background Goroutine cannot guarantee the completion of tasks.

```go
func main() {
	go println("hello")
}
```

## use Sleep avoid concurrency problem

Sleep does not guarantee the output of a complete string: the

```go
func main() {
	go println("hello")
	time.Sleep(time.Second)
}
```

Similarly, by inserting the scheduling statement.

```go
func main() {
	go println("hello")
	runtime.Gosched()
}
```

## CPU exclusive use leads to starvation of other Goroutines(solved after 1.14)

Goroutine is cooperative preemption scheduling, and Goroutine itself does not actively give the CPU.

```go
func main() {
	runtime.GOMAXPROCS(1)

	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(i)
		}
	}()

	for {} // running on a core
}
```

The solution is to add the runtime.Gosched() scheduler function to the for loop.

```go
func main() {
	runtime.GOMAXPROCS(1)

	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(i)
		}
	}()

	for {
		runtime.Gosched()
	}
}
```

Or, to avoid CPU usage by blocking.

```go
func main() {
	runtime.GOMAXPROCS(1)

	go func() {
		for i := 0; i < 10; i++ {
			fmt.Println(i)
		}
		os.Exit(0)
	}()

	select{}
}
```

## Sequential consistency memory model not satisfied between different Goroutines

Because in different Goroutines, the main function is not guaranteed to print `hello, world`:

```go
var msg string
var done bool

func setup() {
	msg = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	println(msg)
}
```

The solution is to use explicit synchronization.

```go
var msg string
var done = make(chan bool)

func setup() {
	msg = "hello, world"
	done <- true
}

func main() {
	go setup()
	<-done
	println(msg)
}
```

The msg is written before the channel is sent, so it is guaranteed to print `hello, world`.

## Closures incorrectly refer to the same variable

```go
func main() {
	for i := 0; i < 5; i++ {
		defer func() {
			println(i)
		}()
	}
}
```

An improved approach is to generate a local variable in each iteration.

```go
func main() {
	for i := 0; i < 5; i++ {
		i := i
		defer func() {
			println(i)
		}()
	}
}
```

Or, passed in via function parameters.

```go
func main() {
	for i := 0; i < 5; i++ {
		defer func(i int) {
			println(i)
		}(i)
	}
}
```

## Execute the defer statement inside the loop

defer is executed only when the function exits, and executing defer at for causes a delay in releasing resources.

```go
func main() {
	for i := 0; i < 5; i++ {
		f, err := os.Open("/path/to/file")
		if err != nil {
			log.Fatal(err)
		}
		defer f.Close()
	}
}
```

The solution could be to construct a local function in for and execute defer inside the local function.

```go
func main() {
	for i := 0; i < 5; i++ {
		func() {
			f, err := os.Open("/path/to/file")
			if err != nil {
				log.Fatal(err)
			}
			defer f.Close()
		}()
	}
}
```

## Slicing will cause the entire underlying array to be locked

Slicing causes the entire underlying array to be locked and the underlying array cannot be freed from memory. If the underlying array is large it can put a lot of pressure on memory.

```go
func main() {
	headerMap := make(map[string][]byte)

	for i := 0; i < 5; i++ {
		name := "/path/to/file"
		data, err := ioutil.ReadFile(name)
		if err != nil {
			log.Fatal(err)
		}
		headerMap[name] = data[:1]
	}

	// do some thing
}
```

The solution is to clone a copy of the array, which frees up the underlying array:

```go
func main() {
	headerMap := make(map[string][]byte)

	for i := 0; i < 5; i++ {
		name := "/path/to/file"
		data, err := ioutil.ReadFile(name)
		if err != nil {
			log.Fatal(err)
		}
		headerMap[name] = append([]byte{}, data[:1]...)
	}

	// do some thing
}
```

## Null pointers and null interfaces are not equal

For example, an error pointer is returned, but not the empty error interface.

```go
func returnsError() error {
	var p *MyError = nil
	if bad() {
		p = ErrBad
	}
	return p // Will always return a non-nil error.
}
```

## Memory address will change

Addresses of objects in Go may change, so pointers cannot be generated from values of other non-pointer types.

```go
func main() {
	var x int = 42
	var p uintptr = uintptr(unsafe.Pointer(&x))

	runtime.GC()
	var px *int = (*int)(unsafe.Pointer(p))
	println(*px)
}
```

When a memory change occurs, the associated pointer is updated synchronously, but the non-pointer type uintptr does not do synchronous updates.

Similarly, CGO cannot store Go object addresses.

## Goroutine leaks

The Go language has automatic memory reclamation, so memory is not generally leaked. However, Goroutine does leak, and the memory referenced by the leaked Goroutine is also not reclaimed.

```go
func main() {
	ch := func() <-chan int {
		ch := make(chan int)
		go func() {
			for i := 0; ; i++ {
				ch <- i
			}
		} ()
		return ch
	}()

	for v := range ch {
		fmt.Println(v)
		if v == 5 {
			break
		}
	}
}
```

In the above program, the background Goroutine inputs a sequence of natural numbers into the pipeline, and the main function outputs the sequence. But when break jumps out of the for loop, the background Goroutine is in a state that cannot be recycled.

We can avoid this problem by using the context package.

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())

	ch := func(ctx context.Context) <-chan int {
		ch := make(chan int)
		go func() {
			for i := 0; ; i++ {
				select {
				case <- ctx.Done():
					return
				case ch <- i:
				}
			}
		} ()
		return ch
	}(ctx)

	for v := range ch {
		fmt.Println(v)
		if v == 5 {
			cancel()
			break
		}
	}
}
```

When the main function jumps out of the loop at break, it notifies the background Goroutine to exit by calling `cancel()`, thus avoiding Goroutine leakage.
