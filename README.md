![](https://github.com/Spinners/Spin.Swift/workflows/Tests/badge.svg)
![](https://github.com/Spinners/Spin.Swift/workflows/SwiftLint/badge.svg)

<img alt="Spin Logo" src="https://raw.githubusercontent.com/Spinners/Spin.Swift/master/Resources/spin-logo.png" border="1"/>

**Spin** is a tool to build feedback loops within a Swift based application, allowing to use a unified syntax whatever the underlying reactive programming framework and whatever Apple UI technology you use (RxSwift, ReactiveSwift, Combine and UIKit, AppKit, SwiftUI).

# Introduction

**What is a state machine ?**

> It is an abstract machine that can be in exactly one of a finite number of states at any given time. The state machine can change from one state to another in response to some external inputs. The change from one state to another is called a transition. A state machine is defined by a list of its states, its initial state, and the conditions for each transition

Guess what ? An application IS a state machine.

We just have to find the right tool to implement it. This is where feedback loops come into play 👍.

A Feedback loop is a system that is able to self regulate by giving the resulting value from its computations as the next input to itself, constantly adjusting this value according to given rules (Feedback loops are used in domains like electronics to automatically adjust the level of a signal for instance).

<img alt="Feedback Loop" src="https://raw.githubusercontent.com/Spinners/Spin.Swift/master/Resources/feedback.png" border="1"/>

Stated like that, it might sound obscur and unrelated to software engineering BUT “adjusting a value according to certain rules” is exactly what a program and by extension an application is made for ! An application is the sum of all kinds of states that we want to regulate to provide a consistent behaviour following precise rules.

Feedback loops are perfect candidates to host and manage state machines inside an application.

# What is Spin ?

Spin is a tool which only purpose is to help you build feedback loops called « Spins ».
A Spin is based on three components. To illustrate each one of them we will rely on a basic example: a “feedback loop / Spin” that counts from 0 to 10.

* The initial state: this is the starting value of our counter, 0.
* A feedback: this is the rule we apply to the counter to accomplish our purpose. If 0 <= counter < 10 then we ask to increase the counter else we ask to stop it.
* A reducer: this is the state machine of our Spin. It describes all the possible transitions of our counter given its previous value and the request computed by the feedback. For instance: if the previous value was 0 and the request is to increase it, then the new value is 1, if the previous was 1 and the request is to increase it, then the new value is 2, and so on and so on. When the request from the feedback is to stop, then the previous value is returned as the new value.

<img alt="Feedback Loop" src="https://raw.githubusercontent.com/Spinners/Spin.Swift/master/Resources/feedback-loop.png" border="1"/>

Feedbacks are the only places where you can perform side effects (networking, local I/O, UI rendering, whatever you do that accesses or mutates a state outside the local scope of the loop)
Conversely, a reducer is a pure function that can only produce a new value given a previous one and a request. Performing side effects in reducers is forbidden, as it would compromise its reproducibility.

In real life applications, you can obviously have several feedbacks per Spin in order to separate preoccupations. Each of the feedbacks will be applied sequentially on the input value.

# The ways to build a Spin

Spin offers two ways to build a feedback loop. Both are equivalent and picking one only depend on your preference.

Let’s try them by building a Spin that regulates two integer values to make them converge to their average value (like some kind of system that would adjust a left and a right channel volume on stereo speakers to make them converge to the same level).

The following example will rely on RxSwift, but you can find their **ReactiveSwift** and **Combine** counterparts here, you will se how similar they are.

We will need a data type for our state:

```swift
struct Levels {
    let left: Int
    let right: Int
}
```

We will also need a data type to describe the transitions to perform on Levels:

```swift
enum Event {
    case increaseLeft
    case decreaseLeft 
    case increaseRight
    case decreaseRight
}
```

Now we can write the two feedbacks that will have an effect on each level:

```swift
func leftEffect(inputLevels: Levels) -> Observable<Event> {
    // this is the stop condition to our Spin
    guard inputLevels.left != inputLevels.right else { return .empty() }

    // this is the regulation for the left level
    if inputLevels.left < inputLevels.right {
        return .just(.increaseLeft)
    }  else {
        return .just(.decreaseLeft)
    }
}

func rightEffect(inputLevels: Levels) -> Observable<Event> {
    // this is the stop condition to our Spin
    guard inputLevels.left != inputLevels.right else { return .empty() }

    // this is the regulation for the right level
    if inputLevels.right < inputLevels.left {
        return .just(.increaseRight)
    }  else {
        return .just(.decreaseRight)
    }
}
```

And finally to describe the transitions of the state we need a reducer:

```swift
func levelsReducer(currentLevels: Levels, event: Event) -> Levels {

   guard currentLevels.left != currentLevels.right else { return currentLevels }

    switch event {
        case .decreaseLeft:
            return Levels(left: currentLevels.left-1, right: currentLevels.right)
        case .increaseLeft:
            return Levels(left: currentLevels.left+1, right: currentLevels.right)
        case .decreaseRight:
            return Levels(left: currentLevels.left, right: currentLevels.right-1)
        case .increaseRight:
            return Levels(left: currentLevels.left, right: currentLevels.right+1)
    }
}
```

## The builder way

In that case, the “**Spinner**” class is you entry point.

```swift
let levelsSpin = Spinner
	.from(initialState: Levels(left: 10, right: 20))
	.add(feedback: RxFeedback(effect: leftEffect))
	.add(feedback: RxFeedback(effect: rightEffect))
	.reduce(with: RxReducer(reducer: levelsReducer))
```

That’s it. The feedback loop is built. What now ?

If you want to start it, then you have to subscribe to the underlying reactive stream. To that end, a new operator “**.stream(from:)**” has been added to Observable in order to connect things together and provide an Observable you can subscribe to.

```swift
Observable<Levels>
	.stream(from: levelsSpin)
	.subscribe()
	.disposed(by: self.disposeBag)
```

## The declarative way

In that case we use a "DSL like" syntax thanks to function builder:

```swift
let levelsSpin = RxSpin(initialState: Levels(left: 10, right: 20),
                        reducer: RxReducer(reducer: levelsReducer)) {
    RxFeedback(effect: leftEffect)
    RxFeedback(effect: rightEffect)
}
```

The way to start it remains unchanged.

# The multiple ways to create a Feedback

As you saw, a “Feedback loop / Spin” is created from several feedbacks. A feedback is a wrapper structure around a side effect function.
Basically, a side effect has this signature (Stream<State>) -> Stream<Event>, Stream being a reactive stream (Observable, SignalProducer, AnyPublisher).

As it might not always be easy to directly manipulate Streams, Spin comes with a bunch of helper constructors for feedbacks allowing to:

* directly receive a State instead of a Stream<State>
* filter the input State by providing a predicate
* extract a substate from the State by providing a lens or a keypath

Please refer to [Feedback+Default.swift](https://github.com/Spinners/Spin.Swift/blob/master/Sources/Spin.Swift/Feedback%2BDefault.swift) for completeness.

# What about the feedback lifecycle ?

There are typical cases where a side effect consist in an asynchronous operation (like a network call). What happens if the very same side effect is called repeatedly ? Are the operations stacked ? Are they cancelled when a new one is performed ?

Well, it depends 😁. By default Spin will cancel the previous operation. But there is a way to override this behaviour. Every feedback constructor that takes a State as a parameter can also be passed an ExecutionStrategy:

* **.cancelOnNewEvent**, to cancel the previous operation
* **.continueOnNewEvent**, to let the previous operation end

Choose wisely the option that fits your needs. Not cancelling previous operations could lead to inconsistency in your state if the reducer is not protected against unordered events.

# What about the feedback scheduling ?

Reactive programming is often associated with asynchronous execution. Even though every reactive framework comes with its own GCD abstraction, it is always about saying on which scheduler should the side effect be executed. 

Spin provides a way to specify that scheduler for each feedback you add to a loop, still being as declarative as possible:

```swift
Spinner
	.from(initialState: Levels(left: 10, right: 20))
	.add(feedback: RxFeedback(effect: leftEffect, on: SerialDispatchQueueScheduler(qos: .userInitiated)))
	.add(feedback: RxFeedback(effect: rightEffect, on: SerialDispatchQueueScheduler(qos: .userInitiated)))
	.reduce(with: RxReducer(reducer: levelsReducer))
```
or

```swift
RxSpin(initialState: Levels(left: 10, right: 20),
       reducer: RxReducer(reducer: levelsReducer)) {
    		RxFeedback(effect: leftEffect)
        		.execute(on: SerialDispatchQueueScheduler(qos: .userInitiated))
    		RxFeedback(effect: rightEffect)
        		.execute(on: SerialDispatchQueueScheduler(qos: .userInitiated))
}
```

Of course, it remains possible to handle the Schedulers by yourself inside the feedback functions.

# How to use a Spin in a UIKit/AppKit based app ?

Although a feedback loop can exist by itself without any visualisation, it makes more sense in our developer world to use it as way to produce a State that we can render on screen and to handle events emitted by the users.

Fortunately, taking a State as an input for rendering and returning a stream of events from the user interactions looks A LOT like the definition of a feedback.

The view is a function of a State and rendering it changes the states of the UI elements, it is a mutation exceeding the local scope of the loop: we have a side effect.

Once a Spin is built, we can “decorate” it with a new feedback dedicated to the UI rendering/interactions. A special type of Spin exists to perform that decoration: RxUISpin, ReactiveUISpin, CombineUISpin depending on your framework.

As a global picture, we can illustrate a feedback loop in the context of a UI with this diagram:

<img alt="Feedback Loop" src="https://raw.githubusercontent.com/Spinners/Spin.Swift/master/Resources/uispin.png" border="1"/>

In a ViewController, let’s say you have a rendering function like:

```swift
func render(state: State) {
    switch state {
    case .increasing(let value):
        self.counterLabel.text = "\(value)"
        self.counterLabel.textColor = .green
    case .decreasing(let value):
        self.counterLabel.text = "\(value)"
        self.counterLabel.textColor = .red
    }
}
```

We need to decorate the “business” Spin with a UI Spin, instance variable of the ViewController so their lifecycle is bound:

```swift
// counterSpin is the Spin that handles our counter business
self.uiSpin = RxUISpin(spin: counterSpin)

// we now want to attach the UI Spin to the rendering function of the ViewController:
self.uiSpin.render(on: self, using: { $0.render(state:) })
```

And once the view is ready (in “viewDidLoad” function for instance) let’s start the loop:

```swift
self.uiSpin.spin()
// internally, this statement builds and subscribes to the reactive stream with the custom operator seen in the “The ways to build a Spin” chapter
```

To send events in the loop, this is very straightforward, simply use the emit function:

```swift
self.uiSpin.emit(Event.startCounter)
```

# How to use a Spin in a SwiftUI based app ?

Because SwiftUI relies on the idea of a binding between a State in a View and takes care of the rendering, the way to connect the UI Spin is slightly different, and even simpler.

In your view you have to annotate the UI Spin variable with “@ObservedObject” (a UISpin being an “ObservableObject”):

```swift
@ObservedObject
private var uiSpin: RxUISpin<State, Event> = {
    // counterSpin is the Spin that handles our counter business
    let spin = RxUISpin(spin: counterSpin)
    spin.spin()
    return spin
}()
```

you can then use the “uiSpin.state” property inside the view to display data and uiSpin.emit() to send events:

```swift
Button(action: {
    self.uiSpin.emit(Event.startCounter)
}) {
    Text("\(self.uiSpin.state.isCounterPaused ? "Start": "Stop")")
}
```

A UISpin can also be used to produce SwiftUI bindings:


```swift
Toggle(isOn: self.uiSpin.binding(for: \.isPaused, event: .toggle) {
    Text("toggle")
}
```

**\\.isPaused** is a keypath designating a sub state of the state, and **.toggle** is the event to emit when the toggle is changed.

# Demo applications

In the Spinners organization, You can find 2 demo applications demonstrating the usage of Spin with RxSwift, ReactiveSwift and Combine.

* A basic counter application: [UIKit version](https://github.com/Spinners/Spin.UIKit.Demo.Basic) and [SwiftUI version](https://github.com/Spinners/Spin.UIKit.Demo.Basic)
* A more advanced “network based” application using dependency injection and a coordinator pattern (UIKit): [UIKit version](https://github.com/Spinners/Spin.UIKit.Demo) and [SwiftUI version](https://github.com/Spinners/Spin.UIKit.Demo)

# Acknowledgements

The advanced demo applications use [Alamofire](https://github.com/Alamofire/Alamofire) for their network stack.

The advanced UIKit demo applications uses [Swinject](https://github.com/Swinject/Swinject) for dependency injection, [Reusable](https://github.com/AliSoftware/Reusable) for view instantiation and [RxFlow](https://github.com/RxSwiftCommunity/RxFlow) for the coordinator pattern.

The following repos have been a source of inspiration:

* [RxFeedback](https://github.com/NoTests/RxFeedback.swift)
* [ReactiveFeedback](https://github.com/babylonhealth/ReactiveFeedback)
* [CombineFeedback](https://github.com/sergdort/CombineFeedback)
