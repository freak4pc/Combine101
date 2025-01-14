# Combine101
A gentle but thorough introduction to Combine. 😁

![width=80%](images/learnCombineHeader.png)

## What is Combine?

Combine is a new **reactive** framework created by Apple that streamlines how you can process asynchronous operations.

Combine code is **declarative** and succinct, making it easy to write and read once you have a basic understanding of it.

Combine also allows you to neatly organize related code in one place instead of having that code spread out across multiple files.

## What is a publisher?

The essence of Combine is: **publishers send values to subscribers**.

To become a publisher, a type must adopt and conform to the `Publisher` protocol, which includes the following:

### The publisher's interface
```swift
associatedtype Output
associatedtype Failure : Error
```
These `associatedtype`s define the interface of the publisher. Specifically, a publisher must define the type of values it will send, and its failure error type or `Never` if it will never send an error.

### The subscriber requests to subscribe

```swift
public func subscribe<S>(_ subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input
```

The _subscriber_ calls this method on the publisher to subscribe to it.

### The publisher creates the subscription

```swift
func receive<S>(subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input
```

The publisher calls this method on itself to actually create the subscription.

### A publisher can finish or fail

In addition to sending values, **a publisher can send a single completion event**.

Once a publisher sends a completion event, it's done and can no longer send any more values.

A completion event can either indicate that the publisher completed normally (`.finished`) or that an error has occurred (`.failure(Failure)`). If a publisher does fail with an error, it will send the error.

### How to create publishers

Combine is integrated throughout the iOS SDK and Swift standard library. For example, this enables you to create a publisher from a `NotificationCenter.Notification`, or even an array of primitive values.

```swift
let notificationPublisher = NotificationCenter.default.publisher(for: Notification.Name("SomeNotification"))

let publisher = Just("Hello, world!")
```

You can create your own custom publisher types by adopting and conforming to the `Publisher` protocol.

However there are two `Publisher` types that will most often suit your needs without having to define a custom publisher: `PassthroughSubject` and `CurrentValueSubject`.

#### `PassthroughSubject`

A passthrough subject enables you to send values through it. It will pass along values and the completion event.

```swift
// 1
let passthroughSubject = PassthroughSubject<Int, Never>()

// 2
passthroughSubject.send(0)
passthroughSubject.send(1)
passthroughSubject.send(2)
```
1. Create a passthrough subject of type `Int` that will never send an error.
1. Send the values `0`, `1`, and `2` on the passthrough subject.

#### `CurrentValueSubject`

A current value subject is initialized with a starting value that it will send to new subscribers, and you can also send new values through it in similar manner to a passthrough subject. You can also ask a current value subject for its current value at any time by accessing its `value` property.

```swift
// 1
let currentValueSubject = CurrentValueSubject<Character, Never>("A")

// 2
print(currentValueSubject.value)

// 3
currentValueSubject.send("B")
currentValueSubject.send("C")
```

1. Create a current value subject of type `String` that will never send an error, with an initial value of `"A"`.
1. Print the current value subject's `value`.
1. Send the values `"B"` and `"C"` on the current value subject.

This will print:

```none
A
```

## What is a subscriber?

A subscriber attaches to a publisher to receive values from it. This is called a subscription.

> **Note**:  A publisher will not send values until it has a subscriber.

Subscribers must adopt and conform to the `Subscriber` protocol, which includes the following:

### The subscriber's interface
```swift
associatedtype Input
associatedtype Failure: Error
```
These `associatedtype`s define the interface of the subscriber. Specifically, a subscriber must define the type of values it will receive, and the failure error type it will receive or `Never` if it will not accept an error.

### The publisher issues the subscription
```swift
func receive(subscription: Subscription)
```
The _publisher_ calls this method on the subscriber to give it the subscription to the subscriber.

### The publisher sends new values
```swift
func receive(_ input: Self.Input) -> Subscribers.Demand
```
The _publisher_ calls this method on the subscriber to send a new value to the subscriber. Notice that the return value is `Subscribers.Demand`. See the **Handling backpressure** section for more info.

### The publisher tells the subscriber when it's done or has failed
```swift
func receive(completion: Subscribers.Completion<Self.Failure>)
```
The _publisher_ calls this method to tell the subscriber that it has completed, either normally or with an error.

### How to create subscriptions

> **Note**: In order for a subscription to be created between a publisher and a subscriber, the publisher's `Output` and `Failure` types must match the subscriber's `Input` and `Failure` types.

There are two ways to create a subscription to a publisher:
1. By using one of the `sink` operators. 
1. By using one of the `assign(to:on:)` operators.

> **Note**: A subscription returns an instance of `AnyCancellable` that represents the subscription. You must save the subscription token or else the subscription will be canceled as soon as program flow exits the current scope.

There are two ways to store a subscription token:
1. As an individual value of type `AnyCancellable`
2. In a collection of `AnyCancellable`.

#### Creating subscriptions with `sink`

The `sink` operators include closure parameters to handle values or a completion event received from a publisher.

```swift
// 1
let publisher = Just("Hello, world!")

// 2
let subscription = publisher
    .sink(receiveValue: { print($0) })
```
1. Create a `Just` publisher that will send its value to each new subscriber and then complete.
1. Subscribe and print out the received value.

This will print:

```none
Hello, world!
```

You can also add subscriptions to a collection of `AnyCancellable`:

```swift
// 1
var subscriptions = Set<AnyCancellable>()

// 2
let publisher = Just("Hello, world!")

// 3
publisher
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions) // 4
```
1. Create a set of `AnyCancellable` to store subscriptions in.
1. Create a publisher.
1. Subscribe to the publisher and print out received values.
1. Store the subscription in `subscriptions`.

#### Creating subscriptions with `assign(to:on:)`

```swift
// 1
class Player {
    var score = 0 {
        didSet {
            print(score)
        }
    }
}

// 2
let player = Player()

// 3
let subscription = [10, 50, 100].publisher
    .assign(to: \.score, on: player) // 4
```
1. Define a `Player` class with a `score` property that prints its new value when set.
1. Create an instance of `Player`.
1. Create a subscription to a publisher of an array of integers.
1. Use `assign(to:on:)` to assign each value received to the `score` property on `player`.

This will print:

```none
10
50
100
```

### How to stop subscriptions

There are two ways to stop subscriptions:
1. Call `cancel()` on a subscription token.
1. Do nothing and let normal memory management rules to apply, i.e., the token or collection of `AnyCancellable` will call `cancel()` on the subscriptions upon deinitialization.

```swift
// 1
let passthroughSubject = PassthroughSubject<Int, Never>()

// 2
let subscription = passthroughSubject
    .sink(receiveCompletion: { print($0) },
          receiveValue: { print($0) })

// 3
passthroughSubject.send(0)
passthroughSubject.send(1)
passthroughSubject.send(2)

// 4
passthroughSubject.send(completion: .finished)
```
1. Create a passthrough subject of type `Int` that will never send an error.
1. Subscribe to the passthrough subject.
1. Send the values `0`, `1`, and `2` on the passthrough subject.
1. Send the completion event on the passthrough subject.

This will print:
```none
0
1
2
finished
```

### Handling backpressure

Backpressure is the pressure caused by the stream of values being sent by a publisher to a subscriber. If a publisher sends too many values to a subscriber, this can cause problems. In order to manage that backpressure, every time a subscriber receives a new value, it must tell the publisher what its willingness is to receive additional values, i.e., its **demand**. Demand can only be adjusted additively. In other words, a subscriber can _increase_ its demand every time it receives a new value, but it cannot _decrease_ it. There are three levels of demand that a subscriber can return from `receive(_:) -> Subscribers.Demand`:

- `.none`
- `.max(value: Int)`
- `.unlimited`

The `sink` and `assign(to:on:)` operators both automatically return `.unlimited` for demand. You can define custom subscribers to return a different demand, however this goes beyond the scope of this introduction.

## What are operators?

**Operators are special methods that return a publisher**.

Several operators are named and work similarly to methods found in the Swift Standard Library, such as `map`, `filter`, and `reduce`.

They can receive values from an _upstream_ publisher, perform some operation on those values, and then send those values _downstream_.

> **Note**: The terms **upstream** and **downstream** are typically used to describe the flow of a subscription. For example, an operator receives values or a completion event from an upstream publisher, it processes those values or completion event, and then it may send values or events downstream to another publisher or a subscriber.

### What are some of the most common operators?

#### Use `map` operators to transform values
The `map` category of operators provide several ways that you can transform values sent by an upstream publisher, to then send downstream.

#### Use `filter` operators to limit which values get through

The `filter` family of operators provide several ways that you can prevent or limit values received from an upstream publisher that are sent downstream.

### How to share a publisher

To understand _why_ you would want to share a subscription, review this example where two subscribers subscribe to the same publisher.

```swift
let subject = PassthroughSubject<Int, Never>()

let publisher = subject
    .handleEvents(receiveOutput: { print("Handling", $0) })

_ = publisher
    .sink(receiveValue: { print("1st subscriber", $0) })

_ = publisher
    .sink(receiveValue: { print("2nd subscriber", $0) })

subject.send(0)
subject.send(1)
```

This will print:

```none
Handling 0
1st subscriber 0
Handling 0
2nd subscriber 0
Handling 1
1st subscriber 1
Handling 1
2nd subscriber 1
```

The `handleEvents` operator includes closures to handle each event in the publisher's lifecycle:

- `receiveSubscription`
- `receiveRequest`
- `receiveOutput`
- `receiveCompletion`
- `receiveCancel`

Each subscriber independently subscribes and handles the values sent by the publisher. In order to be more efficient, you can use the `share()` operator to share the publisher to multiple subscribers.

```swift
let publisher = subject
    .handleEvents(receiveOutput: { print("Handling", $0) })
    .share()
```

This will now print:

```none
Handling 0
1st subscriber 0
2nd subscriber 0
Handling 1
1st subscriber 1
2nd subscriber 1
```

There is one caveat with `share()`: it will only share values to _existing_ subscribers.

If you add the following code to the end of the previous example:

```swift
_ = publisher
  .sink(receiveValue: { print("3rd subscriber", $0) })

subject.send(2)
```

The complete example will now print:

```none
Handling 0
1st subscriber 0
2nd subscriber 0
Handling 1
1st subscriber 1
2nd subscriber 1
Handling 2
1st subscriber 2
2nd subscriber 2
3rd subscriber 2
```

The 3rd subscriber does not get the `1` and `2` because it was not yet subscribed.

### How to see every event that occurs

One very useful operator to use when debugging Combine code is the `print()` operator. You can insert it anywhere in a publisher or subscription chain of operators.

```swift
let subject = PassthroughSubject<Int, Never>()

let publisher = subject
    .print("Publisher")
    .share()

_ = publisher
    .print("Subscriber")
    .sink(receiveValue: { print($0) })

subject.send(0)
subject.send(1)
```

This will print:

```none
Subscriber: receive subscription: (Multicast)
Subscriber: request unlimited
Publisher: receive subscription: (PassthroughSubject)
Publisher: request unlimited
Publisher: receive value: (0)
Subscriber: receive value: (0)
0
Publisher: receive value: (1)
Subscriber: receive value: (1)
1
Subscriber: receive cancel
Publisher: receive cancel
```

### What other kinds of operators are there?

Apple groups Combine operators into these categories:
- Mapping, including Encoding and Decoding
- Filtering, including Matching, Selecting, and Mathematical
- Combining, including Reducing and Sequencing
- Debugging
- Error Handling
- Scheduling
- Type-Erasing
- Adapting
- Sharing, including Multicasting
- Buffering
- Timing
- Connecting

These categories are roughly ordered from highest to lowest in terms of typical frequency of usage, and lowest to highest in terms of complexity. That makes this list a great todo list if you would like to go beyond the basics and become an expert with Combine.

### How do I create complex subscriptions involving multiple operators?

Here is an example of a subscription that involves several operators:

```swift
// 1
let formatter = NumberFormatter()
formatter.numberStyle = .spellOut

// 2
let publisher = (0..<100).publisher

// 3
let subscription = publisher
    .dropFirst()
    .filter { $0 % 2 == 0 }
    .prefix(4)
    .map { NSNumber(integerLiteral: $0)}
    .compactMap { formatter.string(from: $0) }
    .append("Done!")
    .sink(receiveValue: { print($0) }) // 4
```
1. Create a number formatter that will return a string with each number spelled out.
1. Create a publisher from a range of integers from `0` to `100`.
1. Create a subscription to the publisher, using the following operators:
- `dropFirst()` to skip the first value sent.
- `filter(_:)` to only allow even integer values to get through.
-  `prefix(_:)` to only take the first four values.
-  `map(_:)` to initialize and send downstream an `NSNumber` instance for each integer received.
-  `compactMap(_:)` to send the return from calling `formatter.string(from:)`, filtering out `nil`s.
-  `append(_:)` to append a string onto the received value and send the result downstream.
4. Subscribe and print received values in the handler.

This will print:

```none
two
four
six
eight
Done!
```

## What other things can I do with Combine?

Combine is integrated into many existing frameworks in iOS, macOS, watchOS, and tvOS.

There is also a new framework for developing user interfaces called SwiftUI that relies heavily on Combine. You can use Combine and SwiftUI to create reactive apps that require a lot less code and complexity than their predecessor frameworks, that are also much more robust and less prone to common bugs and unexpected behaviors.

## Where can I learn more?

Check out this book that I am a co-author and the technical editor for:

[Combine: Asynchronous Programming with Swift](https://store.raywenderlich.com/a/742/link/27)

<a href="https://store.raywenderlich.com/a/742/link/27" alt="Combine: Asynchronous Programming with Swift"><img src="https://github.com/learncombine/Combine101/blob/master/images/combineBook.png"></a>

It's packed with coverage of Combine concepts, hands-on exercises in playgrounds, and complete iOS app projects. Everything you need to _transform yourself_ from novice to expert with Combine — and have fun while doing it!

I also started a new [HackingCombine](https://github.com/learncombine/HackingCombine) repo where I explore uncharted territory in Combine.

Combine and SwiftUI are Copyright © 2019 Apple Inc.
