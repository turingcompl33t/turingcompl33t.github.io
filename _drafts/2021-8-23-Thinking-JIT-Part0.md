---
layout: "post"
title: "Thinking Just-in-Time, Part 0: World's Simplest JIT"
---

This post is the first in a series on JIT compilation.

### A First Attempt

```c
void* jit_buffer_alloc(size_t size) {
  void* buffer = malloc(size);
  if (NULL == buffer) fatal_error("malloc()");
  return buffer;
}
```

```c
void emit_code(void* ptr, byte_t* code, size_t length) {
  memcpy(ptr, code, length);
}
```

```c
void* buffer = jit_buffer_alloc(BUFFER_SIZE);

/**
 * int return_one() { return 1; }
 */
byte_t code[] = {
  0xB8, 0x01, 0x00, 0x00, 0x00, // mov eax, 0x1
  0xC3                          // ret
};

emit_code(buffer, code, sizeof(code));
```

```c
// Tell the compiler how to call our function
return_one_f fn = buffer;

// Call our function
int one = fn();
```

```bash
$ ./driver0 
[1]    59813 segmentation fault (core dumped)  ./driver0
```

```
$ gdb driver0
...
(gdb) run
Starting program: driver0 

Program received signal SIGSEGV, Segmentation fault.
0x00005555555592a0 in ?? ()
(gdb) bt
#0  0x00005555555592a0 in ?? ()
#1  0x00005555555552b8 in main () at driver0.c:54
```

The offending line in the source code, line 54, which (it should come as no surprise) is the line on which we invoke our JITed code: `int one = fn();`.

GDB gives us a couple other pieces of useful information: the address of the call site for our JITed function in the body of `main()`: `0x00005555555552b8`, and the address of our JITed function itself: `0x00005555555592a0`. 

We can use the `pmap` tool to get some details of the virtual address space for the process in which `driver0` executes.

```
$ pmap 60538
60538:   driver0
0000555555554000      4K r---- driver0
0000555555555000      4K r-x-- driver0
0000555555556000      4K r---- driver0
0000555555557000      4K r---- driver0
0000555555558000      4K rw--- driver0
...
 total             2368K
```

The `main()` function resides in the region that begins at address `0000555555555000`. This region has permissions `r-x--`. In contrast, our JITed function is located in the region that begins at address `0000555555558000` which has permissions `rw---`. Importantly, our JITed code lies in a region that does not have execute (`x`) permissions, so when we try to execute code in this region we get a segmentation fault.

### A Working JIT

- Allocate a page-aligned, writable region of memory
- Write code to this memory region
- Change the protections of this memory region, removing write permissions and adding execute permissions
- Run code in the now-executable region

```c
void* jit_buffer_alloc(size_t size) {
  void* ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if (MAP_FAILED == ptr) fatal_error("mmap()");
  return ptr;
}

void jit_buffer_free(void* ptr, size_t size) {
  if (munmap(ptr, size) == -1) {
    fatal_error("munmap()");
  }
}
```

```c
void jit_buffer_finish(void* ptr, size_t size) {
  if (mprotect(ptr, size, PROT_READ | PROT_EXEC) == -1) {
    fatal_error("mprotect()");
  }
}
```

Now the following works:

```c
/**
 * int return_one() { return 1; }
 */
byte_t code[] = {
  0xB8, 0x01, 0x00, 0x00, 0x00, // mov eax, 0x1
  0xC3                          // ret
};

void* buffer = jit_buffer_alloc(BUFFER_SIZE);
emit_code(buffer, code, sizeof(code));
jit_buffer_finish(buffer, BUFFER_SIZE);
int one = ((return_one_f)buffer)();
```

### Accepting Arguments

```c
/**
 * int add(int x, int y) {
 *  return x + y;
 * }
 */
byte_t code[] = {
  0x48, 0x89, 0xF8,   // mov rax, rdi
  0x48, 0x01, 0xF0,   // add rax, rsi
  0xC3                // ret
};
```

```c
int three = ((add_f)buffer)(1, 2);
```
