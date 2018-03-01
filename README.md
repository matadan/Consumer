[![Travis](https://img.shields.io/travis/nicklockwood/Consumer.svg)](https://travis-ci.org/nicklockwood/Consumer)
[![Coveralls](https://coveralls.io/repos/github/nicklockwood/Consumer/badge.svg)](https://coveralls.io/github/nicklockwood/Consumer)
[![Platform](https://img.shields.io/cocoapods/p/Consumer.svg?style=flat)](http://cocoadocs.org/docsets/Consumer)
[![Swift 3.2](https://img.shields.io/badge/swift-3.2-orange.svg?style=flat)](https://developer.apple.com/swift)
[![Swift 4.0](https://img.shields.io/badge/swift-4.0-red.svg?style=flat)](https://developer.apple.com/swift)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](https://opensource.org/licenses/MIT)
[![Twitter](https://img.shields.io/badge/twitter-@nicklockwood-blue.svg)](http://twitter.com/nicklockwood)

- [Introduction](#introduction)
	- [What?](#what)
	- [Why?](#why)
	- [How?](#how)
- [Usage](#usage)
    - [Installation](#installation)
    - [Parsing](#parsing)
    - [Transforming](#transforming)
    - [Common Transforms](#common-transforms)
    - [Typed Labels](#typed-labels)
    - [Forward References](#forward-references)
- [Example Projects](#example-projects)
    - [JSON](#json)
    - [BASIC](#basic)


# Introduction

## What?

Consumer is a library for Mac and iOS for parsing structured text such as a configuration file, or a programming language source file.

The primary interface is the `Consumer` type, which is used to programmatically build up a parsing grammar.

Using that grammar, you can then parse String input into an AST (Abstract Syntax Tree), which can then be transformed into application-specific data

## Why?

There are many situations where it is useful to be able to parse structured data. Most popular file formats have some kind of parser, typically either written by hand or by using code generation. 

Writing a parser is a time-consuming and error-prone process. Many tools exist in the C world for generating parsers, but relatively few such tools exist for Swift.

Swift's strong typing and sophisticated enum types make it well-suited for creating parsers, and Consumer takes advantage of these features.

## How?

Consumer uses an approach called *recursive descent* to parse input. Each `Consumer` instance consists of a tree of sub-consumers, with the leaves of the tree matching individual strings or characters in the input.

You build up a consumer by starting with simple rules that match individual words or values (known as "tokens") in your language or file format. You then compose these into more complex rules that match sequences of tokens, and so on until you have a single consumer that describes an entire document in the language you are trying to parse.


# Usage

## Installation

The `Consumer` type and its dependencies are encapsulated in a single file, and everything public is prefixed or name-spaced, so you can simply drag `Consumer.swift` into your project to use it.

If you prefer, there's a framework for Mac and iOS that you can import which includes the `Consumer` type. You can install this manually, or by using CocoaPods, Carthage, or Swift Package Manager.

To install Consumer using CocoaPods, add the following to your Podfile:

```ruby
pod 'Consumer', '~> 0.1'
```

To install using Carthage, add this to your Cartfile:

```
github "nicklockwood/Consumer" ~> 0.1
```

## Parsing

The `Consumer` type is an enum, so you can create a consumer by just assigning one of its possible values to a variable. For example, here is a consumer that matches the string "foo":

```swift
let foo: Consumer<String> = .string("foo")
```

To parse a string with this consumer, call the `match()` function:

```swift
do {
    let match = try foo.match("foo")
    print(match) // Prints the AST
} catch {
    print(error)
}
```

In this simple example above, the match will always succeed. If tested against arbitrary input, the match will potentially fail, in which case an Error will be thrown. The Error will be of type `Consumer.Error`, which includes information about the error type and the location in the input string where it occurred.

The example above is not very useful - there are much simpler ways to detect string equality! Let's try a slightly more advanced example. The following consumer matches an unsigned integer:

```swift
let integer: Consumer<String> = .oneOrMore(.charInRange("0", "9"))
```

The top-level consumer in this case is of type `oneOrMore`, meaning that it matches one or more instances of the nested `charInRange("0", "9")` consumer. In other words, it will match any sequence of numeric digits.

There's a slight problem with this implementation though: An arbitrary sequence of digits might include leading zeros, e.g. `01234`, which could be mistaken for an octal number in some programming languages, or even just be treated as a syntax error. How can we modify the `integer` consumer to reject leading zeros?

We need to treat the first character differently from the subsequent ones, which means we need two different parsing rules to be applied in *sequence*. For that, we use a `sequence` consumer:

```swift
let integer: Consumer<String> = .sequence([
    .charInRange("1", "9"),
    .zeroOrMore(.charInRange("0", "9")),
])
```

So instead of `oneOrMore` digits in the range 0 - 9, we're now looking for a single digit in the range 1 - 9, followed by `zeroOrMore` digits in the range 0 - 9. That means that a zero preceeding a nonzero digit will not be matched.

```swift
do {
    _ = try integer.match("0123")
} else {
    print(error) // Unexpected token "0123" at 0
}
```

We've introduced another bug though - Although leading zeros are correctly rejected, `0` on its own will now also be rejected since it doesn't start with 1 - 9. We need to accept *either* zero on its own, *or* the sequence we just defined. For that, we can use `anyOf`:

```swift
let integer: Consumer<String> = .anyOf([
    .string("0"),
    .sequence([
        .charInRange("1", "9"),
        .zeroOrMore(.charInRange("0", "9")),
    ]),
])
```

That will do what we want, but it's quite a bit more complex. To make it more readable, we could break it up into separate variables:

```swift
let zero: Consumer<String> = .string("0")
let oneToNine: Consumer<String> = .charInRange("1", "9")
let zeroToNine: Consumer<String> = .charInRange("0", "9")

let nonzeroInteger: Consumer<String> = .sequence([
    .oneToNine, .zeroOrMore(zeroToNine),
])

let integer: Consumer<String> = .anyOf([
    zero, nonzeroInteger,
])
```

We can then further extend this with extra rules, e.g.

```swift
let sign = .charInString("-+")

let signedInteger: Consumer<String> = .sequence([
    .optional(sign), integer,
])
```

## Transforming

In the previous section we wrote a consumer that can match an integer number. But what do we get when we apply that to some input? Here is the matching code:

```swift
let match = try integer.match("1234")
print(match)
```

And here is the output:

```
(
    '1'
    '2'
    '3'
    '4'
)
```

That's ... odd. You were probably hoping for a String containing "1234", or at least something a bit simpler to work with.

If we dig in a bit deeper and look at the structure of the `Match` value returned, we'll find it's something like this (omitting namespaces and other metadata for clarity):

```swift
Match.node([
    Match.token("1", 0 ..< 1),
    Match.token("2", 1 ..< 2),
    Match.token("3", 2 ..< 3),
    Match.token("4", 3 ..< 4),
])
```

Because each digit in the number was matched individually, the result has been returned as an array of tokens, rather than a single token representing the entire number. This level of detail is potentially useful for some applications, but we don't need it right now - we just want to get the value. To do that, we need to *transform* the output.

The `Match` type has a method called `transform()` for doing exactly that. The `transform()` method takes a closure argument of type `(_ name: Label, _ value: Any) throws -> Any?` and returns an `Any?` value. The closure is applied recursively to all matched values in order to convert them to whatever form your application needs.

Unlike parsing, which is done from the top down, transforming is done from the bottom up. That means that the child nodes of each `Match` will be transformed before their parents, so that all the values passed to the transform closure should have already been converted to the expected types.

So the transform function takes a value and returns a value - pretty straightforward - but you're probably wondering about the `Label` argument. If you look at the definition of the `Consumer` type, you'll notice that it also takes a generic argument of type `Label`. In the examples so far we've been passing `String` as the label type, but we've not actually used it yet.

The `Label` type is used in conjunction with the `label` consumer type. This effectively allows you to assign a name to a given consumer rule, which can be used to refer to it later. Since you can store consumers in variables and refer to them that way, it's not immediately obvious why this is useful, but it has two purposes:

The first purpose is to allow [forward references](#forward-references), which are explained below.

The second purpose is for use when tranforming, to identify the type of node to be transformed. Labels assigned to consumer rules are preserved in the `Match` node after parsing, making it possible to identify which rule was matched to create a particular type of value. Matched values that are not labelled cannot be individually transformed, they will instead be be passed as the value for the first labelled parent node.

So, to transform the integer result, we must first give it a label, by using the `label` consumer type:

```swift
let integer: Consumer<String> = .label("integer", .anyOf([
    .string("0"),
    .sequence([
        .charInRange("1", "9"),
        .zeroOrMore(.charInRange("0", "9")),
    ]),
]))
```

We can then transform the match using the following code:

```swift
let result = try integer.match("1234").transform { label, value in
    switch label {
    case "integer":
        return (value as! [String]).joined()
    default:
        preconditionFailure("unhandled rule: \(name)")
    }
}
print(result ?? "")
```

We know that the `integer` consumer will always return an array of string tokens, so we can safely use `as!` in this case to cast the value to `[String]`. This is not especially elegant, but its the nature of dealing with dynamic data in Swift. Safety purists may prefer to use `as?` and throw an `Error` or return `nil` if the value is not a `[String]`, but that situation could only arise in the event of a programming error - no input data matched by the `integer` consumer we've defined above will ever return anything else.

With the addition of this function, the array of character tokens is transformed into a single string value. The printed result is now simply "1234". That's much better, but it's still a string, and we may well want it to be an actual `Int` if we're going to use the value. Since the `transform` function returns `Any?`, we can return any type we want, so let's modify it to return an `Int` instead:

```swift
switch label {
case "integer":
    let string = (value as! [String]).joined()
    guard let int = Int(string) else {
        throw MyError(message: "Invalid integer literal '\(string)'")
    }
    return int
default:
    preconditionFailure("unhandled rule: \(name)")
}
```

The `Int(_ string: String)` initializer returns an optional in case the string supplied cannot be converted to an `Int`. Since we've already pre-determined that the string only contains digits, you might think we could safely force unwrap this, but it is still possible for the initializer to fail - the matched integer might have too many digits to fit into 64 bits, for example.

We could also just return `Int(string)` directly, since the return type for the transform function is `Any?`, but this would be a mistake because that would silently omit the number from the output, and we actually want to treat it as an error instead. We've used an imaginary error type called `MyError` here, but you can use whatever type you like. Consumer will wrap the erro you throw in a `Consumer.Error` before returning it, which will annotate it with the source input offset and other useful metatdata preserved from the parsing process.

## Common Transforms

Certain types of transform are very common. In addition to the Array -> String converstion we've just done, other examples include discarding a value (equivalent to returning `nil` from the transform function), or substituting a given string for a different one (e.g. replace "\n" with a newline character, or vice-versa).

For these common operations, rather than applying a label to the consumer and having to write a transform function, you can use one of the built-in consumer transforms: 

* `flatten` - flattens a node tree into a single string token
* `discard` - removes a matched string token or node tree from the results
* `replace` - replaces a matched node tree or string token with a different string token

Note that these transforms are applied during the parsing phase, before the `Match` is returned or the regular `transform()` function can be applied.

Using the `flatten` consumer, we can simplify our integer transform a bit:

```swift
let integer: Consumer<String> = .label("integer", .flatten(.anyOf([
    .string("0"),
    .sequence([
        .charInRange("1", "9"),
        .zeroOrMore(.charInRange("0", "9")),
    ]),
])))

let result = try integer.match("1234").transform { label, value in
    switch label {
    case "integer":
        let string = value as! String // matched value is now a string
        guard let int = Int(string) else {
            throw MyError(message: "Invalid integer literal '\(string)'")
        }
        return int
    default:
        preconditionFailure("unhandled rule: \(name)")
    }
}
```

## Typed Labels

Besides the need for force-unwrapping, another inelegance in our transform function is the need for the `default:` clause in the `switch` statement. Swift is trying to be helpful here by insisting that we handle all possible label values, but we *know* that "integer" is the only possible label in this code, so the `default:` is redundant.

Fortunately, Swift's type system can help here. Remember that the label value is not actually a `String` but a generic type `Label`. This allows use to use any type we want for the label (provided it conforms to `Hashable`), and a really good approach is to create an `enum` for the `Label` type:

```swift
enum MyLabel: String {
    case integer
}
```

If we now change our code to use this `MyLabel` enum instead of `String`, we not only avoid error prone copying and pasting of string literals, but we eliminate the need for the `default:` clause in the transform function, since Swift can now determine statically that `integer` is the only possible value. The complete code is shown below:

```swift
enum MyLabel: String {
    case integer
}

let integer: Consumer<MyLabel> = .label(.integer, .flatten(.anyOf([
    .string("0"),
    .sequence([
        .charInRange("1", "9"),
        .zeroOrMore(.charInRange("0", "9")),
    ]),
])))

enum MyError: Error {
    let message: String
}

let result = try integer.match("1234").transform { label, value in
    switch label {
    case .integer:
        let string = value as! String
        guard let int = Int(string) else {
            throw MyError(message: "Invalid integer literal '\(string)'")
        }
        return int
    }
}
print(result ?? "")
```

## Forward References

More complex parsing grammars (e.g. for a programming language or a structured data file) may require circular references between rules. For example, here is an abridged version of the grammar for parsing JSON:

```swift
let null: Consumer<String> = .string("null")
let bool: Consumer<String> = ...
let number: Consumer<String> = ...
let string: Consumer<String> = ...
let object: Consumer<String> = ...

let array: Consumer<String> = .sequence([
    .string("["),
    .optional(.interleaved(json, ","))
    .string("]"),
])

let json: Consumer<String> = .anyOf([null, bool, number, string, object, array])
```

The `array` consumer contains a comma-delimited sequence of `json` values, and the `json` consumer can match any other type, including `array` itself.

You see the problem? The `array` consumer references the `json` consumer before it has been declared. This is known as a *forward reference*. You might think we can solve this by predeclaring the `json` variable before we assign its value, but this won't work - `Consumer` is a value type, so every reference to it is actually a copy - it needs to be defined up front.

In order to implement this, we need to make use the `label` and `reference` features. First, we must give the `json` consumer a label so that it can be referenced before it is declared:

```swift
let json: Consumer<String> = .label("json", .anyOf([null, bool, number, string, object, array]))
```

Then we replace `json` inside the `array` consumer with `.reference("json")`:

```swift
let array: Consumer<String> = .sequence([
    .string("["),
    .optional(.interleaved(.reference("json"), ","))
    .string("]"),
])
```


# JSON Example

In the Examples folder in the Consumer project you can find a full grammar definition for a [JSON](https://json.org) parser, along with a transform function to convert it into valid Swift data.

