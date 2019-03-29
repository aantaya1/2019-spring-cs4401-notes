# JIT Compiler and Security Vulnerabilities

This article will briefly connect some of the things we have learned in
lecture and through assignments to the JIT compiler.

## Motivation

At one point in lecture we discussed how modern hardware doesn't allow
the memory spaces to be both writable and executable. I looked into this a little bit more and started researching into 'Writing XOR Executable'
(W xor X) memory protection policy, which doesn't allow for the same memory
space to be both writable and executable. However, while researching that I
stumbled upon the JIT compiler.

## The JIT compiler

All of the code and content of this article originated
[here](https://eli.thegreenplace.net/2013/11/05/how-to-jit-an-introduction). I am just summarizing it and connecting it back to some of the things we have discussed in class.

JIT is simply an acronym for "Just In Time". First, let's define what "a JIT" actually refers to. Whenever a program, while running, creates and runs some new executable code which was not part of the program when it was stored on disk, it’s a JIT.

JIT technology is easier to explain when divided into two distinct phases:

* **Phase 1: create machine code at program run-time.**
* **Phase 2: execute that machine code, also at program run-time.**

Phase 1 is where 99% of the challenges of JITing are. But it's also the less mystical part of the process, because this is exactly what a compiler does. Well known compilers like gcc and clang translate C/C++ source code into machine code. The machine code is emitted into an output stream, but it could very well be just kept in memory (and in fact, both gcc and clang/llvm have building blocks for keeping the code in memory for JIT execution). Phase 2 is what I want to focus on in this article.

Modern operating systems are picky about what they allow a program to do at runtime. The wild-west days of the past came to an end with the advent of protected mode, which allows an OS to restrict chunks of virtual memory with various permissions. So in "normal" code, you can create new data dynamically on the heap, but you can't just run stuff from the heap without asking the OS to explicitly allow it.

### The Code
The rest of this article contains code samples for a POSIX-compliant Unix OS (specifically Linux). On other OSes (like Windows) the code would be different in the details, but not in spirit. All modern OSes have convenient APIs to implement the same thing.

Here's how we dynamically create a function in memory and execute it.

#### The C Code We Will Be Emulating
```
long add4(long num) {
  return num + 4;
}
```

#### Dynamically Generate Code in JIT
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>


// Allocates RWX memory of given size and returns a pointer to it. On failure,
// prints out the error and returns NULL.
void* alloc_executable_memory(size_t size) {
  void* ptr = mmap(0, size,
                   PROT_READ | PROT_WRITE | PROT_EXEC,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if (ptr == (void*)-1) {
    perror("mmap");
    return NULL;
  }
  return ptr;
}

void emit_code_into_memory(unsigned char* m) {
  unsigned char code[] = {
    0x48, 0x89, 0xf8,                   // mov %rdi, %rax
    0x48, 0x83, 0xc0, 0x04,             // add $4, %rax
    0xc3                                // ret
  };
  memcpy(m, code, sizeof(code));
}

const size_t SIZE = 1024;
typedef long (*JittedFunc)(long);

// Allocates RWX memory directly.
void run_from_rwx() {
  void* m = alloc_executable_memory(SIZE);
  emit_code_into_memory(m);

  JittedFunc func = m;
  int result = func(2);
  printf("result = %d\n", result);
}
```

The main 3 steps performed by this code are:

* **Use mmap to allocate a readable, writable and executable chunk of memory on the heap.**
* **Copy the machine code implementing add4 into this chunk.**
* **Execute code from this chunk by casting it to a function pointer and calling through it.**

Note that step 3 can only happen because the memory chunk containing the machine code is executable. Without setting the right permission, that call would result in a runtime error from the OS (most likely a segmentation fault). This would happen if, for example, we allocated m with a regular call to malloc, which allocates readable and writable, but not executable memory.

#### Fixing Some of It's Security Vulnerabilities
The code shown above has a problem - it's a security hole. The reason is the RWX (Readable, Writable, eXecutable) chunk of memory it allocates - a paradise for attacks and exploits. So let's be a bit more responsible about it. Here's some slightly modified code:
```
// Allocates RW memory of given size and returns a pointer to it. On failure,
// prints out the error and returns NULL. Unlike malloc, the memory is allocated
// on a page boundary so it's suitable for calling mprotect.
void* alloc_writable_memory(size_t size) {
  void* ptr = mmap(0, size,
                   PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
  if (ptr == (void*)-1) {
    perror("mmap");
    return NULL;
  }
  return ptr;
}

// Sets a RX permission on the given memory, which must be page-aligned. Returns
// 0 on success. On failure, prints out the error and returns -1.
int make_memory_executable(void* m, size_t size) {
  if (mprotect(m, size, PROT_READ | PROT_EXEC) == -1) {
    perror("mprotect");
    return -1;
  }
  return 0;
}

// Allocates RW memory, emits the code into it and sets it to RX before
// executing.
void emit_to_rw_run_from_rx() {
  void* m = alloc_writable_memory(SIZE);
  emit_code_into_memory(m);
  make_memory_executable(m, SIZE);

  JittedFunc func = m;
  int result = func(2);
  printf("result = %d\n", result);
}
```
It's equivalent to the earlier snippet in all respects except one: the memory is first allocated with RW permissions (just like a normal malloc would do). This is all we really need to write our machine code into it. When the code is there, we use mprotect to change the chunk's permission from RW to RX, making it executable but no longer writable. So the effect is the same, but at no point in the execution of our program the chunk is both writable and executable, which is good from a security point of view.

## Authors

* **Alexander Antaya**
* **Eli Bendersky** - *Initial work* - [Eli Bendersky](https://eli.thegreenplace.net/2013/11/05/how-to-jit-an-introduction)
