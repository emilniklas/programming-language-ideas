So I've been thinking a lot about language design. I'm not experienced enough to produce an
entire programming language, but I might be able to do a transpiler. Before that, though,
I need a place to scribble my thoughts down. So this repo is just that. Scribbles.

---

## Key concepts – Mutability and immutability
1. Only values within block scope can be declared as mutable.
2. Mutation are only valid within the scope. Nested if need be, but never outside scope.
3. To mutate instance members, method must be declared as mutation

```
myFunction() {
  let x = 3
  // x = 4 <-- Error, cannot mutate
}

class MyClass {
  let x = 3
  // var y <-- Error, only block scope values can be declared mutable

  myMethod() {
    // x = 4 <-- Error, cannot mutate
    var y = 2
    y = 3 // Fine, since y is var
  }

  mutation myMutation() {
    x = 4 // valid, because in mutating state, constants are mutable too
  }
}
```

Mutations cannot have side effects in immutable state, therefore always returns new instance.

```
let myInstance = MyClass()

let myInstanceWithXHaving4 = myInstance.myMutation()
```

This means that mutations cannot have a return type, they always return a new instance of the class it's in.

### Nested mutation
If a class contains a property with another class, calling mutations on that inner class behaves like a `void` method.

```
class A {
  let x = false

  mutation mutate() {
    x = true
  }
}

class B {
  let a: A

  init(this.a)

  mutation mutate() {
    a.mutate() // inferred: a = a.mutate()
  }
}

let defaultA = A()
let defaultB = B(defaultA)
print(defaultA.x) // false
print(defaultB.a.x) // false

let mutatedB = defaultB.mutate()
print(mutatedB.a.x) // true
```

## External operations
One exception to the rule is external operations. They can be called from the immutable context, without the need for
monads or other "pure" ways of handling IO.

An obvious example is `print`. The print function obviously has side effects, but it would be impractical to force print
to only be available in a mutable context. Declaring such a function (a function with no return value and no mutation) should
be marked with return value `Null`.

```
let printFunction: ...Any -> Null = print
```

# Type signatures and type expressions
* A type should, by convention, be cased with UpperCamelCase: `String`
* Type arguments are supplied within angle brackets: `Map<String, Int>`
* Function signatures are designated with an arrow: `String -> Int`
* Tuples are designated with parentheses: `(String, Int)`
* Parameter lists are tuples on left side of arrow: `(String, Int) -> (String, Int)`
* Optional types are designated with a question mark: `String?`
* Parameter lists can have named parameters: `(Int, named: String)`
* Rest parameters are designated with three dots: `...Int -> Int`
* Example – Higher order function with a named parameter taking function which returns a String, returning a
  Map of String and a tuple of optional Int and Bool: `(factory: () -> String) -> Map<String, (Int?, Bool)>`
* Functions as return values should not be wrapped in parens. Example – higher order function taking a string,
  returning a function which takes a string and returs a string: `String -> String -> String`
* While calling a function, optional types in tail position can be omitted.
```
myFunction(arg1: String, arg2: Int, arg3: Bool?) { ... }

myFunction("string", 123)
```
* While declaring a function, named arguments are designated with curly braces:
```
myFunction(arg: String, {named: Int}) { ... }

myFunction("string", named: 123)
```
* While declaring a function, omitted types are inferred, but defaults to `Any` if check fails.
```
takesString(arg: String) { ... }
takesWhat(arg) {
  takesString(arg) // arg is inferred as String
}
takesWhat(3) // Error: arg is String

takesAny(arg) {
  print(arg) // print takes ...Any, so arg is inferred to Any
}
takesAny(123)
takesAny("string")
```
* Within a function, a rest parameter is accessed as a List of the type in the signature:
```
takesManyStrings(strings: ...String) {
  print(strings.type) // List<String>
}
```
* Generics work as expected, however using one letter generic types is discouraged:
```
class MyClass<Item> {
  let item: Item

  convert<Converted>(conversion: Item -> Converted) -> MyClass<Converted> {
    return MyClass(conversion(item)) // MyClass<Converted> is inferred
  }
}
```
* Mutations can also change the type arguments of the class
```
class A<T> {
  let t: T

  init(this.t)

  mutation<O> mutate<O>(convert: T -> O) {
    t = convert(t)
  }
}

let a = A<String>("1") // a is inferred to have type A<String>
let aOfInt = a.mutate<Int>(s => s == "1" ? 1 : 0)
print(a.t) // 1
```

# Encapsulation
As a default, identifiers are public within the scope. To make something private, prefix the identifier with an underscore.
```
class PublicClass {
  _privateMethod() { ... }
  mutation _privateMutation() { ... }
}
```

# Modules
Classes and global functions must be explicitly exported.
```
export class A { ... } // short form

export f() { ... }

let g = "v"
export g // explicit
```

Importing from another file can be explicit with what identifiers are imported, or it can import all of them.
Optionally under a name:
```
import f, A from "path/to/my_module" // f and A are available
import "path/to/my_module" // f, g, A, and B are available
import A, B from "path/to/my_module" as my_module // my_module.A and my_module.B are available
import without g from "path/to/my_module" // A, B, and f are available
```

Identifiers can be piped through by exporting identifiers from other modules:
```
export "path/to/some_module" // export everything
export A from "path/to/my_module" // export only A
```

## Handling world state
Since it isn't allowed to have global variables, state must be passed around. A good pattern is to
inject dependencies in the constructor of a class. Mutations make it easy to mimic global state.

```
class World {
  let population: Int
  let isDestroyed = false

  init({this.population})

  mutation destroy() {
    population = 0
    isDestroyed = true
  }

  mutation kill({amount: Int}) {
    population -= amount
    if (population == 0) {
      isDestroyed = true
    }
  }
}

class EvilEmperor {
  destroy(world: World) -> World {
    return world.destroy()
  }

  killPeasants(world: World) -> World {
    return world.kill(amount: 15)
  }
}

class Scenario {
  let world: World
  let evilOne = EvilEmperor()
  let evilTwo = EvilEmperor()

  init(this.world)

  mutation playOut() {
    print("Everything is fine")
    reportWorldState()

    print("First Emperor kills peasants")
    world = evilOne.killPeasants(world)
    reportWorldState()

    print("Second Emperor destroys world")
    world = evilTwo.destroy(world)
    reportWorldState()
  }

  reportWorldState() -> Null {
    print("Population: {world.population}")
    print("Is the world destroyed? {world.isDestroyed ? "Yes" : "No"}")
    print() // empty line
  }
}

let scenario = Scenario(World(population: 30))

scenario.playOut()
// Everything is fine
// Population: 30
// Is the world destroyed? No
//
// First Emperor kills peasants
// Population: 15
// Is the world destroyed? No
//
// Second Emperor destroys world
// Population: 0
// Is the world destroyed? Yes
//
```

## The four function modifiers
Each function has a return type, even if it is `Any`, `Null`, or inferred. But each function also has a modifier
to declare whether it is an iterative generator, and whether it is asynchronous. This modifier is prepended the return
type, which actually modifies the signature of the actual function.

|                   | sync         | async          |
|-------------------|--------------|----------------|
| **Single**        | `Type`       | `Future<Type>` |
| **Multiple (\*)** | `List<Type>` | `Stream<Type>` |

```
single() sync -> String { ... } // superfluous, sync is default
list() sync* -> String { ... }
future() async -> String { ... }
stream() async* -> String { ... }

print(single.type) // () -> String
print(list.type) // () -> List<String>
print(future.type) // () -> Future<String>
print(stream.type) // () -> Stream<String>
```

The modifiers also dictate what keywords are available in the function.

|          | ` `     | `await`  |
|----------|---------|----------|
| `return` | `sync`  | `async`  |
| `yield`  | `sync*` | `async*` |

* A `sync` function can `return` a value
* A `sync*` function can `yield` values
* An `async` function can `return` a value and `await` values
* An `async*` function can `yield` values and `await` values

# Inheritance & Polymorphism
Multi-inheritance is allowed, but to get around problems that commonly rise with multiple superclasses, classes
which have common instance member names are simple disallowed for extension together.

Each superclass must be initialized within the constructor of the subclass.

```
class Size {
  let height: Int
  let width: Int

  init(this.width, this.height)
}

class Shape {
  let size: Size

  init(this.size)
}

class Colored {
  let color: String

  init(this.color)

  mutation paint(color: String) {
    this.color = color
  }
}

class Box extends Colored, Shape {
  init(size: Size, color: String) {
    super<Colored>(color)
    super<Shape>(size)
  }

  description: String {
    return "A {size.width} by {size.height}, {color} box"
  }
}

let box = Box(Size(10, 10), "green")

print(box.description) // A 10 by 10, green box
```

There are no `interface` or `protocol` keywords. Every class can be implemented, by its public API.

```
class FakeBox implements Box {
  color: String => "transparent"
  size: Size => Size(0, 0)
  description: String => "A fake box"

  mutation paint(color: String) { ... }
}
```
