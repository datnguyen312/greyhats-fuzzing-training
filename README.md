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
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdint.h>

void vuln() {
    char * one = malloc(24);
    char * two = malloc(24);
    char * three = malloc(24);
    printf("one ptr = 0x%p\n", one);
    printf("two ptr = 0x%p\n", two);
    printf("three ptr = 0x%p\n", three);

    free(one);
    puts("free one");
    free(two);
    puts("free two");
    free(one);
    puts("free one");

    char * four = malloc(24);
    char * five = malloc(24);
    char * six = malloc(24);
    printf("four ptr = 0x%p\n", four);
    printf("five ptr = 0x%p\n", five);
    printf("six ptr = 0x%p\n", six);
}

int main() {
    vuln();
}
```

#### Type Confusion

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdint.h>

struct TypeOne {
    char buffer[48];
    uint32_t canary;
};

struct TypeTwo {
    char buffer[24];
    uint32_t canary;
};

void right(void * object) {
    struct TypeOne * converted = (struct TypeOne *) object;
    printf("canary = 0x%x\n", converted->canary);
}

void vuln(void * object) {
    struct TypeTwo * converted = (struct TypeTwo *) object;
    printf("canary = 0x%x\n", converted->canary);
}

int main() {
    struct TypeOne object;
    memset(object.buffer, 'A', sizeof(struct TypeOne));
    object.canary = 0x42424242;
    right(&object);

    vuln(&object);
}
```

## Why are crashes important?

Crashes are indicative of some issue during the program's runtime. It can be a side-effect of some
region of memory being corrupted. For instance, the return pointer is overwritten with an invalid
value, the program will crash when it tries to execute code at that invalid address.

## What are Fuzzers?

See the presentation derived from Terry Chia's talk at CenCon.

## Sanitizers

See the presentation derived from Terry Chia's talk at CenCon.

## LibFuzzer Tutorial

### Dependencies

Install the build dependencies.

```
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install -y clang make gdb
```

### Writing the Harness

If we look at the `main.c` file, we can see that the API includes the `parse_data` call.

```c
#include "vulnerable.h"

int main() {
    char * ptr;

    // Print Date
    struct DateCommand {
        uint32_t id;
        char payload[24];
    } date_command;
    date_command.id = 0xfefefefe;
    strcpy(date_command.payload, "/bin/date");
    ptr = (char *) &date_command;
    parse_data(ptr, sizeof(date_command));

    // Command One
    struct OneCommand {
        uint32_t id;
        uint8_t size;
    } one_command;
    one_command.id = 0;
    one_command.size = 9 + 12;
    ptr = (char *) &one_command;
    parse_data(ptr, sizeof(one_command));

    // Command Two
    struct TwoCommand {
        uint32_t id;
        uint8_t payload[4];
    } two_command;
    two_command.id = 1;
    two_command.payload[0] = 2;
    two_command.payload[1] = 3;
    two_command.payload[2] = 2;
    two_command.payload[3] = 0;
    ptr = (char *) &two_command;
    parse_data(ptr, sizeof(two_command));

    return 0;
}
```

We can use this in our harness. The entry point for the fuzzer looks like this:

```c
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  // DoSomethingWithData(Data, Size);
  return 0;
}
```

It is trivial to apply this to our example.

```c
#include <stdint.h>
#include <stddef.h>
#include "vulnerable.h"

int LLVMFuzzerTestOneInput(const uint8_t * data, size_t size) {
    parse_data((char *) data, size);
    return 0;
}
```

### Instrumenting the Harness and Library

Now, we need to build the harness. To begin with, we can just start with instrumentation and no
sanitizers.

```
clang -g -O1 -o fuzz-basic -fsanitize=fuzzer harness.c vulnerable.c
```

Additionally, we can also build it with ASAN.

```
clang -g -O1 -o fuzz-asan -fsanitize=fuzzer,address harness.c vulnerable.c
```

### Running the Fuzzer

To start with, we will run the fuzzer without any corpus. A benefit of libFuzzer is that it can
generate testcases without any initial input to mutate.

```
./fuzz-basic
./fuzz-asan
```

### Triaging the Crashes

First, we can replay the crash by passing it as an argument.

```
./fuzz-asan crash-ac16616a28de5270a35e47ae291e59144f1cbdea
```

Notice that the symbols aren't resolved. We need to properly set the symbolizer path.

```
sudo ln -s /usr/bin/llvm-symbolizer-6.0 /usr/bin/llvm-symbolizer
````

This will give us some more information about the crash.

We will use GDB to further debug the crashes manually.
