# Notes

This section contains notes that cames from my study about FFI (Foreign function interface).

## Go 

### Pass Go function to C

Starting from Go 1.6 cgo has new rules.

> Go code may pass a Go pointer to C provided that the Go memory to which it points does not contain any Go pointers.

See: https://github.com/golang/go/issues/12416

These rules are checked during the runtime, and if violated program crashes. At the moment it is possible to disable checks using GODEBUG=cgocheck=0 environment variable. But in future that might stop working.

So it is not possible anymore to pass a pointer to C code, if the memory to which it is pointing stores a Go function/method pointer. There are several ways to overcome this limitations, but I guess in most of them you should store a synchronized data structure which represents the correspondence between a certain id and the actual pointer. This way you can pass an id to the C code, not a pointer.

The code solving this problem might look like **[this](1_method.go)**.

For more information see: https://github.com/golang/go/wiki/cgo#function-variables


It depends exactly what you need to do with the callback function - but a trick that might work is to not pass the Go function, but a pointer to a struct with the function on it that you want.

The code solving this problem might look like **[this](2_method.go)**.

Another option might be to use a global variable as you had in your example, and then forgo passing anything useful in the pointer to C.

The code solving this problem might look like **[this](3_method.go)**.

Reference: https://stackoverflow.com/questions/37157379/passing-function-pointer-to-the-c-code-using-cgo

[Call Go functions from C](https://stackoverflow.com/questions/6125683/call-go-functions-from-c)

### Pass Go Array and Slice to C

The **C** meta package allows the interaction between Go and C. In some cases you
need to pass a Go Array or Slice to C function that usually receive two 
parameters an integer that represents the number of elements in the collection
and a pointer to the first element in the collection.

The most important thing in this case is to cast to the right type otherwise you
will risk to access to unexpected area of memory.
 
Here I report two examples:

- **[Pass Go Array to C](pass-array-to-c.go)**
- **[Pass Go Slice to C](pass-slice-to-c.go)**

```go
/*
#include <stdlib.h>
static int execute(size_t size, int32_t* data) {
	int32_t count = 0;
	size_t i = size;
	while(i > 0) {
		count += data[i - 1];
		i -= 1;
    }
	return count;
}
static int executePtr(size_t size, int32_t* data) {
	int32_t count = 0;
	while(size > 0) {
		count += *data++;
		size -= 1;
    }
	return count;
}
*/
import "C"
import (
	"fmt"
	"unsafe"
)

func main() {
	slice := []int32{10, 20, 30, 40, 50}
	raw := unsafe.Pointer(&slice[0])
	fmt.Println("Length of slice is ", len(slice))
	fmt.Println("Capacity of slice is ", cap(slice))
	fmt.Println(C.execute(C.size_t(len(slice)), (*C.int32_t)(raw)))
	fmt.Println(C.executePtr(C.size_t(len(slice)), (*C.int32_t)(raw)))
}
```

### cgo

If a Go source file imports `"C"`, it is using cgo. The Go file will have access 
to anything appearing in the comment immediately preceding the line `import "C"`,
and will be linked against **all other cgo comments in other Go files**, and all 
C files included in the build process.

Note that there must be no blank lines in between the cgo comment and the import 
statement. This is the main source of errors on starting to use **cgo**.

Since variable argument methods like printf aren't supported yet, we will wrap it
in the C a custom C function. See the following example:

```go
package cgoexample

/*
#include <stdio.h>
#include <stdlib.h>

void myprint(char* s) {
	printf("%s\n", s);
}
*/
import "C"

import "unsafe"

func Example() {
	cs := C.CString("Hello from stdio\n")
	C.myprint(cs)
	C.free(unsafe.Pointer(cs))
}
```

Go makes its functions available to C code through use of a special `//export` 
comment. If the Go file contains `//export` directive then it's only possible
include a declaration for a C function or atherwise it's alway passible declare
the function as `static inline`.

### Go and C strings

Go and C represent string object in a different ways. Go strings are the 
combination of a length and a pointer to the first character. C strings are the 
pointer to the first character and are terminated by the null character `\0`.

Go provides 3 utility functions to allow the usage of strings both in Go and C:

- `func C.CString(goString string) *C.char`
- `func C.GoString(cString *C.char) string`
- `func C.GoStringN(cString *C.char, length C.int) string`

One of the important thing to remember is that the `C.CString()` function will
allocate a new memory space for the C string and this means that will be the 
responsibility of the developer manage the new allocated memory. The common 
pattern is reported below: 

```go
// #include <stdlib.h>
import "C"
import "unsafe"

	var cmsg *C.char = C.CString("hi")
	// The responsibility of manage the allocated memory for a C string is not of
	// the gargage collector but of the developer
	defer C.free(unsafe.Pointer(cmsg))
	// do something with the C string
```

A C function may be declared in the Go file with a parameter type of the special 
name **`_GoString_`**.

```C
size_t _GoStringLen(_GoString_ s);
const char *_GoStringPtr(_GoString_ s);
```

### Go and C arrays

C arrays in  C are a sequanche of characters with null termination character or
a sequence of memory element and a defined length.

Go provide the following function to transform a C array in a Go slice of `byte`:

- `func C.GoBytes(cArray unsafe.Pointer, length C.int) []byte`

It is important to keep in mind that the Go garbage collector will not interact 
with this data, and that if it is freed from the C side of things, the behavior 
of any Go code using the slice is nondeterministic.

### Environment variables

Remember that `os.Getenv()` does not see the environment variables set by 
`C.setenv()`.

### Numeric types

The standard C numeric types are available under the names C.char, 
C.schar (signed char), C.uchar (unsigned char), C.short, 
C.ushort (unsigned short), C.int, C.uint (unsigned int), C.long, 
C.ulong (unsigned long), C.longlong (long long), 
C.ulonglong (unsigned long long), C.float, C.double, 
C.complexfloat (complex float), and C.complexdouble (complex double). 
The C type void* is represented by Go's unsafe.Pointer. The C types __int128_t 
and __uint128_t are represented by [16]byte.

A few special C types which would normally be represented by a pointer type in Go
are instead represented by a **uintptr**.

To access a struct, union, or enum type directly, *prefix* it with **struct_**, 
**union_**, or **enum_**, as in **C.struct_stat**.

The size of any C type T is available as **C.sizeof_T**, as in **C.sizeof_struct_stat**.