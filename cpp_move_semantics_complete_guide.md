# C++ Move Semantics: From First Principles

## Table of Contents

1. Why Move Semantics Exist
2. Lvalues and Rvalues
3. Lvalue References vs Rvalue References
4. Function Overloading with References
5. Building Move Semantics from Scratch
6. Copy Constructors
7. Move Constructors
8. Why `std::move` Exists
9. The Biggest Misconception About `std::move`
10. Resource Management
11. Shared Pointer Example
12. Unique Pointer Example
13. Constructor Conventions
14. Function Parameter Design
15. Language Support and Overload Selection
16. Temporaries
17. Return Values
18. Practical Rules
19. Common Mistakes
20. Mental Model Summary

---

# 1. Why Move Semantics Exist

Before C++11, expensive objects could only be copied.

Examples:

- `std::string`
- `std::vector`
- `std::shared_ptr`
- file handles
- sockets
- GPU buffers

Copying often means:

- allocating memory
- copying bytes
- updating bookkeeping

This can be expensive.

Move semantics were introduced to transfer ownership instead of duplicating resources.

---

# 2. Lvalues and Rvalues

## Lvalue

Has a name and can be referred to later.

```cpp
int x = 5;
```

`x` is an lvalue.

---

## Rvalue

Temporary value.

```cpp
5
```

```cpp
Foo()
```

```cpp
makeFoo()
```

Rvalues are generally about to die.

---

# 3. Lvalue References vs Rvalue References

```cpp
int&
```

Lvalue reference.

```cpp
int&&
```

Rvalue reference.

Think of them as two colors of references.

Both are references.

Neither stores data.

Neither automatically copies.

Neither automatically moves.

The only difference is that they participate differently in overload resolution.

---

# 4. Function Overloading with References

```cpp
void f(const String&);
void f(String&&);
```

When calling:

```cpp
String s;

f(s);
```

Compiler selects:

```cpp
f(const String&);
```

because `s` is an lvalue.

---

```cpp
f(String{});
```

Compiler selects:

```cpp
f(String&&);
```

because the argument is temporary.

---

# 5. Building Move Semantics from Scratch

Imagine:

```cpp
class String {
public:
    char* data;
};
```

Copying duplicates ownership.

Moving transfers ownership.

---

# 6. Copy Constructors

```cpp
String(const String& other);
```

Purpose:

Create a new object while leaving the source untouched.

Example:

```cpp
String a("hello");
String b(a);
```

Memory:

```text
a --> [hello]

b --> [hello]
```

Two buffers.

Two owners.

---

# 7. Move Constructors

```cpp
String(String&& other)
{
    data = other.data;
    other.data = nullptr;
}
```

Purpose:

Steal ownership.

Example:

```cpp
String a("hello");
String b((String&&)a);
```

Before:

```text
a --> [hello]
```

After:

```text
a --> nullptr

b --> [hello]
```

No character copy.

Ownership transferred.

---

# 8. Why std::move Exists

Without helper:

```cpp
(String&&)a
```

This is ugly.

So we write:

```cpp
std::move(a)
```

Conceptually:

```cpp
template<typename T>
T&& move(T& obj)
{
    return static_cast<T&&>(obj);
}
```

---

# 9. Biggest Misconception

Many people believe:

```cpp
std::move(x);
```

moves an object.

Wrong.

It moves nothing.

It merely casts.

This:

```cpp
std::move(x)
```

is approximately:

```cpp
(String&&)x
```

The actual move happens later when an overload is selected.

---

# 10. Resource Management

Move semantics matter most when resources exist.

Examples:

- heap memory
- file descriptors
- sockets
- mutex ownership
- reference counts

Moving avoids expensive duplication.

---

# 11. Shared Pointer Example

Simplified implementation:

```cpp
class MySharedPtr {
public:
    int* ptr;
    int* count;
};
```

Initial state:

```text
ptr ----> Object

count = 1
```

## Copy

```cpp
MySharedPtr b(a);
```

Copy constructor:

```cpp
++(*count);
```

Result:

```text
count: 1 -> 2
```

Two owners.

---

### Why Is This Expensive?

Real shared pointers use:

```cpp
std::atomic<int>
```

Atomic increments require:

- cache synchronization
- memory barriers
- CPU coordination

Millions of such operations become measurable.

---

## Move

```cpp
MySharedPtr(MySharedPtr&& other)
{
    ptr = other.ptr;
    count = other.count;

    other.ptr = nullptr;
    other.count = nullptr;
}
```

Reference count unchanged.

No increment.

No decrement.

Ownership transferred.

---

# 12. Unique Pointer Example

Unique ownership.

Copying forbidden.

```cpp
std::unique_ptr<int> a;
std::unique_ptr<int> b(a);
```

Compiler error.

Copy constructor deleted.

---

Move allowed.

```cpp
std::unique_ptr<int> b((std::unique_ptr<int>&&)a);
```

Before:

```text
a --> Object
```

After:

```text
a --> nullptr

b --> Object
```

Exactly one owner remains.

---

# 13. Constructor Conventions

Typical copy constructor:

```cpp
String(const String&);
```

Reason:

Copying should not modify source.

---

Typical move constructor:

```cpp
String(String&&);
```

Reason:

Moving often modifies source.

Example:

```cpp
other.data = nullptr;
```

Cannot do that with const.

---

# 14. Function Parameter Design

Three common styles.

## Const Reference

```cpp
void f(const String&);
```

Read only.

No ownership transfer.

---

## Rvalue Reference

```cpp
void f(String&&);
```

Caller intends transfer.

Function may consume object.

---

## Pass By Value

```cpp
void f(String);
```

Caller decides:

```cpp
f(x);            // copy
f(std::move(x)); // move
```

Useful when function stores the object.

---

# 15. Language Support

Compiler automatically prefers:

- lvalue overload for named objects
- rvalue overload for temporaries

This is what makes move semantics practical.

---

# 16. Temporaries

Temporary:

```cpp
Foo()
```

Temporary:

```cpp
makeFoo()
```

Temporary:

```cpp
Foo{}
```

These are candidates for moving.

---

Named variable:

```cpp
Foo foo;
```

Not temporary.

Compiler assumes it may be reused.

Uses lvalue overload.

---

# 17. Return Values

Example:

```cpp
Foo makeFoo()
{
    Foo local;
    return local;
}
```

Compiler knows:

```text
local is about to die
```

Move becomes attractive.

Modern compilers frequently apply copy elision as well.

---

# 18. Practical Rules

### Rule 1

Copy means:

```text
Keep original ownership.
Create another owner.
```

### Rule 2

Move means:

```text
Transfer ownership.
```

### Rule 3

`std::move` does not move.

### Rule 4

Move constructors perform moves.

### Rule 5

Rvalue references mainly exist to enable efficient ownership transfer.

---

# 19. Common Mistakes

## Mistake 1

Thinking:

```cpp
std::move(x);
```

moves data.

It does not.

---

## Mistake 2

Using moved-from objects.

After:

```cpp
std::move(x)
```

and a subsequent move,

the state of `x` should generally not be relied upon.

---

## Mistake 3

Believing `&&` automatically moves.

It does not.

It is still a reference.

---

## Mistake 4

Using `const T&&` for move semantics.

Usually wrong.

Move operations often need mutation.

---

# 20. Final Mental Model

```text
T&
= normal reference

const T&
= read-only reference

T&&
= permission to steal resources
```

```text
Copy Constructor
= duplicate ownership

Move Constructor
= transfer ownership
```

```text
std::move(x)
= tell compiler to treat x as disposable
```

```text
Actual move
= code inside move constructor
```

The entire move semantics system is fundamentally:

1. Two kinds of references (`&` and `&&`)
2. Function overloading
3. Ownership transfer conventions
4. Compiler preference for temporaries
