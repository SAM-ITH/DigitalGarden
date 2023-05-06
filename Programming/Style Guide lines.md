[[Swift]]

- Variables, constants and function name use camelCase
```swift
var  name : String
let  tax = 7.8
func printMessage(){ }
```

- Data types use TitleCase
```swift

class CustomerOrder{ }

enum UserType{ }

protocol MyProtocol{ }

struct ApiResponse{ }

```
- when you are naming the functions and parameters of the function. name it in express able way. for an example,
```swift
func sum(_ a: Int, with b: Int) -> Int {
		return a+b
}
```
- so when you calling the function. its look like this
		sum(10, with: 5)
	- function naming says what that function is doing.