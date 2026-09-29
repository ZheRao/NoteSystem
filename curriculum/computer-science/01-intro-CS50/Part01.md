# CS50 — Arrays

## Compilation

Recall:

```bash
make hello
./hello
```

`make` is **not itself a compiler**. It is a build tool that can invoke a compiler such as `clang`.

Using `clang` directly:

```bash
clang hello.c
```

By default, this creates an executable named `a.out`:

```bash
./a.out
```

To specify the executable's name:

```bash
clang -o hello hello.c
./hello
```

If the program uses an external library such as CS50:

```c
#include <cs50.h>
```

the library must also be **linked**:

```bash
clang -o hello hello.c -lcs50
```

`-lcs50` tells the linker to link the CS50 library into the program.

### Four Stages of Compilation

**1. Preprocessing**

Processes directives beginning with `#`, especially:

```c
#include <stdio.h>
```

Conceptually, the declarations from the header file are made available to the source code, such as the declaration of `printf`.

This allows the compiler to know that functions such as `printf` exist, what arguments they expect, and what they return.

**2. Compiling**

The compiler translates the preprocessed C source code into **assembly language**, containing instructions such as:

```text
mov
push
xor
```

Assembly is a human-readable, low-level representation of machine instructions.

**3. Assembling**

The assembler translates assembly language into **machine code** — binary instructions that the CPU can execute.

**4. Linking**

The linker combines the machine code produced from our source files with other required compiled code, such as libraries.

Conceptually:

```text
hello.c ──→ machine code ──┐
                           ├──→ executable
CS50 library ──────────────┘
```

The final `a.out` or `hello` is the **executable containing machine code**, not assembly.

---

## Memory

### Type Sizes

Typical sizes introduced in CS50:

```text
bool / char    1 byte
int / float    4 bytes
long / double  8 bytes
```

One **byte = 8 bits**.

These sizes mean values occupy actual finite regions of memory and therefore have finite representations.

### Memory Model

Conceptually, we can visualize memory as a large sequence/grid of **bytes**:

```text
[ byte ][ byte ][ byte ][ byte ][ byte ] ...
```

A `char`, for example, typically occupies one byte.

Larger types occupy multiple consecutive bytes.

---

## Arrays

### Contiguous Memory

An array is one region of **contiguous memory** containing multiple elements of the same type.

For example:

```c
int scores[3];
```

conceptually requests enough contiguous memory for three integers:

```text
scores
  ↓
[ int ][ int ][ int ]
   0      1      2
```

Rather than declaring three independent variables, we have one array and **index into it** to access individual elements.

### Array Length

Unlike higher-level structures such as Python lists, a C array does not inherently carry its length in a way that lets a function generally ask the array itself:

```text
"How many elements do you contain?"
```

The programmer often needs to track or pass the length separately.

### Characters as Integers

Characters are represented numerically according to an encoding such as ASCII.

For example:

```text
'H' → 72
'I' → 73
'!' → 33
```

Therefore, the bits in memory acquire meaning according to how the program interprets them.

### Strings

A C string is essentially an **array of characters** terminated by a special character:

```text
'H'  'I'  '!'  '\0'
```

or numerically:

```text
72   73   33    0
```

`\0` is the **NUL character** and has all bits set to zero:

```text
00000000
```

It marks where the string ends.

Therefore, `"HI!"` requires four bytes rather than three:

```text
[ H ][ I ][ ! ][ \0 ]
```

This illustrates a broader principle:

> Structured information requires both a representation and conventions for interpreting that representation.

---

## `main`

`main` is the entry point of a C program. When the executable starts, execution eventually enters `main`.

A program that doesn't accept command-line arguments can use:

```c
int main(void)
```

A program that accepts command-line arguments can use:

```c
int main(int argc, char *argv[])
```

In CS50, `string argv[]` may be used because `string` is provided by the CS50 library.

- `argc` = **argument count**
- `argv` = **argument vector** containing the arguments

For example:

```bash
./greet XYZ
```

conceptually produces:

```text
argc = 2

argv[0] = "./greet"
argv[1] = "XYZ"
```

Therefore:

```c
printf("hello, %s\n", argv[1]);
```

outputs:

```text
hello, XYZ
```

`argc` can be checked before accessing `argv` to ensure the expected number of arguments was provided.

### Exit Status

`main` can return an integer representing the program's exit status:

```c
return 0;
```

By convention:

```text
0       → success
nonzero → some kind of failure/error
```

In a shell, the previous program's exit status can be inspected with:

```bash
echo $?
```

---

## Key Takeaways

The important concepts from this lecture are not C syntax itself, but the machinery being exposed underneath higher-level languages:

```text
source code
    ↓
preprocessing
    ↓
compilation → assembly
    ↓
assembling → machine code
    ↓
linking
    ↓
executable
```

and:

```text
memory = bytes
    ↓
types determine how bytes are interpreted
    ↓
arrays organize values in contiguous memory
    ↓
characters have numeric representations
    ↓
strings are character arrays + a termination convention
```

The central idea is:

> **Data structures that appear abstract at the programming-language level ultimately have concrete representations in memory.**