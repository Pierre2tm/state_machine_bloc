# BLoC State Machine

An extension to the bloc state management library to define state machines that support state's data using a nice declarative API.

🚧 This project is still early. Use it with caution! 🚧

## Overview

`state_machine_bloc` export a `StateMachine` class, a lightweight wrapper around bloc designed to declare state machines a nice builder API.
`StateMachine` _is_ a `Bloc`, meaning that you can use use it in the same way as a regular bloc and it's compatible with the rest of the ecosystem.

`state_machine_bloc` supports:

* [X] Storing data in states
* [X] Asynchronous state transitions
* [X] Applying guard conditions to transitions
* [X] state's onEnter/onChange/onExit sideEffect
* [X] Nested states without depth limit

State machines eliminates bugs and weird situations because they won't let the UI transition to a state which we don’t know about.
Its also eliminates the need for code to protects other codes from execution because StateMachine do not process events that are not explicitly defined as acceptable for the current state.

```dart
class Timer extends StateMachine<Event, State> {
    Timer() : super(Idle()) {

        define<Idle>((b) => b
            ..on<StartButtonPressed>(
                (StartButtonPressed event, Idle currentState) => Running(0),
            ),
            ..onExit(_startTicker)
        );

        //Started state handle reset event for both Running and Paused sub-states
        define<Started>((b) => b
            ..on<ResetButtonPressed>(
                (ResetButtonPressed event, Started currentState) => Idle(),
            )
            ..define<Running>((b) => b
                ..on<Ticked>(
                    (PauseButtonPressed event, Running state) =>
                        state.duration >= 60
                        ? Completed()
                        : Running(state.duration + 1),
                )
                ..on<PauseButtonPressed>(
                    (PauseButtonPressed event, Running state) => Paused(state.duration),
                )
                ..onEnter((Running state) => print(state.duration)) //0 and each timer is resumed
                ..onChange((Running old, Running next) => print(next.duration)) //1, 2, 3, ...
            )
            ..define<Paused>((b) => b
                ..onEnter(_pauseTicker)
                ..onExit(_resumeTicker)
                ..on<Resume>(
                    (Resume event, Paused state) => Running(state.duration),
                )
            )
            ..define<Completed>() //empty state
        );
    }
}
```

### Project Status

This project is still early. Use it with caution! Any opinions, feedback, thought and contributions are welcome.

### Goals

* [X] PoC
* [X] Make timer state machine example pass all original unit tests
* [X] nested states machines
* [ ] Unit tests
* [ ] Improve documentation
* [ ] Implements more bloc's examples
* [ ] 0.0.1 release

**features to be explored**

* [ ] dedicated flutter builder/listener widgets
* [ ] ~~make state machine usable with other library than bloc~~

### Usage

`StateMachine` has a narrow and user-friendly API.

See the example and test folders for additional examples.

Example of creating a StateMachine with two states:

```dart
// Base class for events handled by StateMachine
class Event {}
// Base class for StateMachine's State
class State {}

class Start extends Event {}
class Stop extends Event {}

class Idle extends State {}
class Run extends State {}

class MyStateMachine extends StateMachine<Event, State> {
    MyStateMachine() : super(Idle()) {
        define<Idle>((b) => b
            ..on<Start>(
                (Start event, Idle state) => Run(),
            ));
        define<Run>((b) => b
            ..on<Stop>(
                (Stop event, Run state) => Idle(),
            ));
    }
}
```
Use `StateMachine` like a regular bloc:

```dart
Future<void> main() async {
  /// Create a `MyStateMachine` instance.
  final stateMachine = MyStateMachine();

  /// Access the state of the `bloc` via `state`.
  print(stateMachine.state); // Idle object

  /// Interact with the `stateMachine` to trigger `state` changes.
  stateMachine.add(Start());

  /// Wait for next iteration of the event-loop
  /// to ensure event has been processed.
  await Future.delayed(Duration.zero);

  /// Access the new `state`.
  print(stateMachine.state); // Run object

  /// Close the `stateMachine` when it is no longer needed.
  await stateMachine.close();
}
```

with `flutter_bloc`:
```dart
BlocProvider(
    create: (_) => MyStateMachine(),
    child: ...,
);
```

### Defining States and Events

`StateMachine` expose `define<State>` method that should be used to define each state machine possible state and its transitions to other states. You can also register side effects to react to state lifecycle event.

Just like `Bloc`, a state machine defined as `StateMachine<Event, State>` should have each of its defined states a sub-type of `State` and each of its defined events a sub-type of `Event`.

You can call `define` as many time you want, but each defined state should have an unique type. An other rule is that the state machine should never be in a state that it's has not been defined. If this happen, `StateMachine` will throw an `InvalidState` error.

**`define` should only be used inside `StateMachine`'s constructor**
**Don't use `on<Event>` inside `StateMachine`. `StateMachine` takes care of calling `on<Event>` under the hood for you.**

```dart
 define<State>((b) => b
    ..onEnter((State state) { /* Side effect */ }) 
    ..on<Event>((Event event, State state) => NextState()) //transition to NextState
```

### Transitions
```dart
//inside MyStateMachine's constructor
define<State>((b) => b
    ..on<ButtonPressed>( //create new transition
        (ButtonPressed event, State state) => state.enabled ? NextState() : null, //return null to prevent transition
    )
    ..on<DataReceived>( //transition are evaluated sequentially
        (DataReceived event, State state) => OtherState(),
    )
    ..on<DataReceived>( //you can have as many transition you want, even of the same Event type
        (DataReceived event, State state) => OtherState(),
    )
    ..on<OtherEvent>( // transitions can be async
        (OtherEvent event, State state) async => await nextState(),
    )

define<NextState>();
define<OtherState>();
```
Transitions are evaluated sequentially. Transition are evaluated in the same order that they are defined. If a transition is `async`, it will be awaited before evaluating next one.

A transition could return a newState or null to indicate that the transition is refused. If null is returned, next transition is evaluated.
If all transitions return null, current state remain unchanged and no side effects are triggered.

**If a new state is returned from a transition where `newState == state`, the new state will be ignored**. If you're using state containing data, make sure you've implemented `==` operator. You could use `freezed` or `equatable` package for this purpose.

### Side Effects
```dart
define<State>((b) => b
    ..onEnter((State state) { /* called when entering State */ })
    ..onChange((State current, State next) { /* called when State data changed */ })
    ..onExit((State state) { /* called when exiting State */ })
```

**onEnter** is called when State Machine enter a state and `PrevState is! State`. If `State` is the initial `StateMachine`'s state, onEnter is called at initialization.
**onChange** is called when State Machine's current state data changed and `PrevState is State`. It's **not** called when the state machine enter the state for the first time.
**onExit** is called before State Machine's exit a state.

You can give async function as parameter for side effects, but remember they will **not** be awaited.
```dart
define<State>((b) => b
    ..onEnter((State state) async { /* not awaited */ })
```

### Nested State
```dart
define<State>((b) => b
    ..define<NestedState>((b) => b //create nested state
        ..onEnter( /* ... */ )
        /* ... */
    )
```

## Additional ressources

* [You are managing state? Think twice.](https://krasimirtsonev.com/blog/article/managing-state-in-javascript-with-state-machines-stent)
* [The rise of state machine](https://www.smashingmagazine.com/2018/01/rise-state-machines/)
* [Robust React User Interfaces with Finite State Machines](https://css-tricks.com/robust-react-user-interfaces-with-finite-state-machines/)
