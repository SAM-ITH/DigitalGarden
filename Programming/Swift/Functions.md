
## Functions in Swift 

functions are consider as a core part of many programming languages. if we have a simple explanation about what is function, it lets you define block of code that perform a some kind of task.when ever your app need to execute a task you can call the function and run it other than copy and paste the same code block over and over again. 

In here we will see how the swift will use functions and how it make easier to use it. 

### Basics of functions 
lets say you want to greet the user when he open your app.you can write a function to doing that.

``` swift
func greetings(){
print("Wellcome to app")
}
```
above code is known as a function declaration. we define a function using a **func** keyword. after that comes name of the function.followed by the parentheses. all the inputs that going inside the function is come between these parentheses. after parentheses comes an opening braces followed by the code you want to run in the function, followed by a closing bracket. 

you can call you functions as follows.
``` swift 
greetings()
```
this will print out the following.
``` swift
Wellcome to the app
```

### Function parameters
In the previous example we see function simply prints out a message. but sometime we need parameterize our function.that function perform differently according to the passing data via it's **parameters**.

let's consider the following example.
``` swift
func printthewellcome(name: String){
print("Hello, \(name)")
}

printwellcome(name: "John")
```

- In here you can see definition of one parameter inside the parentheses after function name. parameter named as "name" and value type is a string. 
- In any function, the parentheses contains what's known as a parameter list.
- you need to add parentheses even the parameter list is empty to invoke the function.  
- in this example we pass the name "John" to the function. so you can say function with an **argument** "john".
> Don't confuse the terms **parameter** and **Argument**. a function declare its parameters in its parameter list. when you call a function, you provide values as arguments for the functions parameters.

Also you can add multiple parameters feel the function like more general

``` swift 
func printAdditionOf(firstNo: Int, andSecondNo: Int){
print("\(firstNo) + \(secondNo) = \(firstNo + andSecondNo)")
}

printAdditionOf(firstNo: 4, andSecondNo: 5)
```
- in swift you should try to make your functions calls and read like a sentence. in previous example you can read the last code line like this 

`print addition of firstNo 4 and secondNo 5` 

we can make this even better by giving a parameter a different external name. in here we can change the name of the **andSecondNo** parameter:

``` swift 
func printAdditionOf(firstNo: Int, and SecondNo: Int){
print("\(firstNo) + \(secondNo) = \(firstNo + andSecondNo)")
}

printAdditionOf(firstNo: 4, and: 5)
```
- you can assign different external name by writing in front of the parameter name.


### Advance parameter handling

- functions parameters are constant by default. that means they can not be modified. 

let's take below code snippet as an example.

``` swift
func incerementAndPrint(_ value: Int){
    value += 1
	print(value)
	}
```
it's gives us the error saying `Left side of mutating operator isn't mutable: 'value' is a 'let' constant` 

this explain the previous point. that the all the parameters are constant by default.

the important fact that we have to remember here is to swift follows the **[[pass by value]]** behavior.

- pass by value means swift copies the value before passing it to the function. we need this type of behaviour in swift that functions doesn't alter it parameters. if it did then you can not be sure about the parameter values and you can make incorrect assumptions about your code.leading to the wrong idea.

sometimes we need to let functions to change their parameter directly. this can be done by using the behavior known as 	[[call by value result]]. let see this concept with an example.

``` swift
	func incrementAndPrint(_ value: inout Int){
	value += 1 
	print(value)
	}
```
this inout keyword indicates that this parameter should be copied in,local copy used within the function and then copied back out when the function returns. 
- when you calling for the function that using this behavior you need to add ampersand(&) before the argument. 

``` swift
var value = 5
incrementAndPrint(&value)
print(value)
```
now the function can change the value however its want.

### Overloading 
- we can use the same function name for several different functions. this techniques in called as overloading 

however complier should able to identify the each function separately. this can be achieved by using following,

- different number of parameters 
```swift 
funcPrint(multipler:Int, value:Int)
funcPrint(multipler:Int, value:Int, amount:Double)
```
- different parameter types 
```swift 
funcPrint(multipler:Int, value:Int)
funcPrint(multipler:Double, value:Double)
```
- different external parameter names 

- changing the return type of the function 
```swift 
func getValue() -> String {
	return "samith"					   
}

func getValue() -> Int {
	return 13 
}
```

but we are facing an issue when we trying to call function. because complier don't know which function is needed to be called. 
```swift 
let value = getValue()
```
this can be fixed by calling the function with the value type you want 
```swift 
let valueInt : Int = getValue()
let valueString: String = getValue()
```

### Functions as variables 
