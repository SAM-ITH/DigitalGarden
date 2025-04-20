- what does it means by static 
static means there will be only one copy of each static variables and methods. simply saying they are the originals no copies of them. 

- how it's going to work on swift 
swift let us create properties and methods that belongs to a type, rather than instance of a type. 

we can create a one using `static` keyword. once it created you can access the property by using the full name of type. check the below example 

```swift 
struct TaylorFan {
    static var favoriteSong = "Look What You Made Me Do"

    var name: String
    var age: Int
}

let fan = TaylorFan(name: "James", age: 25)
print(TaylorFan.favoriteSong)
```

- in here Taylor swifts fans has a name and age. but all of them have the same favorite song.


reference - https://www.hackingwithswift.com/read/0/18/static-properties-and-methods 