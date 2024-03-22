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

| Document Number: | d2927r2            |
| ---------------- | ------------------ |
| Date:            | 2024-03-22         |
| Target:          | LEWG               |
| Revises:         | p2927r1            |
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

## Revision history

**r0** - restart the proposal in a simplified form

**r1** - implement "strict" behavior (`exception_ptr_cast<logic_error>`, as opposed to also allowing cv-ref qualified types, as in `exception_ptr_cast<const logic_error&>`, for example)

**r2** - rename to `exception_ptr_cast`, add motivation section, add feature test macro.

## Proposal at a glance

We introduce a single function `exception_ptr_cast` that takes `std::exception_ptr` as an argument `e` and returns a pointer to an object referred to by `e`. 

```c++
template <typename T>
const T* 
exception_ptr_cast(const exception_ptr& e) noexcept;
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
    if (auto* x = std::exception_ptr_cast<Baz>(exp))
        printf("got '%s' i: %d j: %d\n", typeid(*x).name(), x->i, x->j); 
    if (auto* x = std::exception_ptr_cast<Bar>(exp))
        printf("got '%s' i: %d j: %d\n", typeid(*x).name(), x->i, x->j);
    if (auto* x = std::exception_ptr_cast<Foo>(exp))
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

## Motivation

In asynchronous programming, errors frequently
travel packaged in an `exception_ptr`. For example, see: `std::future`, `boost::future`, nvidia's `exec::task`, `Folly::Future`, `libunifex::task`, `ppl::task`, etc.

However, exception_ptr itself is an opaque type and
to get to the exact error stored requires
rethrow and catch.

The facility offered here allows getting access to
the stored error that is significantly (100x) faster.
While theoretically, some exception patterns could
be optimized by the compilers (see https://wg21.link/p1676),
no major compiler has implemented any of these
optimizations since C++ existed.

### Before 

Consider a read_harder routine that wraps an async_read to retry on certain failures upto a limit.

```c++
task<int> read_harder(my::channel<int>& q) {
    for (int attempt = 0;; ++attempt) {
        try {
            co_return co_await q.async_read();
        }
        catch (const std::system_error& e) {
            if (e.code() == errc::device_or_resource_busy && attempt < 10)
              continue;

            throw;
        }
    }
}
```

With the offered facility we can implement the same logic without
having to rethrow and catch.

### After

```c++
task<int> read_harder(my::channel<int>& q) {
    for (int attempt = 0;; ++attempt) {
        auto result = co_await q.async_read().as_expected();
        if (result)
           co_return *result;

        if (auto* e = std::exception_ptr_cast<system_error>(result.error());
           e->code() == errc::device_or_resource_busy && attempt < 10)
           continue;

        std::rethrow_exception(result.error());
    }
}
```
Similar code size, but, significantly faster. This code assumes the existence of an adapter algorithm that converts `sender<T>` to `sender<expected<T, exception_ptr>>`


## Simplification post Kona 2023

Previous revision tentatively proposed a complicated signature imitating the syntax of a catch-parameter, as in `old::exception_ptr_cast<const std::exception&>(p)`.
In Kona, we were convinced to simplify the signature to assume catch-by-const-reference no matter what: `std::exception_ptr_cast<std::exception>(p)`.

- With the old design, there were two ways to simulate catching by const reference: `old::exception_ptr_cast<const std::exception&>(p)`
and `old::exception_ptr_cast<std::exception>(p)`. The new syntax removes that needless variability of style.
- Users of the old syntax might think they had to write `old::exception_ptr_cast<const std::exception&>(p)` in order to catch by reference;
that's unnecessarily verbose and hard to read. The new syntax is short and readable.
- The old syntax permitted `old::exception_ptr_cast<std::exception&>(p)` to catch by non-const reference; this is sometimes legitimate but usually it's just an unnecessary violation of const-correctness. Users might omit the `const` out of inattention or laziness. The new syntax always returns `const*`, privileging the common and const-correct case.
- Users who want to modify the in-flight exception object can still explicitly write `const_cast<std::exception*>(std::exception_ptr_cast<std::exception>(p))`. The `const_cast` is visible and greppable; this is a good thing.
- The old syntax permitted nonsensical scenarios such as `old::exception_ptr_cast<int(&)()>(p)`. Such a catch handler is physically impossible to hit, because you can't throw a function; functions are not objects. The new syntax admits only `old::exception_ptr_cast<int()>(p)`, which is ill-formed by our new Mandates element because `int()` is not an object type. Similarly, we disallow `old::exception_ptr_cast<int[5]>(p)` via the Mandates element.

P2927R0 proposed that `exception_ptr_cast` should be able to catch pointer types, just like an ordinary catch clause. That is, not only were you allowed to inspect a thrown `Derived` object with `old::exception_ptr_cast<const Base&>` (which would return a possibly null `const Base*`), you were also allowed to inspect a thrown `Derived*` object with `old::exception_ptr_cast<const Base*>` (which would return a possibly null `const Base**`). This turned out to be unimplementable. When a catch-handler catches `Derived*` as `Base*`, it may need to adjust the pointer for multiple and/or virtual base classes. The pointer caught by the core language, then, is a temporary. We can't return a `const Base**` pointing to that temporary adjusted pointer, because there's nowhere for the temporary adjusted pointer to live after the call to `exception_ptr_cast` has returned.

In other words, the new design has a strict invariant: the pointer returned from `exception_ptr_cast` always points to the in-flight exception object itself. It never points to any other object, such as a temporary or global. Thus, we must disallow:

```c++
    using IntPtr = int*;
    std::nullptr_t np;
    auto p = std::make_exception_ptr(np);
      // The in-flight exception object is of type std::nullptr_t
    const IntPtr *ex = old::exception_ptr_cast<IntPtr>(p);
      // ex cannot possibly point to the in-flight exception object, because the in-flight object is not an IntPtr!
    try {
        std::rethrow_exception(p);
    } catch (const IntPtr& ex) {
        // OK, ex refers to a temporary that lives only as long as this catch block
    }
```

Our solution is simply to extend our Mandates element to also forbid `std::exception_ptr_cast` with a template argument of pointer or pointer-to-member type; these are the only two kinds of types where a core-language catch-handler parameter would sometimes bind to a temporary, so these are the only kinds of types we need to forbid. Later, we found that [[ScyllaDB](https://github.com/scylladb/scylladb/blob/946d281/utils/exceptions.hh#L128-L151)] had independently implemented the same solution (i.e. explicitly forbid pointer types) in 2022.

Throwing pointers is rare — probably unheard of in real code. This does prevent users from using `std::exception_ptr_cast<const char*>(p)` to inspect the results of `throw "foo"`, which comes up sometimes in example code; but it shouldn't happen in real code.

<!--
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

In this paper we only offer a `exception_ptr_cast` API that offer an ability to inspect, 
but not modify stored exception. If in the future, we will discover an important use
case that requires changing the stored exception a different API with lengthier
and less convenient name, say `try_cast_mutable` or something along those lines can be added.

The desire is to offer the simplest name for the most common use case that is a good
default for the majority of the users.

Similarly, we chose to specify the desired type in the simplest possible form, i.e:

```sql
auto* x = exception_ptr_cast<const int*>(eptr); // no
auto* x = exception_ptr_cast<int*>(eptr);       // no
auto* x = exception_ptr_cast<int>(eptr);        // yes
```

Thus, the thought process above led to the following API shape:

```c++
template <typename T>
const decay_t<T>* 
exception_ptr_cast(const exception_ptr& e) noexcept;
```

**Q**: Could we reuse `any_cast` name?

**A**: No. It encodes `any` in its name strongly hinting it is meant
to be used with std::any.

**Q**: Maybe `get_if` would work?

**A**: Possibly, but, it has slightly different semantics.
Unlike `any_cast`, `dynamic_cast` and offered here `exception_ptr_cast` have
`cast` in its name, implying that it does not have to be exact match,
whereas `get_if` expects the type to be exact match from the list of variant
alternatives.

It is possible that in the future, we will have a unifying
facility (pattern matching, for example) that would allow uniform access across
variant, any, exception_ptr and other types. Finding such facility is out of
scope of this paper.

**Q**: Why `const std::decay_t<T>*` in the return value? Could it be just T*?

**A**: The intent here is to make sure that people who habitually write `catch (const E& e) { ... }` can continue doing it with `exception_ptr_cast<const E&>`, as opposed to getting a compilation error. This is an error from the category:
The compiler/library knows what you mean, but, will force you to write exactly as it wants.
-->

## Pattern matching

We expect that `exception_ptr_cast` will be integrated in the [pattern matching facility](https://wg21.link/p1371) and
will allow inspection of `exception_ptr` as follows:

```c++
inspect (eptr) {
   <logic_error> e => { ... }
   <exception> e => { ... }
   nullptr => { puts("no exception"); }
   __ => { puts("some other exception"); }
}
```

## Other names considered

* `try_cast<E>`
* `cast_exception_ptr<E>`
* `cast_exception<E>`
* `try_cast_exception_ptr<E>`
* `try_cast_exception<E>`
* `catch_as<E>`
* `exception_cast<E>`

Based on the recent discussion on LEWG mattermost on March 22, 2024, the two top names favored were:

* `exception_ptr_cast`
* `exception_cast`

This revision optimistically chosen the first alternative,
as it seemed to get more likes, but, the authors will gladly rename the facility to
any LEWG approved name.

## Implementation

GCC, MSVC implementation is possible using available (but undocumented) APIs https://godbolt.org/z/ErePMf66h. Implementability was also confirmed by MSVC and libstdc++ implementors.

A similar facility is available in Folly and supports Windows, libstdc++, and libc++ on linux/apple/freebsd.

https://github.com/facebook/folly/blob/v2023.06.26.00/folly/lang/Exception.h
https://github.com/facebook/folly/blob/v2023.06.26.00/folly/lang/Exception.cpp

Implementation there under the names:
folly::exception_ptr_get_object
folly::exception_ptr_get_type

Extra constraint imposed by MSVC ABI: it doesn't have information stored to do a full dynamic_cast. It can only recover types for which a catch block could potentially match.
This does not conflict with the `exception_ptr_cast` facility offered in this paper.

Arthur has implemented P2927R1 `std::exception_ptr_cast` in his fork of libc++; see [[libc++](https://github.com/Quuxplusone/llvm-project/commit/6e20a0b9d5a2280bfab8ab42bee841cfbcc4a8bd)] and [[Godbolt](https://godbolt.org/z/3Y8Gzfr7r)].

## Usage experience

A version of exception_ptr inspection facilities is deployed widely in Meta as
a part of Folly's future continuation matching.

ScyllaDB implements almost exactly the wording of this proposal, under the name `try_catch<E>(p)`; see [[ScyllaDB](https://github.com/scylladb/scylladb/blob/946d281/utils/exceptions.hh#L128-L151)]. The only difference is that they return `E*` instead of `const E*`. We hear from them that they don't actually use the mutability for anything; and even if they did, they could add `const_cast` as mentioned above.

## Proposed wording (relative to n4950)

In section [exception.syn] add definition for `exception_ptr_cast`
as follows:

<div style="margin-left: 30px;">
<code>
exception_ptr current_exception() noexcept;<br>
[[noreturn]] void rethrow_exception(exception_ptr p);<br>
<ins>
template &lt;class E&gt;<br>
&nbsp;&nbsp;const E* exception_ptr_cast(const exception_ptr& p) noexcept;
</ins><br>
template &lt;class T&gt; [[noreturn]] void throw_with_nested(T&& t);
</code>
</div>
<p></p>
Modify paragraph 7 of section Exception propagation [propagation] as follows:
<p></p>
<div style="margin-left: 30px;">
For purposes of determining the presence of a data race, operations on <code>exception_ptr</code> objects shall access and modify only the <code>exception_ptr</code> objects themselves and not the exceptions they refer to. Use of <code>rethrow_exception</code> <ins>or <code>exception_ptr_cast</code></ins>
on <code>exception_ptr</code> objects that refer to the same exception object shall not introduce a data race.
</div>
<br>
Add the following paragraph immediately after paragraph 8 of section Exception propagation [propagation]:
<p></p>
<div style="margin-left: 30px;">
<ins>
template &lt;class E&gt;<br>
&nbsp;&nbsp;const E* exception_ptr_cast(const exception_ptr& p) noexcept;
<br>
<br><i>Mandates:</i> <code>E</code> is a <i>cv</i>-unqualified complete object type. <code>E</code> is not an array type. <code>E</code> is not a pointer or pointer-to-member type. <i>[Note: When <code>E</code> is a pointer or pointer-to-member type, a handler of type <code>const E&amp;</code> can match without binding to the exception object itself. —end note]</i>
<br>
<br><i>Returns: </i>
A pointer to the exception object referred to by <code>p</code>,
if <code>p</code> is not null and a handler of type <code>const E&amp;</code> would be a
match [except.handle] for that exception object.
Otherwise, <code>nullptr</code>.
</ins><br>
<div style="margin-left: 20px;">

</div>
</div>
<br>
<b>[version.syn]</b>
<br><br>
Add:
<br>

<div style="margin-left: 20px;">
<ins>#define __cpp_lib_exception_ptr_cast   202403L // also in &lt;exception&gt;</ins>
</div>

<!--
## Kona 2023 EWG Feedback

Suggestions mentioned but not polled:

* Add a note that to modify the stored exception, one can use `const_cast`. For example: `const_cast<int*>(exception_ptr_cast<int>(eptr))`. (This was mentioned as an alternative to having dedicated `mutable_try_cast`).
* Make `exception_ptr_cast<int[5]>` or `exception_ptr_cast<void(int)>` ill-formed
* Make `exception_ptr_cast<int&>` ill-formed
* Alternatively, give mutable and non-mutable versions depending on the type:
    * `exception_ptr_cast<int>(eptr) -> const int*`
    * `exception_ptr_cast<int&>(eptr) -> int*`
-->

## Acknowledgments

Many thanks to those who provided valuable feedback, among them:
Aaryaman Sagar,
Barry Revzin,
Gabriel Dos Reis, Jan Wilmans, Joshua Berne,
Lee Howes, Lewis Baker, Michael Park, Peter Dimov, Ville Voutilainen, Yedidya Feldblum.

## References

https://godbolt.org/z/E8n69xKjs (gcc and msvc implementation)

https://wg21.link/p0933 Runtime introspection of exception_ptr

https://wg21.link/p1066 How to catch an exception_ptr without even try-ing

https://wg21.link/p1371 Pattern Matching

https://github.com/scylladb/scylladb/blob/946d281/utils/exceptions.hh#L128-L151 ScyllaDb

An implementation of try cast and libc++ and matching godbolt:

https://github.com/Quuxplusone/llvm-project/commit/6e20a0b9d5a2280bfab8ab42bee841cfbcc4a8bd

https://godbolt.org/z/3Y8Gzfr7r

https://wg21.link/p1676 Optimizing exceptions


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

const_cast<int*>(exception_ptr_cast<int>(eptr))


-->
