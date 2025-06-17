## Go-Panikint

### Overview

`Go-Panikint` is a modified version of the Go compiler that adds **automatic overflow/underflow detection** for signed integer arithmetic operations. When overflow is detected, a **panic** with an "integer overflow" message will pop.

It can handle addition `+`, subtraction `-`, multiplication `*`, and division `/` for types `int8`, `int16`, `int32`. The division case specifically detects the MIN_INT / -1 overflow condition where the mathematical result exceeds the maximum representable value. Regarding `int64`, `uintptr`, they are not checked.

### Usage and installation :
```bash
# Clone, change dir and compile the compiler
git clone https://github.com/kevin-valerio/go-panikint && cd go-panikint/src && ./make.bash

# Full path to the root of the forked compiler
export GOROOT=/path/to/go-panikint

# Compile and run a Go program
./bin/go run test_simple_overflow.go

# Compile only
./bin/go build test_simple_overflow.go

# Fuzz only
./bin/go test -fuzz=FuzzIntegerOverflow -v
```

### How does it work ?
#### What is being done exactly ?
We basically patched the intermediate representation (IR) part of the Go compiler so that, on every math operands (i.e `OADD`, `OMUL`, `OSUB`, `ODIV`, ...), the compiler does not only add the IR opcodes that perform the math operation but also **insert** a bunch of checks for arithmetic bugs and insert a panic call with `ir.Syms.Panicoverflow` if those checks are met. This code will ultimately end up to the binary code (Assembly) of the application, so use with caution.
Below is an example of a Ghidra-decompiled addition `+`:

```c++
if (*x_00 == '+') {
  val = (uint32)*(undefined8 *)(puVar9 + 0x60);
  sVar23 = val + sVar21;
  puVar17 = puVar9 + 8;
  if (((sdword)val < 0 && sVar21 < 0) && (sdword)val < sVar23 ||
      ((sdword)val >= 0 && sVar21 >= 0) && sVar23 < (sdword)val) {
    runtime.panicoverflow(); // <-- panic if overflow caught
  }
  goto LAB_1000a10d4;
}
```

#### Why do we use package filters ?
As you can see [here](https://github.com/kevin-valerio/go-panikint/blob/0d6340a37c6cc7a3e44c556f8df42e8ec9d1efc8/src/cmd/compile/internal/ssagen/ssa.go#L5258-L5270), we do not apply our panics for arithmetic issues on every packages: standard library packages (`runtime`, `sync`, `os`, `syscall`, etc.), internal packages (`internal/*`) and math and unsafe packages are not concerned by the overhead. There is different reasons for this.

First, we need to compile Go, with Go itself. Because of this, some behaviors of the vanilla compiler must remain. Some functions like hashing sometimes rely on overflow for bit mixing. Moreover, some components like routine scheduling or memory management cannot be interrupted with panics.

### Examples

#### Example 1:

```go
package main

import "fmt"

func main() {
	fmt.Println("Testing overflow detection...")

	// Test int8 addition overflow
	var a int8 = 127
	var b int8 = 1
	fmt.Printf("Before: a=%d, b=%d\n", a, b)
	result := a + b  // Should panic with "integer overflow"
	fmt.Printf("After: result=%d\n", result)
}
```

**Expected output:**

```bash
bash-5.2$ GOROOT=/path/to/go-panikint && ./bin/go run test_simple_overflow.go
Testing overflow detection...
Before: a=127, b=1
panic: runtime error: integer overflow

goroutine 1 [running]:
main.main()
	/path/to/go-panikint/test_simple_overflow.go:12 +0xfc
exit status 2
```

#### Example 2:

```bash
bash-5.2$ go run  vim_panik.go
1. Signed int8 overflow:
Max int8: 127
After adding 1: -128 (wrapped to minimum)

bash-5.2$ GOROOT=/path/to/go-panikint && ./bin/go run  vim_panik.go
1. Signed int8 overflow:
Max int8: 127
panic: runtime error: integer overflow

goroutine 1 [running]:
main.main()
	/path/to/go-panikint/vim_panik.go:14 +0x11c
exit status 2
```

#### Example 3 (fuzzing):
**Fuzzing harness:**
```go
package fuzztest
import "testing"

func FuzzIntegerOverflow(f *testing.F) {
	f.Fuzz(func(t *testing.T, a, b int8) {
		result := a + b
		t.Logf("%d + %d = %d", a, b, result)
	})
}
```

**Output:**
```bash
GOROOT=/path/to/go-panikint/go-panikint  ../bin/go test -fuzz=FuzzIntegerOverflow -v
=== RUN   FuzzIntegerOverflow
fuzz: elapsed: 0s, gathering baseline coverage: 0/15 completed
fuzz: elapsed: 0s, gathering baseline coverage: 9/15 completed
--- FAIL: FuzzIntegerOverflow (0.03s)
    --- FAIL: FuzzIntegerOverflow (0.00s)
        testing.go:1822: panic: runtime error: integer overflow
            goroutine 23 [running]:
            runtime/debug.Stack()
            	/path/to/go-panikintgo-panikint/src/runtime/debug/stack.go:26 +0xc4
            testing.tRunner.func1()
            	/path/to/go-panikintgo-panikint/src/testing/testing.go:1822 +0x220
            panic({0x100e27b00?, 0x100fa4d60?})
            	/path/to/go-panikintgo-panikint/src/runtime/panic.go:783 +0x120
            fuzztest.FuzzIntegerOverflow.func1(0x0?, 0x0?, 0x0?)
            	/path/to/go-panikintgo-panikint/fuzz_test/fuzz_test.go:10 +0xf8
            reflect.Value.call({0x100e24ca0?, 0x100e5d5f8?, 0x14000093e28?}, {0x100dba042, 0x4}, {0x1400012a180, 0x3, 0x0?})
            	/path/to/go-panikintgo-panikint/src/reflect/value.go:581 +0x960
            reflect.Value.Call({0x100e24ca0?, 0x100e5d5f8?, 0x14000154000?}, {0x1400012a180?, 0x100e5ce40?, 0x100d2abf3?})
            	/path/to/go-panikintgo-panikint/src/reflect/value.go:365 +0x94
            testing.(*F).Fuzz.func1.1(0x140001028c0?)
            	/path/to/go-panikintgo-panikint/src/testing/fuzz.go:341 +0x258
            testing.tRunner(0x140001028c0, 0x14000152000)
            	/path/to/go-panikintgo-panikint/src/testing/testing.go:1931 +0xc8
            created by testing.(*F).Fuzz.func1 in goroutine 7
            	/path/to/go-panikintgo-panikint/src/testing/fuzz.go:328 +0x4a4


    Failing input written to testdata/fuzz/FuzzIntegerOverflow/8a8466cc4de923f0
    To re-run:
    go test -run=FuzzIntegerOverflow/8a8466cc4de923f0
```
