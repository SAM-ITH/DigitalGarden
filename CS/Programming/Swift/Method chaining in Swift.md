- this is regarding how the methods are call in swift using the dot notation. 
- my main problem is why so much dots are used in swiftUI. for example we can take the below code snippet.
```swift
button.setTitle("Click me", for: .normal)
```
- there are few main points to keep in your mind here. 
1. In swift methods are typically called using do notation. 
2. properties are also accessed using dot notation. 
```swift
let buttonTitle = button.title
```
3. swiftUI take this to crazy level by using method chaining and property wrappers. they have done all these to make code more readable and have a compact code structure. 
4. many of the dot-notations thats called in swiftUI are view modifiers. 
5. **parameter styles** : swiftUI often use trailing closure syntax and labeled parameters to make code more readable. some parameters are look like they called with dot notation. but they're actually enum cases or static properties. 

- lets decipher the code snippet that becomes a problem for me. 
```swift 
.contentTransition(.symbolEffect(.replace))
```
- `contentTransition` - is a view modifier method. 
- `.symbolEffect` - instance method 
- `.replace` - this is highly likely to be a static property or enum case of symbolEffect method