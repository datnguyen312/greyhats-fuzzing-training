# greyhats-fuzzing-training

Greyhats Training: Introduction to Fuzzers.

## Objectives

1. Learn what some memory corruption vulnerabilities are and why they are important in vulnerability
   research.
2. Learn why crashes sometimes indicate a memory corruption vulnerability.
3. Learn what fuzzers are.
4. Learn how fuzzers help with finding crashes.
5. Learn what sanitizers are.
6. Build your first fuzzing harness with libfuzzer with an example vulnerable library.
7. Write exploits using the discovered vulnerabilities.

## Memory Corruption Vulnerabilities

Memory corruption vulnerabilities stem from the improper management or use of memory. The impact of
a memory corruption bug can vary from a simple denial-of-service condition to full-blown remote
code execution.

They typically involve accessing data that was not intended for use at a particular point in a
program by the developer. For example, the Heartbleed vulnerability is an example of an attacker
being able to trick the OpenSSL library into accessing and revealing more data (outside of the
normal boundaries) than that function call should have revealed.

Some of the common categories include:

1. Stack overflows - read/write outside a buffer on the stack.
2. Heap overflows - read/write outside a buffer on the heap.
3. Use of uninitialized values
4. Use after frees - Using an allocated region of memory after it has been freed.
5. Double frees - Freeing of an already freed resource.
6. Type confusion - Interpreting a region of memory as the wrong type

### Examples

These code snippets are some examples of memory corruption vulnerabilties.

#### Stack Overflow

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdint.h>

void vuln(char * input) {
    uint32_t canary = 0xdeadbeef;
    char buffer[24];
    printf("canary = 0x%x\n", canary);
    strcpy(buffer, input);
    printf("canary = 0x%x\n", canary);
}

int main() {
    char input[512] = {0};
    memset(input, 'A', sizeof(input) - 1);
    vuln(input);
}
```

#### Heap Overflow

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdint.h>

struct Object {
    char buffer[24];
    uint32_t canary;
};

void vuln(struct Object * object, char * input) {
    printf("object->canary = 0x%x\n", object->canary);
    strcpy(object->buffer, input);
    printf("object->canary = 0x%x\n", object->canary);
}

int main() {
    char input[512] = {0};
    memset(input, 'A', sizeof(input) - 1);
    struct Object * object = malloc(sizeof(struct Object));
    object->canary = 0x42424242;
    vuln(object, input);
}
```

#### Uninitialized Variables

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdint.h>

void vuln() {
    uint64_t uninit;
    printf("uninit = 0x%lx\n", uninit);
}

void set_vars() {
    uint64_t var;
    var = 0x4141414142424242;
    printf("var = 0x%lx\n", var);
}

int main() {
    vuln();
    set_vars();
    vuln();
}
```

#### Use After Free

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdint.h>

#define items 12

void vuln() {
    uint64_t * canary = malloc(sizeof(uint64_t) * items);
    memset(canary , 'A', sizeof(uint64_t) * items);
    for (int i = 0; i < items; i++) {
        printf("canary = 0x%lx\n", canary[i]);
    }

    free(canary);

    char * buffer = malloc(sizeof(uint64_t) * items);
    memset(buffer, 'B', sizeof(uint64_t) * items);
    for (int i = 0; i < items; i++) {
        printf("canary = 0x%lx\n", canary[i]);
    }
}

int main() {
    vuln();
}
```

#### Double Free

```c
```

#### Type Confusion

```c
```

## Why are crashes important?

Crashes are indicative of some issue during the program's runtime. It can be a side-effect of some
region of memory being corrupted. For instance, the return pointer is overwritten with an invalid
value, the program will crash when it tries to execute code at that invalid address.

## What are Fuzzers?

## Sanitizers

## LibFuzzer Tutorial

### Dependencies

Install the build dependencies.

```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y clang make
```

### Instrumenting the Library

### Writing the Harness

### Triaging the Crashes

### Writing the Exploit
