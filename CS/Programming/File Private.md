## File Private 
- `fileprivate` this means those variables can be access from any class in the particular file.
- for an example lets says there are two classes in the same file. if i am trying to access the classA's variables in class that can be done by using `fileprivate` key word

``` swift
class ClassA {
    fileprivate var name = "samith"
}

class ClassB {
    func test(){
		var object = ClassA()
        object.name
    }
}
```