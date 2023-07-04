---
title: 'Sneaky Snek: Messing with Python''s Internal Functions (Part II)'
date: 2023-07-03 21:34:31
tags:
- tech
- python
- internals
- hacking
---

Greetings, fellow hackers and tinkerers, and welcome to the second installation of the "Sneaky Snek" blog post series! This series is dedicated to having fun messing around with CPython's core functionality and modifying arithmetic operator implementation in runtime. In the [Previous Part](/2023/07/03/Sneaky-Snek-Messing-with-Python-s-Internal-Functions-Part-I/), we've started diving into CPython's implementation of the `+` operator, `PyNumber_Add`, intercepted number addition operations using `gdb` breakpoint commands, inspected the `PyObject` arguments and modified the result in an automated fashion.

In this part, we will explore inline hooking of the function from within a Python script using `ctypes`.

### Initial Recon
First, we will recall the function definition:
```C
PyAPI_FUNC(PyObject *) PyNumber_Add(PyObject *o1, PyObject *o2);
```

Let's examine `PyNumber_Add` and see where it's code resides in memory.
```
(gdb) x/10i PyNumber_Add
   0x632512 <PyNumber_Add>:     endbr64
   0x632516 <PyNumber_Add+4>:   push   %r12
   0x632518 <PyNumber_Add+6>:   push   %rbp
   0x632519 <PyNumber_Add+7>:   push   %rbx
   0x63251a <PyNumber_Add+8>:   mov    %rdi,%rbx
   0x63251d <PyNumber_Add+11>:  mov    %rsi,%rbp
   0x632520 <PyNumber_Add+14>:  mov    $0x0,%edx
   0x632525 <PyNumber_Add+19>:  call   0x630fec <binary_op1>
   0x63252a <PyNumber_Add+24>:  cmp    $0x8f5ef0,%rax
   0x632530 <PyNumber_Add+30>:  je     0x632537 <PyNumber_Add+37>
(gdb) info proc map
      Start Addr           End Addr       Size     Offset  Perms  objfile
        0x400000           0x423000    0x23000        0x0  r--p   /usr/bin/python3.8d
        0x423000           0x691000   0x26e000    0x23000  r-xp   /usr/bin/python3.8d
        0x691000           0x8e7000   0x256000   0x291000  r--p   /usr/bin/python3.8d
...
```

Great, we have at least 30 bytes of code. That's more than enough to insert a `jmp`! There is a problem, though - our code is mapped to a segment with `r-xp` permissions, so we are [missing](https://www.youtube.com/watch?v=otCpCn0l4Wo) the "write" permission. This is common for code segments in programs, as there is usually no need to have write permissions for code segments (unless [JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation) is being used). We will deal with this later.

### The Plan

Our objective is to trick Python into believing that "2 + 2 = 5".
For this purpose, we will redirect execution to a custom function that will check the operands and modify the result if necessary.

We want a Python script that will:
- Create a custom version of the function
- Load it into Python's memory
- Locate the `PyNumber_Add`'s memory address
- Change it's page permissions to writable
- Patch the first bytes of `PyNumber_Add` with a jump to our function
- Run some tests

Let's break these down, one by one.

### Creating `my_add`
While there are advanced ways of inserting code into the memory space of a running process, we will keep things simple and go with loading a shared object (`.so`). Python has it's own [C API](https://docs.python.org/3/c-api/) that allows us to extend Python, deal with `PyObject`s and even run Python from C. This will come in handy!

**Note**: You may need to install `python3-dev` for the CPython headers.

Since we are planning to practically destroy `PyNumber_Add`, we will need a different way to calculate the result. There are safer ways to implement a generic and reusable hook, but for our purposes and in the spirit of having fun, I've decided on a quirky approach: `a + b` is mathematically equivalent to `a - (-b)`! We will use this to our advantage and implement `my_add` as follows:

```C
#include <Python.h>

PyObject* my_add(PyObject* a, PyObject* b) {
    // Increase reference count to prevent garbage collection
    Py_INCREF(a);
    Py_INCREF(b);

    // Check if both arguments are ints
    if (PyLong_Check(a) && PyLong_Check(b)) {
        // Check if both arguments are 2
        if (PyLong_AsLong(a) == 2 && PyLong_AsLong(b) == 2) {
            Py_DECREF(a);
            Py_DECREF(b);

            // Return 5
            return PyLong_FromLong(5);
        }

        // Return a - (-b)
        PyObject* neg_b = PyNumber_Negative(b);
        PyObject* result = PyNumber_Subtract(a, neg_b);
        Py_DECREF(a);
        Py_DECREF(b);
        Py_DECREF(neg_b);
        return result;
    }

    // Invalid arguments, return Py_None
    Py_DECREF(a);
    Py_DECREF(b);
    return Py_None;    
}
```

We will compile this into a shared object using the following command:
```bash
gcc -shared -fPIC -o my_add.so my_add.c $(python3-config --cflags --ldflags)
```

### Loading `my_add` into Python's memory
We can now use `ctypes.CDLL()` to load `my_add.so` into Python's memory and get a pointer to our function. This proves to be a little tricky, because `ctypes` does not expose a direct way to get a pointer to a function. `ctypes` does, however, allow us to get a pointer to the variable that holds the function pointer. We can then dereference it to get the actual function pointer.

```python
import ctypes

def deref_ptr(ptr):
    return ctypes.cast(ptr, ctypes.POINTER(ctypes.c_void_p)).contents.value

my_add = deref_ptr(ctypes.addressof(ctypes.CDLL('./my_add.so').my_add))
```

### Locating `PyNumber_Add`'s memory address
We can use `ctypes` again to get a pointer to `PyNumber_Add` (It knows...!):
```python
pynumber_add = deref_ptr(ctypes.addressof(ctypes.pythonapi.PyNumber_Add))
```

### Changing `PyNumber_Add`'s page permissions to writable
We will use [`mprotect()`](https://linux.die.net/man/2/mprotect) to change the page permissions of `PyNumber_Add` to `rwx` and make the memory writable. To grab the page address, we can use a simple trick ~~stolen~~ borrowed from [this StackOverflow answer](https://stackoverflow.com/a/6387879/3722114). Oh yeah, and `ctypes` let's us call `mprotect()` directly from Python!

```python
page_size = os.sysconf('SC_PAGE_SIZE')
page_addr = pynumber_add & ~(page_size - 1)
page_perms = 0x1 | 0x2 | 0x4  # PROT_READ | PROT_WRITE | PROT_EXEC
libc = ctypes.CDLL(ctypes.util.find_library('c'))
result = libc.mprotect(page_addr, page_size, page_perms)
print(f"mprotect(addr={hex(page_addr)}, size={hex(page_size)}, perms={hex(page_perms)}) ret={result} errno={ctypes.get_errno()}")
```

*Note*: If we wanted to support Windows, we could use [`VirtualProtect()`](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualprotect) instead of `mprotect()`.

### Patching the first bytes of `PyNumber_Add` with a jump to our function
Let's code a simple trampoline that will redirect execution to `my_add`:
```asm
mov rax, 0x123456789abcdef0
jmp rax
```

Assembling this code, we get some bytes. Let's use them and replace the `0x123456789abcdef0` with the address of `my_add`:
```python
shellcode = b'\x48\xb8' + my_add.to_bytes(8, 'little') + b'\xff\xe0'
shellcode_size = len(shellcode)
shellcode = ctypes.c_char_p(shellcode)
print("Address of shellcode trampoline:", hex(ctypes.addressof(shellcode)))
```

At this point, we have everything we need to patch `PyNumber_Add` with a jump to `my_add`. We will use `ctypes` again to write our shellcode to the first bytes of `PyNumber_Add`:
```python
result = ctypes.memmove(pynumber_add, shellcode, shellcode_size)
print("memmove:", hex(result))
```

### Adding some tests
We'll insert some number addition operations and see what happens. There is a caveat to be aware of - Python calculates `2 + 2` at compile time and stores the result in the bytecode. We can use `eval()` to force Python to recalculate the result every time.

```python
print("1 + 1 =", eval("1 + 1"))
print("2 + 2 =", eval("2 + 2"))
```

### Putting it all together

```py
import ctypes
import ctypes.util
import os
import sys

def deref_ptr(ptr):
    return ctypes.cast(ptr, ctypes.POINTER(ctypes.c_void_p)).contents.value

# Get a pointer to my_add
my_add = deref_ptr(ctypes.addressof(ctypes.CDLL('./my_add.so').my_add))
print("Address of my_add:", hex(my_add))

# Get a pointer to PyNumber_Add
pynumber_add = deref_ptr(ctypes.addressof(ctypes.pythonapi.PyNumber_Add))
print("Address of PyNumber_Add:", hex(pynumber_add))

# Use ctypes and mprotect to make the memory writable
page_size = os.sysconf('SC_PAGE_SIZE')
page_addr = pynumber_add & ~(page_size - 1)
page_perms = 0x1 | 0x2 | 0x4  # PROT_READ | PROT_WRITE | PROT_EXEC
libc = ctypes.CDLL(ctypes.util.find_library('c'))
result = libc.mprotect(page_addr, page_size, page_perms)
print(f"mprotect(addr={hex(page_addr)}, size={hex(page_size)}, perms={hex(page_perms)}) ret={result} errno={ctypes.get_errno()}")

# Create a trampoline to our function
shellcode = b"\x48\xb8" + my_add.to_bytes(8, byteorder='little') # mov rax, my_add
shellcode += b"\xff\xe0" # jmp rax
shellcode_size = len(shellcode)

# Get a pointer to our shellcode trampoline
shellcode = ctypes.c_char_p(shellcode)
print("Address of shellcode trampoline:", hex(ctypes.addressof(shellcode)))

# Copy the shellcode to the start of PyNumber_Add
result = ctypes.memmove(pynumber_add, shellcode, shellcode_size)
print("memmove:", hex(result))

# Test
print("1 + 1 =", eval("1 + 1"))
print("2 + 2 =", eval("2 + 2"))
```

### Hammer Time!
Let's run our script and see what happens:
```bash
➜  ~ python snek.py
Address of my_add: 0x7f1070ce51a0
Address of PyNumber_Add: 0x50f750
mprotect(addr=0x50f000, size=0x1000, perms=0x7) ret=0 errno=0
Address of shellcode trampoline: 0x7f1070c24390
memmove: 0x50f750
1 + 1 = 2
2 + 2 = 5
```

It works!! Good snek!

I hope you enjoyed taking this journey into Python internals with me!
In the [Next Part](/2023/07/04/Sneaky-Snek-Messing-with-Python-s-Internal-Functions-Part-III/), we will explore further extending this into a backdoor that given specific input, will give us a reverse shell.
