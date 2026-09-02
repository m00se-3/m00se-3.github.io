---
layout: post
title: Cadlib - API Design (Part 1)
tag: "cadlib"
---

When designing an API for cadlib, recall what I said my goals were in the introduction post. The API should be "is **easy to use correctly**, no swapping parameters, no
raw strings when not needed, no switching over integers." 

To accomplish this goal, I spent a lot of time reading and re-reading the documentation, as well as studying the examples. In so doing, I discovered what I was 
in for. Everything I had learned about Modern C++ and type safety was put to the test. 

This is part 1 of a small series of posts about API design. There is way too much to talk about to put it all in one place.

## Objects, Tables, and Memory

In a nutshell, a Civil3D drawing is organized like a database. It contains tables that hold records of various object types, each record could hold attributes, blocks
text fields, geometry, etc. All this data in encapsulated into a large and complex class hierarchy, an example is shown in the image below. 

![Image not found or removed.]({{ "assets/images/acdbblocktable-hierarchy.png" | relative_url }})<br/>
<i>Screenshot taken from the public ObjectARX documentation.</i>

In order access any individual object in a drawing, you must request a pointer of the correct base type from the correct table and manage its lifetime. Polymorphism is
relied on heavily and you can easily find yourself working with several pointers in a single operation. Each pointer handle you request must be cleaned up in the correct
manner, otherwise you can have a memory leak, or worse, corrupt the drawing. Fortunaltely, we can start off by looking at the fundamental containers and work our way up. 

Let's try to understand the basics of tables and records. There are several different table types in the database, each one having it's own iterator and record types.
Each record corrosponds to an object of some description. You work with all three through raw pointers. Before the lifetimes of those pointers end, you must clean up after
yourself. Pointers to objects that are owned by the drawing itself must be release using a close() function, even if you allocated the memory yourself. Pointers that are 
allocated by you and never passed on to the database must be cleaned up using the **delete** operator.

To create order in this indirection, the C++ type system gives us templates, concepts, and smart pointers. To start with, database objects must be clearly distinguished by
their cleanup function. I created concepts to accomplish this:

```cpp
template<typename T>
concept db_object = std::derived_from<T, AcDbObject>;

template <typename T>
concept closable = db_object<T> && requires (T* t)
{
  {t->close()} -> std::same_as<Acad::ErrorStatus>;
  };
```

By using concepts like this, I can create a std::unique_ptr type alias that automatically closes the object I'm using:

```cpp
template <closable T>
struct ptr_closer{ constexpr auto operator()(T* t) noexcept {
  if (t != nullptr) { t->close(); } } 
};

template <closable T>
using unique_closable = std::unique_ptr<T, ptr_closer<T>>;
```

Shared pointers work slightly differently, you pass the deleter function at construction. To accomodate this I wrote a factory function. I also wrote a 
similar function for unique_closable, for consistency:

```cpp
template<closable T, typename... Args>
constexpr auto make_unique_closable(Args&&... args) {
  return unique_closable<T>{
    std::make_unique<T>(std::forward<Args>(args)...)
  };
}

template<closable T, typename... Args>
constexpr auto make_shared_closable(Args&&... args) {
  return std::shared_ptr<T>{
    new T{std::forward<Args>(args)...}, ptr_closer<T>{}
  };
}
```

Great! Now we can create and release resource handles appropriately. What about tables and records? I utilized template specialization to make them and their
iterators strongly typed.

```cpp
template <typename T>
struct table_sub_types;

// An example of a specialization of the template above.
template <>
struct table_sub_types<AcDbBlockTable> {
  using record_type = AcDbBlockTableRecord;
  using iterator_type = AcDbBlockTableIterator;
};
```

From that I created concepts that would ensure all the required type information was present. This will all make sense in a second, trust me.

```cpp
template <typename T>
concept closable_db_table = closable<T> && requires (T)
{
  typename table_sub_types<T>::record_type;
  typename table_sub_types<T>::iterator_type;
};

template <typename T>
concept db_table_iterator = std::derived_from<T, AcDbSymbolTableIterator>;

template <typename T>
concept db_table_record = std::derived_from<T, AcDbSymbolTableRecord>;
```

From here, we can construct a generic table class that encapsulates all the related types, utilizing a shared_closable smart pointer to automatically handle 
cleanup. 

```cpp
template <closable_db_table T>
class db_table {
public:

  using record_type = table_sub_types<T>::record_type;
  using iterator = table_sub_types<T>::iterator_type;

  class iterator_t {
  public:
    using value_type = iterator*; // More about iterators later.
// The rest of the class is clipped.

// Using this API, and creating helpful type aliases, we can open the 
// table we want simply by passing a pointer to the database.
auto table = db_block_table{database_ptr}; // db_block_table is an alias for 
                                           // db_table<AcDcBlockTable>.

// And we can access records just by providing the name of the record, in unicode.
// This uses C++ 17's if statements with initializers.
if(const auto my_record = table.import_record(L"some_record_object"); my_record){
  // Carry on.
}
```

Cool stuff! A mess of steps all packaged into a class. Now accessing the data we want is only a few lines of code. I'd call this a win, but we are not done yet.
We still need to iterate over tables and records.

To handle iterating, I took some inspiration from the C++ Standard Library's functional style algorithms. I wrote generic functions that took a C++ lambda as the 
operation to use on each iteration. I also created a way to pass function objects/lambdas as predicates for filtering through iterations. This allows for more 
control over what objects get processed without writing the if statement yourself.

```cpp
template<typename F, typename Result, typename Iter>
concept table_predicate = db_table_iterator<Iter> && std::invocable<F, Iter*> 
&& requires (F f, Iter it)
{
  { f(&it) } -> std::same_as<Result>;
};

template<typename F, typename Result>
concept object_predicate = std::invocable<F, AcDbBlockTableRecordIterator*> 
&& requires (F f, AcDbBlockTableRecordIterator it)
{
  { f(&it) } -> std::same_as<Result>;
};

// An algorithm function for iterating through model space in a drawing.
template<object_predicate<bool>... Preds>
auto for_each_model_space(const db_block_table& table, std::invocable<AcDbBlockTableRecordIterator*> auto&& func,
  Preds&&... preds) {

  const auto model_space = table.import_record(ACDB_MODEL_SPACE, AcDb::kForWrite);
  AcDbBlockTableRecordIterator* temp_it{};
  model_space->newIterator(temp_it);

  for (const auto it_2 = std::unique_ptr<AcDbBlockTableRecordIterator>(temp_it); !it_2->done(); it_2->step()) {

    // This decides, at compile time, whether to incurr the overhead of evaluating predicates.
    if constexpr (sizeof...(preds) > 0uz) {
      if ((std::forward<Preds>(preds)(it_2.get()) && ...)) {
        std::invoke(func, it_2.get());
      }
    } else {
      std::invoke(func, it_2.get());
    }
  }
}
```

I know that looks horrifying at first. But with it, we can do this:

```cpp
auto my_table = db_block_table{database_ptr};
for_each_model_space(my_table, [&](const AcDbBlockTableRecordIterator* it) -> auto {
  // Do something with the records
},
is_table{});
```

With this interface, we have eliminated a lot of clutter from our client code. Accessing tables is only 2 lines of code. We've eliminated the possibility of accessing
data through the wrong record or iterator types. We've created a way to iterate through tables simply by providing a lambda and a variadic list of preconditions, 
eliminating many places where we would be working with raw pointers. This also reduces the need to remember to validate pointers. And we've made releasing resources
automatic.

This is just the beginning.
