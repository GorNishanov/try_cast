<style type="text/css">
p {text-align:justify}
li {text-align:justify}
blockquote.note
{
background-color:#E0E0E0;
padding-left: 15px;
padding-right: 15px;
padding-top: 1px;
padding-bottom: 1px;
}
code
{
color:#000000;
}
ins {background-color:#A0FFA0}
del {background-color:#FFA0A0}
table {border-collapse: collapse;}
table, th, td {
border: 1px solid black;
border-collapse: collapse;
}
</style>

| Document Number: | d2927r1            |
| ---------------- | ------------------ |
| Date:            | 2024-02-14         |
| Target:          | LEWG               |
| Revises:         | p2927r0            |
| Reply to:        | Arthur O'Dwyer (arthur.j.odwyer@gmail.com), Gor Nishanov (gorn@microsoft.com) |


# Inspecting exception_ptr

## Abstract 

Provide facility to observe exceptions stored in `std::exception_ptr` without throwing or catching exceptions.

## Introduction

This is a followup to two previous papers in this area:

Date | Link | Title
-----|------|------
Feb 7, 2018 | https://wg21.link/p0933 | Runtime introspection of exception_ptr
Oct 6, 2018 | https://wg21.link/p1066 | How to catch an exception_ptr without even try-ing

These papers received positive feedback. In 2018 Rapperswil meeting, EWG expressed strong desire in having such facility. This was reaffirmed 
in 2023 Kona meeting.

This paper brings back exception_ptr inspection facility in a simplified form addressing
the earlier feedback.

## Proposal at a glance

We introduce a single function `try_cast` that takes `std::exception_ptr` as an argument `e` and returns a pointer to an object referred to by `e`. 

```c++
template <typename T>
const T* 
try_cast(const exception_ptr& e) noexcept;
```

**Example:**

Given the following error classes:
```c++
struct Foo {
    virtual ~Foo() {}
    int i = 1;
};
struct Bar : Foo, std::logic_error {
    Bar() : std::logic_error("This is Bar exception") {}
    int j = 2;
};
struct Baz : Bar {};
```
The execution of the following program
```c++
int main() {
    const auto exp = std::make_exception_ptr(Baz());
    if (auto* x = std::try_cast<Baz>(exp))
        printf("got '%s' i: %d j: %d\n", typeid(*x).name(), x->i, x->j); 
    if (auto* x = std::try_cast<Bar>(exp))
        printf("got '%s' i: %d j: %d\n", typeid(*x).name(), x->i, x->j);
    if (auto* x = std::try_cast<Foo>(exp))
        printf("got '%s' i: %d\n", typeid(*x).name(), x->i);
}
```
results in this output:

```
got '3Baz' what:'This is Bar exception' i: 1 j: 2
got '3Baz' what:'This is Bar exception' i: 1 j: 2
got '3Baz' i: 1
```

See implementation for GCC and MSVC using available (but undocumented) APIs https://godbolt.org/z/E8n69xKjs.

## Post Kona 2023 update

The following questions were raised during LEWG review.

**Should `try_cast` take exception_ptr by pointer or reference?**

LEWG affirmed that it should be taken by reference

**Should `try_cast` be "permissive" or "strict"?**

1. Permissive (as proposed in R0 of this paper):
```c++
try_cast<int>(e) -> const int*
try_cast<int&>(e) -> const int*
try_cast<const int&>(e) -> const int*
try_cast<int[5]>(e) -> int* const*
try_cast<int*>(e) -> int* const*
try_cast<int()>(e) -> int(* const*)()
try_cast<int(*)()>(e) -> int(* const*)()
```

2. Strict alternative (as was suggested in Kona 2023):
```c++
try_cast<int>(e) -> const int*
try_cast<int*>(e) -> int* const*
try_cast<int(*)()>(e) -> int(* const*)()
try_cast<int&>(e)        // ill-formed
try_cast<const int&>(e)  // ill-formed
try_cast<int[5]>(e)      // ill-formed
try_cast<int()>(e)       // ill-formed
```

In Kona 2023 LEWG selected strict option.

## Discussion

Let's look at existing facilities in the library that serve somewhat similar purpose:

**any_cast:**

```c++
template<class T> T any_cast(const any& operand);
template<class T> T any_cast(any& operand);
template<class T> T any_cast(any&& operand);

template<class T> const T* any_cast(const any* operand) noexcept;
template<class T> T* any_cast(any* operand) noexcept;
```

Last two overloads perform inspection of a stored value and give a pointer to it
if the typeid matches, otherwise return `nullptr`.

The first three, throw `bad_any_cast` exception if the stored value does not match the requested type.

**get/get_if:**

```c++
// get forms that takes a reference and throw if the type is not active alternative
template<class T, class... Types> T* get_if(variant<Types...>* pv) noexcept;
template<class T, class... Types> const T* get_if(const variant<Types...>* pv) noexcept;
```

Similarly to `any_cast`, get and get_if offer throwing versions that take a reference
to a variant and a non-throwing that take a variant by pointer and return a pointer
to the value of the requested type or `nullptr` if it is not an active alternative.

**exception_ptr:**

**Q**: Should we follow the the examples above and have two flavors of exception_ptr inspection,
one that throws if the requested type `E` is not stored and one that does not.

**A**: No. The goal of this facility is to provide a way to inspect a stored exception value
without relying on throwing exception.

**Q**: Non-throwing versions of `dynamic_cast`, `any_cast` and `get_if` all take a pointer to a value to be inspected,
should this API take the pointer as well?

**A**: `dynamic_cast` and `any_cast` relies on a distinction on how the value is passed
(by pointer or by reference) to select what semantic to provide. In our case,
we always perform a conditional access, thus, requiring a pointer would be a needless syntactic noise.

**Q**: The `variant` and `any` inspection APIs offers a version that allows both const and
non const access to the stored value. Should we do the same for `exception_ptr`?

**A**: No. N4928/[propagation]/7 states that:

> For purposes of determining the presence of a data race, operations on exception_ptr objects shall access and modify only the exception_ptr objects themselves and not the exceptions they refer to.

Offering the API flavor that allows mutation of the stored exception
has potential of injecting a data race. Cpp core guidelines recommend 
catching exception by const reference.

In this paper we only offer a `try_cast` API that offer an ability to inspect, 
but not modify stored exception. If in the future, we will discover an important use
case that requires changing the stored exception a different API with lengthier
and less convenient name, say `try_cast_mutable` or something along those lines can be added.

The desire is to offer the simplest name for the most common use case that is a good
default for the majority of the users.

Similarly, we chose to specify the desired type in the simplest possible form, i.e:

```sql
auto* x = try_cast<const int*>(eptr); // no
auto* x = try_cast<int*>(eptr);       // no
auto* x = try_cast<int>(eptr);        // yes
```

Thus, the thought process above led to the following API shape:

```c++
template <typename T>
const decay_t<T>* 
try_cast(const exception_ptr& e) noexcept;
```

**Q**: Could we reuse `any_cast` name?

**A**: No. It encodes `any` in its name strongly hinting it is meant
to be used with std::any.

**Q**: Maybe `get_if` would work?

**A**: Possibly, but, it has slightly different semantics.
Unlike `any_cast`, `dynamic_cast` and offered here `try_cast` have
`cast` in its name, implying that it does not have to be exact match,
whereas `get_if` expects the type to be exact match from the list of variant
alternatives.

It is possible that in the future, we will have a unifying
facility (pattern matching, for example) that would allow uniform access across
variant, any, exception_ptr and other types. Finding such facility is out of
scope of this paper.

**Q**: Why `const std::decay_t<T>*` in the return value? Could it be just T*?

**A**: The intent here is to make sure that people who habitually write `catch (const E& e) { ... }` can continue doing it with `try_cast<const E&>`, as opposed to getting a compilation error. This is an error from the category:
The compiler/library knows what you mean, but, will force you to write exactly as it wants.

### Section added after Kona 2023 EWG session

1. Permissive (as proposed in the paper up to this point):
```c++
try_cast<int>(e) -> const int*
try_cast<int&>(e) -> const int*
try_cast<const int&>(e) -> const int*
try_cast<int[5]>(e) -> int* const*
try_cast<int*>(e) -> int* const*
try_cast<int()>(e) -> int(* const*)()
try_cast<int(*)()>(e) -> int(* const*)()
```

2. Strict alternative (suggested by some in Kona 2023):
```c++
try_cast<int>(e) -> const int*
try_cast<int*>(e) -> int* const*
try_cast<int(*)()>(e) -> int(* const*)()
try_cast<int&>(e)        // ill-formed
try_cast<const int&>(e)  // ill-formed
try_cast<int[5]>(e)      // ill-formed
try_cast<int()>(e)       // ill-formed
```

```c++
template<class E>
  const E* try_cast(const exception_ptr& p);
```

Mandates: E is a complete object type. E is not an array type. E is not a pointer to an incomplete type.<br>
Returns: A pointer to the exception object referred to by p, if p is not null and a handler of type const E& would be a match [except.handle] for that exception object. Otherwise, `nullptr`.


## Other names considered but rejected

* `cast_exception_ptr<E>`
* `cast_exception<E>`
* `try_cast_exception_ptr<E>`
* `try_cast_exception<E>`
* `catch_as<E>`

## Pattern matching

We expect that `try_cast` will be integrated in the [pattern matching facility](https://wg21.link/p1371) and
will allow inspection of `exception_ptr` as follows:

```c++
inspect (eptr) {
   <logic_error> e => { ... }
   <exception> e => { ... }
   nullptr => { puts("no exception"); }
   __ => { puts("some other exception"); }
} 
```

## Implementation

GCC, MSVC implementation is possible using available (but undocumented) APIs https://godbolt.org/z/ErePMf66h. Implementability was also confirmed by MSVC and libstdc++ implementors.

A similar facility is available in Folly and supports Windows, libstdc++, and libc++ on linux/apple/freebsd.
 
https://github.com/facebook/folly/blob/v2023.06.26.00/folly/lang/Exception.h
https://github.com/facebook/folly/blob/v2023.06.26.00/folly/lang/Exception.cpp
 
Implementation there under the names:
folly::exception_ptr_get_object
folly::exception_ptr_get_type

Extra constraint imposed by MSVC ABI: it doesn't have information stored to do a full dynamic_cast. It can only recover types for which a catch block could potentially match.
This does not conflict with the `try_cast` facility offered in this paper.

## Usage experience

A version of exception_ptr inspection facilities is deployed widely in Meta as
a part of Folly's future continuation matching.

## Proposed wording

## Proposed wording (relative to n4950)

In section [exception.syn] add definition for `try_cast`
as follows:

<div style="margin-left: 30px;">
<code>
exception_ptr current_exception() noexcept;<br>
[[noreturn]] void rethrow_exception(exception_ptr p);<br>
<ins>
template &lt;class T&gt;<br>
&nbsp;&nbsp;add_pointer_t&lt;const decay_t&lt;T&gt;&gt;<br>
&nbsp;&nbsp;try_cast(const exception_ptr& p) noexcept;
</ins><br>
template &lt;class T&gt; [[noreturn]] void throw_with_nested(T&& t);
</code>
</div>
<p></p>
Modify paragraph 7 of section Exception propagation [propagation] as follows:
<p></p>
<div style="margin-left: 30px;">
For purposes of determining the presence of a data race, operations on <code>exception_ptr</code> objects shall access and modify only the <code>exception_ptr</code> objects themselves and not the exceptions they refer to. Use of <code>rethrow_exception</code> <ins>or <code>try_cast</code></ins>
on <code>exception_ptr</code> objects that refer to the same exception object shall not introduce a data race.
</div>
<br>
Add the following paragraph immediately after paragraph 8 of section Exception propagation [propagation]:
<p></p>
<div style="margin-left: 30px;">
<ins>
template &lt;class T&gt;<br>
&nbsp;&nbsp;add_pointer_t&lt;const decay_t&lt;T&gt;&gt;<br> 
&nbsp;&nbsp;try_cast(const exception_ptr& p) noexcept;
<br>
<br><i>Mandates: </i> T is a complete type, or a pointer or lvalue reference to a complete type, or a pointer to <i>cv</i> <code>void</code>.<br>
<br><i>Returns: </i>
A pointer to the value stored in the exception_ptr,
if <code>p != nullptr</code> and a handler <code>catch(decay_t&lt;T&gt;&)</code> would be a
match [except.handle] for the exception object stored in p.
Otherwise, returns <code>nullptr</code>.
</ins><br>
<div style="margin-left: 20px;">

</div>
</div>


## Kona 2023 EWG Feedback

Suggestions mentioned but not polled:

* Add a note that to modify the stored exception, one can use `const_cast`. For example: `const_cast<int*>(try_cast<int>(eptr))`. (This was mentioned as an alternative to having dedicated `mutable_try_cast`).
* Make `try_cast<int[5]>` or `try_cast<void(int)>` ill-formed
* Make `try_cast<int&>` ill-formed
* Alternatively, give mutable and non-mutable versions depending on the type:
    * `try_cast<int>(eptr) -> const int*`
    * `try_cast<int&>(eptr) -> int*`


## Acknowledgments

Many thanks to those who provided valuable feedback, among them:
Aaryaman Sagar,
Arthur O'Dwyer, Barry Revzin,
Gabriel Dos Reis, Jan Wilmans, Joshua Berne, 
Lee Howes, Lewis Baker, Michael Park, Peter Dimov, Ville Voutilainen, Yedidya Feldblum.

## References

https://godbolt.org/z/E8n69xKjs (gcc and msvc implementation)

https://wg21.link/p0933 Runtime introspection of exception_ptr
 
https://wg21.link/p1066 How to catch an exception_ptr without even try-ing

https://wg21.link/p1371 Pattern Matching

<!--
get_if

template<size_t I, class... Types>
constexpr add_pointer_t<const variant_alternative_t<I, variant<Types...>>>
  get_if(const variant<Types...>* v) noexcept;
Mandates: I < sizeof...(Types).
Returns: A pointer to the value stored in the variant, if v != nullptr and v->index() == I.
Otherwise, returns nullptr.

any_cast
Returns: If operand != nullptr && operand->type() == typeid(T), a pointer to the object con- tained by operand; otherwise, nullptr.

14.4 Handling an exception [except.handle]
1 The exception-declaration in a handler describes the type(s) of exceptions that can cause that handler to be entered. The exception-declaration shall not denote an incomplete type, an abstract class type, or an rvalue reference type. The exception-declaration shall not denote a pointer or reference to an incomplete type, other than “pointer to cv void”.
2 A handler of type “array of T” or function type T is adjusted to be of type “pointer to T”.
§ 14.4 455
(3.1)
(3.2) (3.3)
(3.3.1)
(3.3.2) (3.3.3) (3.4)
— The handler is of type cv T or cv T& and E and T are the same type (ignoring the top-level cv-qualifiers), or
— the handler is of type cv T or cv T& and T is an unambiguous public base class of E, or
— the handler is of type cv T or const T& where T is a pointer or pointer-to-member type and E is a
pointer or pointer-to-member type that can be converted to T by one or more of
— a standard pointer conversion (7.3.12) not involving conversions to pointers to private or protected
or ambiguous classes
— a function pointer conversion (7.3.14)
— a qualification conversion (7.3.6), or
— the handler is of type cv T or const T& where T is a pointer or pointer-to-member type and E is std::nullptr_t.






f(T, eptr) -> T*

```c++
f<T>(eptr) -> T*

f<const T&>(eptr)

f<int>(eptr) -> const int*


f<const int&>(eptr) -> const int& *


catch (const int&)
f<const int>
f<int>
```


mutable_try_cast - unnecessary

const_cast<int*>(try_cast<int>(eptr))


-->
