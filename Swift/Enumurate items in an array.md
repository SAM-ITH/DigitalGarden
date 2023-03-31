- method to looping through an array.
- this method it iterates over each of the items while also telling you the items's position in the array.

```swift
let array = ["Apples", "Peaches", "Plums"]

for (index, item) in array.enumerated() {
    print("Found \(item) at position \(index)")
}
```
