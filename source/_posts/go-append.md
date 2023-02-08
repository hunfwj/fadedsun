---
title: go append函数详解
date: 2021-05-27 20:14:45
tags:
---

函数声明如下：
```
// The append built-in function appends elements to the end of a slice. If
// it has sufficient capacity, the destination is resliced to accommodate the
// new elements. If it does not, a new underlying array will be allocated.
// Append returns the updated slice. It is therefore necessary to store the
// result of append, often in the variable holding the slice itself:
//	slice = append(slice, elem1, elem2)
//	slice = append(slice, anotherSlice...)
// As a special case, it is legal to append a string to a byte slice, like this:
//	slice = append([]byte("hello "), "world"...)
func append(slice []Type, elems ...Type) []Type
```
<!-- more -->
append函数的作用是将一些元素追加到切片的末尾， 并返回新的切片

此处的新， 只是可能。只有切片底层空间不够， 扩容的时候， 返回的切片地址才会更改。
如果底层空间足够， 就会发现两次的地址是一致的。

测试程序
```
func TestAppend(t *testing.T) {
	num:= make([]int, 0, 5)
	//num := []int{1, 2, 3}  // 使用此方式初始化， 追加元素的时候， 地址就会更改
	fmt.Printf("%p \n", num)

	res := append(num, 4)
	fmt.Printf("%p \n", res)
}
```
输出
```
=== RUN   TestAppend
0xc00001a0c0 
0xc00001a0c0 
--- PASS: TestAppend (0.00s)
PASS
```

当我们观察到空间不够的时候， append函数会自动扩容。自然就会想要去了解扩容的机制。

```
package service

import (
	"testing"
)

func TestAppend(t *testing.T) {
	num:= make([]int, 0, 5)
	res := append(num, 4)
	print(res)
}
```

查看汇编
```
go tool compile -S biz/service/service_test.go
```
可以看到当空间足够的时候， 是直接执行MOVQ指令

```
        0x0033 00051 (biz/service/service_test.go:9)    MOVQ    $4, ""..autotmp_3+24(SP)
```

```
package service

import (
	"testing"
)

func TestAppend(t *testing.T) {
	num:= []int{1, 2, 3}
	res := append(num, 4)
	print(res)
}

```
当空间不够时， 可以看到如下指令
可以看到关键的一行runtime.growslice(SB)
```
        0x004d 00077 (biz/service/service_test.go:9)    PCDATA  $0, $1
        0x004d 00077 (biz/service/service_test.go:9)    LEAQ    type.int(SB), AX
        0x0054 00084 (biz/service/service_test.go:9)    PCDATA  $0, $0
        0x0054 00084 (biz/service/service_test.go:9)    MOVQ    AX, (SP)
        0x0058 00088 (biz/service/service_test.go:9)    PCDATA  $0, $1
        0x0058 00088 (biz/service/service_test.go:9)    LEAQ    ""..autotmp_5+80(SP), AX
        0x005d 00093 (biz/service/service_test.go:9)    PCDATA  $0, $0
        0x005d 00093 (biz/service/service_test.go:9)    MOVQ    AX, 8(SP)
        0x0062 00098 (biz/service/service_test.go:9)    MOVQ    $3, 16(SP)
        0x006b 00107 (biz/service/service_test.go:9)    MOVQ    $3, 24(SP)
        0x0074 00116 (biz/service/service_test.go:9)    MOVQ    $4, 32(SP)
        0x007d 00125 (biz/service/service_test.go:9)    CALL    runtime.growslice(SB)
        0x0082 00130 (biz/service/service_test.go:9)    PCDATA  $0, $1
        0x0082 00130 (biz/service/service_test.go:9)    MOVQ    40(SP), AX
        0x0087 00135 (biz/service/service_test.go:9)    PCDATA  $1, $1
        0x0087 00135 (biz/service/service_test.go:9)    MOVQ    AX, "".res.ptr+104(SP)
        0x008c 00140 (biz/service/service_test.go:9)    MOVQ    48(SP), CX
        0x0091 00145 (biz/service/service_test.go:9)    MOVQ    CX, ""..autotmp_12+72(SP)
        0x0096 00150 (biz/service/service_test.go:9)    MOVQ    56(SP), DX
        0x009b 00155 (biz/service/service_test.go:9)    MOVQ    DX, "".res.cap+64(SP)
        0x00a0 00160 (biz/service/service_test.go:9)    PCDATA  $0, $0
        0x00a0 00160 (biz/service/service_test.go:9)    MOVQ    $4, 24(AX)
```

现在我们找到了扩容的代码。
```
// growslice handles slice growth during append.
// It is passed the slice element type, the old slice, and the desired new minimum capacity,
// and it returns a new slice with at least that capacity, with the old data
// copied into it.
// The new slice's length is set to the old slice's length,
// NOT to the new requested capacity.
// This is for codegen convenience. The old slice's length is used immediately
// to calculate where to write new values during an append.
// TODO: When the old backend is gone, reconsider this decision.
// The SSA backend might prefer the new length or to return only ptr/cap and save stack space.
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```

有空再补充一下代码解读吧

切片定义
```
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```