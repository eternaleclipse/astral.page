---
title: "Sneaky Snek: Messing with Python's Internal Functions (Part I)"
date: 2023-07-03 03:26:51
tags:
- tech
- python
- internals
- hacking
---

***Disclaimer:*** *No Pythons were harmed in the making of this post.*
***Version used:*** *CPython 3.8.10 on Ubuntu 20.04 x86_64 (WSL)*

Snakes can be scary, but Python is a good snek. Python is [our friend](https://www.youtube.com/watch?v=L-PI-YqDkwk)!
As all good friends must do, we will gaslight it into believing that [2 + 2 = 5](https://en.wikipedia.org/wiki/2_%2B_2_%3D_5).

As an interpreted script language, some parts of Python's runtime can be changed directly.
For example, we can easily replace built-in functions such as `int()`:
```Python
>>> int = lambda x: 1
>>> int(2)
1
>>> print = lambda x: None
>>> print("hello")
>>>
```

Good, we are getting intimate! But how do we modify arithmetic operations, stuff like `+`, `-`? When we implement our own class, it's [quite straightforward](https://www.marinamele.com/2014/04/modifying-add-method-of-python-class.html). By overriding `__add__` and `__radd__`, we can decide exactly what logic will take place when instances (objects) of our class are being added. For example, we can make a `Kitten` object, and then make `kitten1 + kitten2` return the string `"Huge pile of cat hair"`.

When it comes to literals (ex. `int`) though, Python does not provide us with means of changing the default behavior for `+`. The implementation is buried inside the CPython interpreter's C code, so we'll need to work a little harder.

### Locating the Function
Looking through the [CPython source code](https://github.com/python/cpython), in `Include/abstract.h`, we find the following function definition:
```C
/* Returns the result of adding o1 and o2, or NULL on failure.

   This is the equivalent of the Python expression: o1 + o2. */
PyAPI_FUNC(PyObject *) PyNumber_Add(PyObject *o1, PyObject *o2);
```

In theory, every `int` addition operation arrives here!
We can check that with `gdb`, but first, we'll need to set up a few things.


### Setup

Things we'll need:
- gdb
- A version of Python with debug symbols
- CPython source code that matches the version

In Ubuntu, the Python 3 debug symbols are shipped in the `python3-dbg` package.
**Note**: You may need to edit your APT configuration and [add debug symbol sources](https://wiki.ubuntu.com/Debug%20Symbol%20Packages).

```
➜  ~ sudo apt install -y python3-dbg
...
Setting up python3-dbg (3.8.2-0ubuntu2) ...
Processing triggers for man-db (2.9.1-1) ...
➜  ~ python3-dbg -V
Python 3.8.10
```

Good! We can download this version either from the git repo (v3.8.10 tag), or the official Python website. Let's download and extract it.

```
➜  ~ wget https://www.python.org/ftp/python/3.8.10/Python-3.8.10.tar.xz
...
2023-07-03 04:30:57 (9.00 MB/s) - ‘Python-3.8.10.tar.xz’ saved [18433456/18433456]

➜  ~ tar xvf Python-3.8.10.tar.xz
```

Let's see if we could get `gdb` to debug Python with source code support. The `dir Python-3.8.10/Python` command tells `gdb` to add the directory to the source search path, so it will be able to show source code.

```
➜  ~/pymod gdb python3-dbg
GNU gdb (GDB) 12.1
...
Reading symbols from python3-dbg...
(gdb) dir Python-3.8.10/Python
Source directories searched: /home/user/pymod/Python-3.8.10/Python:$cdir:$cwd
(gdb) b main
Breakpoint 1 at 0x426dd6: file ../Programs/python.c, line 15.
(gdb) r
Starting program: /usr/bin/python3-dbg
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, main (argc=1, argv=0x7fffffffdd28) at ../Programs/python.c:15
15      {
(gdb)
```

We got a breakpoint on `main`, awesome!

### Debugging the Interpreter

Now that we have everything set up correctly, let's see if we land in that `PyNumber_Add` function we saw earlier!
We'll close `gdb` and run Python again:
```
➜  ~ python3-dbg
>>>
```

Now we can attach to it with `gdb` and set a breakpoint on our target function.
**Note**: You may need to set `ptrace_scope` to 0.

```
➜  ~ gdb -p `pidof python3-dbg`
GNU gdb (GDB) 12.1
...
0x00007f6196ad3f7a in select () from /lib/x86_64-linux-gnu/libc.so.6
(gdb) b PyNumber_Add
Breakpoint 1 at 0x632512: file ../Objects/abstract.c, line 957.
(gdb) c
Continuing.
```

Let's do a simple addition:
```py
>>>  `0x1234 + 0x5678`
```

We get a breakpoint hit! let's look back at the function definition.
```
Breakpoint 1, PyNumber_Add (v=0x7f6196567f00, w=0x7f619649a800) at ../Objects/abstract.c:957
957     {
```

It receives two pointers of the type `PyObject *`. Without looking into the `PyObject` struct too much yet, we'll try to find our values somewhere in the memory pointed to by these arguments. We will use `x/10wx v` to e**x**amine **20** **w**ords (4-byte values, so, a total of 40 bytes), in he**x** format, in the memory pointed to by `v`.

```
(gdb) x/10wx v
0x7f6196567f00: 0x00000001      0x00000000      0x008f2fe0      0x00000000
0x7f6196567f10: 0x00000001      0x00000000      0x00001234      0xfdfdfdfd
0x7f6196567f20: 0xfdfdfdfd      0xdddddddd
(gdb)
```

Our value, `0x1234`, is at an offset of 24 bytes, so, 6 words (4-byte each). 
Let's print it directly for both arguments.
We need to cast `v` from `PyObject *` to `unsigned int *`, add the offset, and dereference the pointer (with `*`).

```
(gdb) p/x * ((unsigned int *) v + 6)
$46 = 0x1234
(gdb) p/x * ((unsigned int *) w + 6)
$47 = 0x5678
```

Good, now let's modify the result!
Here is the interesting part from the function:

```c
PyObject *
PyNumber_Add(PyObject *v, PyObject *w)
{
   PyObject *result = binary_op1(v, w, NB_SLOT(nb_add));
   if (result == Py_NotImplemented) { // We want to break here set the result
   ...
```

We can setup a simple hook using breakpoint commands.
This feature that allows you to run a `gdb` command when the debugger reaches a certain breakpoint.
We'll combine this with a conditional breakpoint - if `v` and `w` both contain 2, write 5 to the returned `result` object.

Using `Ctrl-x + a`, we can switch to source code view and find the target line to set the breakpoint on - In my case, it is `abstract.c:959`.

```
(gdb) b abstract.c:959 if ((* ((unsigned int *) v + 6) == 2) && (* ((unsigned int *) v + 6) == 2))
Breakpoint 2 at 0x63252a: file ../Objects/abstract.c, line 959.
(gdb) commands
Type commands for breakpoint(s) 2, one per line.
End with a line saying just "end".
>set * ((unsigned int *) result + 6) = 5
>c
>end
(gdb) c
```

Let's see what happens.
```py
>>> 1 + 1
2
>>> 2 + 2
5
>>>
```

We are successfully intercepting number addition operations with `gdb`!1

In the [Next part](/2023/07/03/Sneaky-Snek-Messing-with-Python-s-Internal-Functions-Part-II/), we will look at implementing this hook from within Python, without the use of a debugger.
