---
layout: post
title: "Cadlib - API Design (Part 2)"
tag: "cadlib"
previous: /2026/09/01/cadlib-api-design-1
---

My last post left off with working with objects using smart pointers and tables like ranges. There are more areas of interest when working
with AutoCAD. This post will focus on AutoLisp, the primary scripting language of AutoCAD, and Result Buffers. We'll see more of the same theme
being expounded upon in addition to using compile time logic to help reduce error-prone code.

## Result Buffers and Result Codes

We'll talk about buffers first, since we need to understand this to understand certain parts of how AutoLisp communicates with AutoCAD. Result Buffers
are a linked list of data and metadata. The metadata identifies the type of data on the next node. Basically it's a linked list of result code -> result data 
-> result code -> result data. The "code" is just an ID number, and the data payload is contained in a large union. The payload union consists of a lot 
of plain data types and raw pointers to arrays.

Numerical designations, raw pointers, union of plain data types, this sounds like a potentially error-prone mess. In addition, we need to match the correct
data types used with the result code. Switching on integer types is already a bad practice because cases can be missed, either now or down the road. Also, we 
can't easily use the data later hoping to remember we are using it correctly. Remember, my goal is to make an interface that is <b>easy to use correctly.</b> 
The approach I chose to take required writing a lot of boilerplate code. But it was worth the effort in the end. 

First things first, the linked list. We can easily treat resbuf pointers as modern ranges by encapsulation, similarly to how it was described in the 
[previous post.]({{ page.previous | absolute_url }}) I chose to treat resbufs as doubly-linked lists. This way, I can modify from the front or back at any 
time without keeping track of any iterators and with little overhead. I ended up creating two classes, one for ownership and one that simply references a resbuf. 

```cpp
class owned_result {
  public:
  class iterator {
    public:
    using value_type = resbuf*;
    using defference_type = std::ptrdiff_t; // Required for C++ range algorithms.
    // Clipped
  };
  // Clipped
};

class weak_result {
  public:
  using iterator = owned_result::iterator; // No need to maintain a separate type for both.
  using value_type = resbuf*;
  // Clipped
};
```

You might think I created <b>weak_result</b> to reference <b>owned_result</b> similarly to std::weak_ptr and std::shared_ptr. I didn't quite get around to that 
level of sophistication, but it was good enough at the time.

Now for result codes. In order to make things easier for the user, I went with some good ol' template meta programming. First I defined scoped enums for the
various values a result code could be. 

```cpp
// Uses the result code values in adscodes.h
enum class res_code : short {
  // Clipped
};

// Values copied from acdb.h
enum class dxf : AcDb::DxfCode {
  // Clipped
};
```

Everything will start coming together soon.

## AutoLisp

Working with AutoLisp will rely heavily on what I explained earlier. But first comes more boilerplate. I created partially specialized templates 
to link all the correct types and union fields at compile time.

```cpp
template <res_code T>
struct lisp_type_t;

// An example specialization.
template <>
struct lisp_type_t<res_code::none> {
  static constexpr auto type = L"NONE";
};

template <res_code T>
struct lisp_getter_t;

// An example specialization.
template <>
struct lisp_getter_t<res_code::real> {
  static auto get(const resbuf* buf) { return buf->resval.rreal; }
};
```

There are several benefits to doing it this way. If we need to get the string representation of a result code, for logging errors, the compiler will 
make sure we cover every case in our switch statement. By avoiding a default case, the compiler will give us an error if we forgot to account for a result code. 
(Assuming all the appropriate warnings are enabled, which they were.) 

When we need to retrieve a Lisp symbol, we have another benefit. By passing the res_code as a template parameter, we can write our wrapper functions to give us the 
exact symbol type we need and where in the union we need to get it from. The compiler will generate the exact code we need because we gave it all the information 
upfront. We will also be able to catch more errors in our code at compile time because we used this approach.

```cpp
template<res_code T>
auto get_auto_lisp_symbol(const AcString& auto_lisp_symbol) {
  // Returns an owned_result.
  const auto result = get_raw_symbol(static_cast<std::remove_const_t<AcString>>(auto_lisp_symbol));

  if (!result) {
    // Custom exception type to facilitate unicode strings.
    throw lisp_exception{ std::format(L"Invalid auto lisp symbol: {}", auto_lisp_symbol.constPtr()) };
  }

  auto verify_result_code = [](const res_code type_got) -> bool {
    return type_got == T;
  };


  if (verify_result_code(res_code{ result->restype })) {
    return lisp_getter_t<T>::get(*result);
  }

  throw lisp_exception{
    std::format(L"Invalid symbol access: {} is not of type {}. Type {} found.\n", auto_lisp_symbol.constPtr(), 
      lisp_type_t<T>::type, type_code_to_string(res_code{result->restype}))
  };
}
```

I'll just finish up by saying that dealing with the union field types themselves was pretty straight forward. All I needed to do was make modern alternatives to any 
unsafe union field types and simply use them in the lisp_getter_t\<T\>::get() function. For example, instead of a 3D coordinate being a raw array of 3 doubles, I created 
a struct that holds 3 doubles and placed the array items in that.

How does all that look on the user side? Something like this:

```cpp
// Returns a std::wstring.
const auto lisp_symbol = get_auto_lisp_symbol<res_code::str>(L"my_string_variable");
```

And that just works, provided the expected type and the type of "my_string_variable" match.
