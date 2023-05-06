## View - thing with the Struct 

- this is a data structure that behaves like a view.
- view keyword is a behavioral element.

``` swift 
struct ContentView: View{
	var body: some view {
		Text("Hello Samith").padding(20)
	}
}
```
in here the text is a function and also adding padding is calling an another functions. so function always returns something. in here text is returning a text view. if we add padding modifier it will return a view that have a padding around it. whats really happening is SwiftUI taking the text view and return it with the modifier view that we have added. 

> swiftUI is in the functional programming paradigm. so functions are the kings in swiftUI.

- when working with the swiftUI think all the concepts as Lego set and Lego bricks.