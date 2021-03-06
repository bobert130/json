# Type Traits

* [Overview](#overview)
* [Create JSON Value from (custom) type](#create-value-from-type)
* [Convert JSON Value into (custom) type](#convert-value-into-type)
* [Compare JSON Value to (custom) type](#compare-value-with-type)
* [Produce JSON Events from (custom) type](#produce-events-from-type)
* [Default Traits Specialisations](#default-traits-specialisations)
* [Default Key for Objects](#default-key-for-objects)

For brevity we will often write "the traits" instead of "the (corresponding/appropriate/whatever) specialisation of the traits class template".

## Overview

The class template passed as `Traits` template parameter, most prominently to `tao::json::basic_value<>`, controls the interaction between the JSON library and other C++ types.

The library includes the traits class template `tao::json::traits<>` with [many specialisations](#default-traits) that is used throughout the library as default.
A custom traits class can be used to change the behaviour of the default traits, and/or to add support for new types.

In the first case it is necessary to create a new traits class template.
In the second case it is also possible to (partially) specialise `tao::json::traits<>` for the new types.

For a type `T`, the traits class template instantiated with `T` as template argument is used as traits class for `T`.

The default traits use a second template parameter that defaults to `void` to allow to selective enable or disable a particular (partial) specialisation via SFINAE.
For this reason, any template that takes a traits class template as template parameter should be templated over `template< typename... > class Traits`, rather than the more obvious `template< typename > class Traits`.

We will use the following class in the example implementations of the traits functions.

```c++
struct my_type
{
   std::string title;
   std::vector< int > values;
};
```

While we could simply specialise `tao::json::traits<>` for `my_type`, we will prefer to define a new traits class, an approach that also works when redefining traits for types that the default traits already cover.

```c++
template< typename T >
struct my_traits
   : public tao::json::traits< T >
{
   // Inherit the default traits by default,
   // otherwise we would have to implement
   // specialisations for ALL types (that we
   // want to use).
};
```

For convenience, we might add a `using` for the JSON Value class with the new traits.

```c++
using my_value = tao::json::basic_value< my_traits >;
```

Or, if we are planning on using [custom value annotations](Advanced-Use-Cases.md#custom-value-annotations), we might be more elaborate.

```c++
template< typename Base >
using my_basic_value = tao::json::basic_value< my_traits, Base >;
using my_value = tao::json::basic_value< my_traits >;
```

Note that all traits functions are `static` member functions, and that not a traits specialisation need not implement all traits functions.

## Create Value from Type

The traits' `assign()` functions are used to create Value instances from any type `T`.
The first argument is always a default-initialised `tao::json::basic_value<>` that reports `tao::json::UNINITIALIZED` as type, wherefore it is safe to call the `unsafe_assign_foo()` methods on it.

We will use the implementation of `assign()` in the specialisation of `my_traits` for `my_type` as example.

```c++
template<>
struct my_traits< my_type >
{
   template< template< typename... > class Traits, typename Base >
   static void assign( tao::json::basic_value< Traits, Base >& v, const my_type& t ) noexcept
   {
      v = {
         { "title", t.title },
         { "values", t.values }
      };
   }
};
```

The first thing to note is that `assign()` is "templated over" the traits class template.
If it weren't templated over the traits, the `assign()` function would only work with `my_traits`, but not when using a third traits class template that inherits from `my_traits` by default without changing the specialisation for `my_type`.

For greater flexibility and future compatibility, we recommend to *always* template over the traits, regardless of whether creating other traits class templates based on the traits in question is planned or not.

The second point of interest is that we have chosen to encode `my_type` as JSON Object with two sub-values, one for the title and one for the values, using the [initialiser-list syntax for Objects as explained in the documentation of the Value class](Value-Class.md).

Nothing more needs to be done for `t.title` and `t.values` since the default traits already know how to create Values from `std::string`, `std::vector<>`, and `int`, and therefore also `std::vector< int >`.
The default traits use JSON Arrays for vectors, lists and sets.

Given the above traits specialisation for `my_type` it is now possible to write all of the following.

```c++
const my_type a = make_my_type();
const my_value v1 = a;
const my_value v2 = {
   { "it's my type", a }
};

const std::list< my_type > l = make_list_of_my_type();
const my_value v3 = l;
```

As a performance optimisation it is possible to provide an additional overload of `assign()` for the case that the source `my_type` instance is `move()`-able.
The moving `assign()` is of course only beneficial when there is data that can actually be moved, like `title` in the example.
The vector `values` can not be moved to the JSON Value since it is a `std::vector< int >`, not one of the types used to represent data in a JSON Value.

```c++
template<>
struct my_traits< my_type >
{
   template< template< typename... > class Traits, typename Base >
   static void assign( tao::json::basic_value< Traits, Base >& v, my_type&& t ) noexcept
   {
      v = {
         { "title", std::move( t.title ) },
         { "values", t.values }
      };
   }
};
```

There are of course other possibilities for the JSON structure for `my_type`, for example it would be possible to create a flat JSON Array with the title as first element followed by the integers from `values`.
In the following we will assume the implementation from above.

## Convert Value into Type

The traits' `as()` functions are used to convert Values into any type `T`.

There are two possible signatures for the `as()` function of which *only one needs to be implemented*.
If not particularly awkward or slow it is recommended to implement the version that returns the `T`.

The user-facing functions `tao::json::as()` and `tao::json::basic_value<>::as()` are always available in both versions, regardless of which version the traits implement, subject to the following limitation:

* In order to provide the one-argument version when the traits implement only the two-argument form the type has to be default-constructible.

Both possible implementations are shown even though a real-world traits class will typically only implement either one of them.

```c++
template<>
struct my_traits< my_type >
{
   template< template< typename... > class Traits, typename Base >
   static void as( const tao::json::basic_value< Traits, Base >& v, my_type& d )
   {
      const auto& object = v.get_object();
      d.title = v.at( "title" ).template as< std::string >();
      d.values = v.at( "values" ).template as< std::vector< int > >();
   }

   template< template< typename... > class Traits, typename Base >
   static my_type as( const tao::json::basic_value< Traits, Base >& v )
   {
      my_type result;
      const auto& object = v.get_object();
      result.title = v.at( "title" ).template as< std::string >();
      result.values = v.at( "values" ).template as< std::vector< int > >();
      return result;
   }
};
```

The `as<>()` functions again template over the traits class, just like `assign()`, and for the same reasons.

The employed `get_object()`, `as<>()` and `at()` functions will all throw an exception when something goes wrong, i.e. when the accessed JSON Value is not of the correct type, or when the indexed key does not exist.

In this example no error is thrown when the top-level JSON Object contains additional keys beyond `"title"` and `"values"`.

## Compare Value with Type

The comparison operators `==`, `!=`, `<`, `<=`, `>` and `>=` in namespace `tao::json` that compare instances of `basic_value<>` with other types *ideally* use the traits' `equal()`, `greater_than()` and `less_than()` functions.

That is "ideally", because as long as the traits for the type in question have an `assign()` function, the comparison operators will still work -- by creating a temporary JSON Value and then compare the two Values.
In other words, adding `equal()`, `greater_than()` and `less_than()` functions to traits is "only" a performance optimisation, it prevents the creation of the temporary JSON Value from the non-Value argument.

The `equal()` function has to check whether an instance `d` of the type for which the traits are specialised is equal to a Value.
This check should be consistent with the other traits functions, i.e. the check should return `true` if and only if comparing the Value to a Value created with the traits' `assign()` function (or the traits' `produce()` function together with `tao::json::events::to_value`) would also return `true`.

```c++
template<>
struct my_traits< my_type >
{
   template< template< typename... > class Traits, typename Base >
   static bool equal( const tao::json::basic_value< Traits, Base >& v, const my_type& d ) noexcept
   {
      if( !v.is_object() ) {
         return false;
      }
      const auto& o = v.unsafe_get_object();
      const auto i = o.find( "title" );
      const auto j = o.find( "values" );
      return ( o.size() == 2 )
          && ( i != o.end() )
          && ( i->second == d.title )
          && ( j != o.end() )
          && ( j->second == d.values );
   }
};
```

The other two functions, `less_than()` and `greater_than()`, have the same signature, and need to return whether the first argument is less than, or greater than the second, respectively.
The same consistency conditions as for `equal()` should be applied.
When all traits functions are consistent which each other then the following assertions will never fail, regardless of the values of `m` and `d`.

```c++
const my_value m = some_value();

const my_type d = make_my_type();
const my_value v = d;

assert( ( d == m ) == ( v == m ) );
assert( ( d != m ) == ( v != m ) );
assert( ( d < m ) == ( v < m ) );
assert( ( d <= m ) == ( v <= m ) );
assert( ( d > m ) == ( v > m ) );
assert( ( d >= m ) == ( v >= m ) );

// When the traits have a produce() function:

const auto e = tao::json::produce::to_value< my_traits >( d );

assert( ( d == m ) == ( e == m ) );
assert( ( d != m ) == ( e != m ) );
assert( ( d < m ) == ( e < m ) );
assert( ( d <= m ) == ( e <= m ) );
assert( ( d > m ) == ( e > m ) );
assert( ( d >= m ) == ( e >= m ) );
```

## Produce Events from Type

Producing [JSON Events](Events-Interface.md) from a type is performed by the type's traits' `produce()` function.

Implementing the `produce()` function is only required for some optimisation techniques, namely:

1. To directly serialise any type to JSON or another supported file format.
2. To create an [Opaque pointer JSON Value](Advanced-Use-Cases.md#instance-sharing-with-opaque-pointers) instead of using building the JSON data structure for a type.

The `produce()` function receives an arbitrary [Events consumer](Events-Interface.md) as first argument, and can call any [Events functions](Events-Interface.md) on it.

It should/must again template over the traits, and, since there is no `basic_value<>` instance from which the traits can be derived, the traits `produce()` functions and the functions in namespace `tao::json::produce` need to be called with the traits class template as explicit template parameter.

```c++
template<>
struct my_traits< my_type >
{
   template< template< typename... > class Traits, typename Consumer >
   static void produce( Consumer& c, const my_type& d )
   {
      c.begin_object( 2 );
      c.key( "title" );
      tao::json::produce< Traits >( c, d.title );
      c.member();
      c.key( "values" );
      tao::json::produce< Traits >( c, d.values );
      c.member();
      c.end_object( 2 );
   }
};
```

For the first use case mentioned above, the following directly and efficiently generates the JSON representation of a `my_type` instance without creating any JSON Value instance along the way.

```c++
const my_type d = make_my_type();
const std::string json = tao::json::produce::to_string< my_traits >( d );
```

As with the `assign()` functions, and depending on which Events consumers are used, it might be beneficial to provide a moving-version of `produce()` that moves strings and binary data.

## Default Traits Specialisations

The defaults traits support the following types.

| Specialised for | Remarks |
| -------------- | -------- |
| `null_t` | |
| `bool` | |
| *signed integers* | |
| *unsigned integers* | |
| `double`, `float` | |
| `empty_binary_t` | |
| `empty_array_t` | |
| `empty_object_t` | |
| `std::string` | |
| `tao::string_view` | |
| `const char*` | |
| `const std::string&` | |
| `std::vector< tao::byte >` | |
| `tao::binary_view` | |
| `const std::vector< tao::byte >&` | |
| `std::vector< basic_value< Traits, Base > >` | Corresponds to JSON Array. |
| `basic_value< Traits, Base >*` | Creates Raw pointer; must not be `nullptr`. |
| `const basic_value< Traits, Base >*` | Creates Raw pointer; must not be `nullptr`. |
| `std::map< std::string, basic_value< Traits, Base > >` | Corresponds to JSON Object. |
| `tao::optional< T >` | Empty optional corresponds to JSON Null. |
| `std::shared_ptr< T >` | Null pointer corresponds to JSON Null. |
| `std::unique_ptr< T >` | Null pointer corresponds to JSON Null. |
| `std::list< T >` | Corresponds to JSON Array. |
| `std::set< T >` | Corresponds to JSON Array. |
| `std::vector< T >` | Corresponds to JSON Array. |
| `std::map< std::string, T >` | Corresponds to JSON Object. |

*`std::pair` and `std::tuple` coming soon.*

The type traits correctly work with nested types.
Given that `std::string`, `double`, `std::vector`, `std::shared_ptr`, and `std::map` with `std::string` as `key_type` are supported, so is for example the following type:

```c++
std::shared_ptr< std::map< std::string, std::shared_ptr< std::vector< double > > > >
```

## Default Key for Objects

The use of default keys is [shown in the section on creating Values](Value-Class.md#creating-values).

The default key for a type is a C-string that needs to be declared in the traits specialisation.

```c++
template<>
struct my_traits< my_type >
{
   static const char* default_key;
};
```

And this is the corresponding definition that needs to be placed in an implementation (`.cpp`) file.

```c++
const char* my_traits< my_type >::default_key = "fraggle";
```

The default traits supplied with the library do not define default keys for any type.

Copyright (c) 2018 Dr. Colin Hirsch and Daniel Frey
