Optional are a powerful source of safety in Swift, but can also be annoying if you find them littered throughout your code. Swift's nil coalescing operator helps you solve this problem by either unwrapping an optional if it has a value, or **providing a default if the optional is empty.

Here's an example to get you started:

```swift
let name: String? = nil
let unwrappedName = name ?? "Anonymous"
```

Because `name` is an optional string, we need to unwrap it safely to ensure it has a meaningful value. The nil coalescing operator – `??` – does exactly that, but if it finds the optional has no value then it uses a default instead. In this case, the default is "Anonymous". What this means is that `unwrappedName` has the data type `String` rather than `String?` because it can be guaranteed to have a value.

You don't need to create a separate variable to use nil coalescing. For example, this works fine too:

```swift
print("Hello, \(name ?? "Anonymous")!")
```
