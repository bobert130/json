# Events Interface

* [Overview](#overview)
* [Consumer Interface](#consumer-interface)
* [Arrays and Objects](#arrays-and-objects)
* [Reduced Interface](#reduced-interface)
* [Included Producers](#included-producers)
* [Convenience Functions](#convenience-functions)
* [Included Consumers[#included-consumers)
* [Convenience Functions](#convenience-functions-1)
* [Included Filters](#included-filters)

## Overview

The Events interface is a calling convention, a kind of abstract (in the conceptual, not object-oriented sense) interface that allows different parts of the library to communicate with each other.

The Events interface is modelled after the SAX-interface for XML and has functions like `begin_object()` and `boolean( const bool )`.

We distinguish betwen *event producers* that usually take some input and generate calls to the Events interface from it, and *event consumers* which implement the functions of the Events interface to usually produce some output.
*Event filters* occupy both roles, accepting calls to the Events interface and transforming them into (other) calls to another Events interface implementation.

For example the `tao::json::events::from_value()` function is an Events producer that takes a Value object of type `tao::json::basic_value< Traits, Base >` for any traits class `Traits` and annotation class `Base` as input, and generates Events depending on the contents.
It could be used together with the `tao::json::events::to_string()` class, an Events consumer that generates the JSON string representation corresponding to the Events it receives.
In order to transform binary data in the Value object into something compatible with standard JSON it might be necessary to use an Events filter like `tao::json::events::binary_to_hex` between the producer and consumer.

Producers and consumers can be freely coupled, it is not necessary for the beginning or the end of the chain to be a JSON Value object.
For example an Events producer that parses CBOR can be directly used with an Events consumer that creates UBJSON.

```c++
#include <tao/json.hpp>

std::string cbor_to_ubjson( const std::string& input )
{
   tao::json::ubjson::events::to_string consumer;
   tao::json::cbor::events::from_string( consumer, input );
   return consumer.value();
}
```

A more common use case is parsing a JSON representation into a JSON Value...

```c++
tao::json::value from_string( const std::string& input )
{
   tao::json::events::to_value consumer;
   tao::json::events::from_string( consumer, input );
   return std::move( consumer.value );
}
```

...which is in fact so common that a (more general) convenience function `tao::json::from_string()` is included with the library.
See the tables below for a complete list of included Events producers and consumers, and the convenience functions that use them in conjunction with JSON Values.

## Consumer Interface

```c++
struct complete_consumer
{
   void null();
   void boolean( const bool );
   void number( const double );
   void number( const std::int64_t );
   void number( const std::uint64_t );
   void string( std::string && );
   void string( const std::string & );
   void string( const tao::string_view & );
   void binary( std::vector< tao::byte > && );
   void binary( const std::vector< tao::byte > & );
   void binary( const tao::binary_view & );
   void begin_array();
   void begin_array( const std::size_t );
   void element();
   void end_array();
   void end_array( const std::size_t );
   void begin_object();
   void begin_object( const std::size_t );
   void key( std::string && );
   void key( const std::string & );
   void key( const tao::string_view & );
   void member();
   void end_object();
   void end_object( const std::size_t );
};
```

## Arrays and Objects

For arrays and objects there are two versions of the begin and end methods, one that receives the size of the array (what `std::vector< Value >::size()` would return), or the size of the object (what `std::map< std::string, Value >::size()` would return), respectively, and one that doesn't.

An Events producer will usually only either always call the begin/end-array/object functions with a size, or those without a size, depending on whether the size is readily available.

Within an Array, i.e. after a call to `begin_array()` but before a call to the corresponding `end_array()`, the `element()` function is called after the function(s) for each array sub-value.

Similarly within an Object the sequence for every object entry is `key()`, then the function(s) for the value, then `member()`.

If the Value within an Array or Object is simple, like a string or number, it is defined by a single corresponding Events call.

However the data model allows for Arrays and Objects to be nested, wherefore the same holds for corresponding matching calls to an Event consumer's begin/end-array/object methods.
If a Value in an Array or Object is itself an Array or Object, then it will lead to at least the corresponding begin and end calls, with the possibility of arbitrary many sub-values, including arbitrarily nested other Arrays and Objects.

Calls to `element()` or `member()` are naturally understood to belong to the inner-most Array or Object, i.e. to the last begin-method without end.

An Events producer MUST NOT generate inconsistent sequences of Event calls.
For example pairing `begin_array()` with `end_object()`, using `member()` when the inner-most container is not an Object, or similar, must all be avoided.

## Reduced Interface

It will not always be necessary, desirable, or even possible, to implement all of these functions.

For example consider a consumer that writes some representation to a stream.
Such a consumer will not benefit from implementing the functions with movable arguments since it has no place to move the arguments to.
If the format it writes requires the encoding of objects and arrays to start with the number of array elements or object members, respectively, then the consumer might not implement the begin/end-array/object methods without size.

A consumer that does not need the sizes of arrays and objects in the begin/end-calls, and that does not benefit from receiving movable arguments, can implement a reduced interface that - thanks to default arguments and the view-classes' implicit constructors - can still accept all function calls that the full interface can.

```c++
struct reduced_consumer
{
   void null();
   void boolean( const bool );
   void number( const double );
   void number( const std::int64_t );
   void number( const std::uint64_t );
   void string( const tao::string_view & );
   void binary( const tao::binary_view & );
   void begin_array( const std::size_t = 0 );
   void element();
   void end_array( const std::size_t = 0 );
   void begin_object( const std::size_t = 0 );
   void key( const tao::string_view & );
   void member();
   void end_object( const std::size_t = 0 );
};
```

## Included Producers

Producers are functions that take some input from which they generate Events that they call on a consumer that they are also provided with.

**The common namespace prefix `tao::json::` is omitted.**

| Producer | Description |
| -------- | ----------- |
| `events::from_stream` | Reads JSON from a `std::istream`. |
| `events::from_string` | Reads JSON from a `std::string`. |
| `events::from_value` | Generates Events for a `basic_value<>`. |
| `events::parse_file` | Reads JSON from the file. |
| `jaxn::events::from_stream` | Reads JAXN from a `std::istream`. |
| `jaxn::events::from_string` | Reads JAXN from a `std::string`. |
| `jaxn::events::parse_file` | Reads JAXN from the file. |
| `cbor::events::from_string` | Reads CBOR from a `std::string`. |
| `cbor::events::parse_file` | Reads CBOR from the file. |
| `msgpack::events::from_string` | Reads MSGPACK from a `std::string`. |
| `msgpack::events::parse_file` | Reads MSGPACK from the file. |
| `ubjson::events::from_string` | Reads UBJSON from a `std::string`. |
| `ubjson::events::parse_file` | Reads UBJSON from the file. |

## Convenience Functions

Functions that combine the similarly named Events producer from above with an Events consumer to return a corresponding Value.

**The common namespace prefix `tao::json::` is omitted.**

| Function | Description |
| -------- | ----------- |
| `from_stream` | Reads JSON from a `std::istream`. |
| `from_string` | Reads JSON from a `std::string`. |
| `parse_file` | Reads JSON from the file. |
| `jaxn::from_stream` | Reads JAXN from a `std::istream`. |
| `jaxn::from_string` | Reads JAXN from a `std::string`. |
| `jaxn::parse_file` | Reads JAXN from the file. |
| `cbor::from_string` | Reads CBOR from a `std::string`. |
| `cbor::parse_file` | Reads CBOR from the file. |
| `msgpack::from_string` | Reads MSGPACK from a `std::string`. |
| `msgpack::parse_file` | Reads MSGPACK from the file. |
| `ubjson::from_string` | Reads UBJSON from a `std::string`. |
| `ubjson::parse_file` | Reads UBJSON from the file. |

## Included Consumers

Consumers are structs or classes that implement (a complete subset of) the Events interface functions.
Consumers are usually instantiated and then passed to an Events producer function as argument.

**The common namespace prefix `tao::json::` is omitted.**

| Consumer | Description |
| -------- | ----------- |
| `events::debug` | Writes all Events to a `std::ostream` in human-readable form. |
| `events::discard` | Does nothing with the Events it receives. |
| `events::hash` | Calculates a SHA-256 hash for the received Events. |
| `events::statistics` | Counts Events, and string and binary lengths. |
| `events::to_pretty_stream` | Writes nicely formatted JSON to a `std::ostream`. |
| `events::to_stream` | Writes compact JSON to a `std::ostream`. |
| `events::to_string` | Writes compact JSON to a `std::string`. |
| `events::to_value` | Builds a Value with the received Events. |
| `events::validate_event_order` | Checks whether the order of received Events is consistent. |
| `jaxn::events::to_pretty_stream` | Writes nicely formatted JAXN to a `std::ostream`. |
| `jaxn::events::to_stream` | Writes compact JAXN to a `std::ostream`. |
| `jaxn::events::to_string` | Writes compact JAXN to a `std::string`. |
| `cbor::events::to_stream` | Writes CBOR to a `std::ostream`. |
| `cbor::events::to_string` | Writes CBOR to a `std::string`. |
| `msgpack::events::to_stream` | Writes MSGPACK to a `std::ostream`. |
| `msgpack::events::to_string` | Writes MSGPACK to a `std::string`. |
| `ubjson::events::to_stream` | Writes UBJSON to a `std::ostream`. |
| `ubjson::events::to_string` | Writes UBJSON to a `std::string`. |

## Convenience Functions

Functions that combine the similarly named Events consumers with a producer that generates Events from a Value.

**The common namespace prefix `tao::json::` is omitted.**

| Function | Description |
| -------- | ----------- |
| `to_stream` | Writes compact or nicely formatted JSON to a `std::ostream`. |
| `to_string` | Writes compact or nicely formatted JSON to a `std::string`. |
| `jaxn::to_stream` | Writes compact or nicely formatted JAXN to a `std::ostream`. |
| `jaxn::to_string` | Writes compact or nicely formatted JAXN to a `std::string`. |
| `jaxn::to_stream` | Writes CBOR to a `std::ostream`. |
| `jaxn::to_string` | Writes CBOR to a `std::string`. |
| `msgpack::to_stream` | Writes MSGPACK to a `std::ostream`. |
| `msgpack::to_string` | Writes MSGPACK to a `std::string`. |
| `ubjson::to_stream` | Writes UBJSON to a `std::ostream`. |
| `ubjson::to_string` | Writes UBJSON to a `std::string`. |

Optionally apply an arbitrary list of Filters given as additional template parameters.

## Included Filters

Filters are structs or classes that implement the Events functions as they are consumers that then produce events for another consumer.

**The common namespace prefix `tao::json::` is omitted.**

| Filter | Description |
| ------ | ----------- |
| `events::binary_to_base64` | Passes through all Events except for binary data which is converted to base64-encoded strings. |
| `events::binary_to_base64url` | Passes through all Events except for binary data which is converted to URL-safe base64-encoded strings. |
| `events::binary_to_exception` | Passes through all Events except for binary data which provokes an exception. |
| `events::binary_to_hex` | Passes through all Events except for binary data which is is converted to hex-dumped strings. |
| `events::key_camel_case_to_snake_case` | Passes through all Events except for keys in Objects which are converted from "CamelCaseStyle" to "snake_case_style". |
| `events::key_snake_case_to_camel_case` | Passes through all Events except for keys in Objects which are converted from "snake_case_style" to "CamelCaseStyle". |
| `events::limit_nesting_depth` | Passes through all Events but throws an exception when the nesting of Arrays and Objects exceeds a limit. |
| `events::limit_value_count` | Passes through all Events but throws an exception when the total number of (sub)-value exceeds a limit. |
| `events::events::non_finite_to_exception` | Passes through all Events except for numbers of type `double` which contain non-finite values which provoke an exception. |
| `events::non_finite_to_null` | Passes through all Events except for numbers of type `double` which contain non-finite values which are passed on as `null()`. |
| `events::non_finite_to_string` | Passes through all Events except for numbers of type `double` which contain non-finite values which are passed on as appropriate `string()`. |
| `events::prefer_signed` | Passes through all Events except for numbers of type `std::uint64_t` which fit into a `std::int64_t` and are passed on as such. |
| `events::prefer_unsigned` | Passes through all Events except for numbers of type `std::int64_t` which fit into a `std::uint64_t` and are passed on as such. |
| `events::ref` | Passes all Events to another consumer or filter to which it holds a C++ reference. |
| `events::tee` | Passes all Events to an arbitrary number of other consumers or filters which it holds in a `std::tuple<>`. |
| `events::validate_keys` | Passes through all Events except for keys which are first validated against a provided PEGTL grammar rule. |
| `events::virtual_ref` | Like `ref`, but implements the virtual Event functions from `virtual_base`. |

# Still Need Description

* `events::apply`
* `events::basic_compare`
* `events::compare`
* `events::transformer`
* `nlohmann::from_value`
* `nlohmann::to_value`
* TODO: Anything missing?

Copyright (c) 2018 Dr. Colin Hirsch and Daniel Frey
