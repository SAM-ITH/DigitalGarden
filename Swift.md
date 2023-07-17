#### Architecture of swift 
- Swift commands are statements 
- In swift everything is an object. in here definition of object is little bit different than others. if you can send a message to something then its becomes an object. for example we can send bark and sit messages to dog object. 
- in swift Syntax of  message sending look as follows   `tang.bark()` - tang is the object and then `.` after the dot there will be the message. 
- lets take a example to justify the statement that everything is an object in swift. think about a 1 in swift. this can be write as follows in swift. 
`let s = 1.description`  this is meaningful and legal way in swift. so i is a object and its got a message named as description. 
- In Swift, every noun is an object, and every verb(methods and functions) is a message. 
- Object type can be extend in swift. for example you can not normally say `1.sayHello` but this can be done by changing the number type as below.
```swift 
extension Int {
			   func sayhello {
			   print("Hello, I'm \(self)")
			   }
}

1.sayHello // output - "Hello, I'm 1"
```
- there are three kind of objects in swift 
	- [[Classes]]
	- [[Structs]]
	- [[ENUM]]

### Core concepts 
[[Functions]]
[[Closures]]
[[Protocols]]
[[Optionals]]
[[Control flow]]
[[Collection types]]

___
### Basic concepts


[[Complex types]]
[[Operators & conditions]]
[[Static Properties and Methods]]
[[typecasting]]


### Other 
[[Naming style guidelines]]



### Code Structure and readability and principle 
[[Code Structure]]
