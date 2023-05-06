## Call by Value
- this also known as copy-in copy-out.
- we use inout key word before mentioning the type of the parameter. 

``` swift
	func incrementAndPrint(_ value: inout Int){
	value += 1 
	print(value)
	}
```
- this inout keyword indicates that this parameter should be copied in,local copy used within the function and then copied back out when the function returns. 
- when you calling for the function that using this behavior you need to add ampersand(&) before the argument. 

``` swift
var value = 5
incrementAndPrint(&value)
print(value)
```