first we will talk about the code structure in a single swift file. 

- **Classes, Protocols and Extensions**  these guys are not surround by anything so they are sit in their own. 
- structs and enums are always in the top of the file 
- constant and variables should declared at the lowest level possible. this means if you only need variable inside if statement they only declare it inside the if statement. 

below is an example how you should layout your code inside a view controller 

```swift

//MARK: My Global and File Constants

struct GlobalThings{

    public static var myGlobalcont: String = "hello sonny"

}

//MARK: - Enums

enum myEnum { }

enum myOtherEnum {}

//MARK: - Protocols

protocol myProtocol {}

protocol myOtherProtocol {}

//MARK: - Home Details

struct myHouse {

// myHouse properties and functions for later use

}

//MARK: - View Controller

class ViewController: UIViewController {

//MARK: - Properties

@IBOutlet weak var myLabel: UILabel!

let myClassConstant = "Default Text"

let myReasoningForTheSpaces = "Spaces help separate variables logically by use, if I need to change an initial value I know where to look"

var myClassVariable: String = "I declare constants before variables, but keep them grouped together"

var session: URLSession = URLSession.default

//MARK: - View Life Cycle

override func viewDidLoad() {

// set up the session variable here

}

override func viewWillAppear() {

// start drawing objects needed for view here

// also set up animations that should load with the view

}

//MARK: - Helper Methods

func getUserInfo() {

...

}

//MARK: - Private Methods

private func updateInfo() {

myClassVariable = "I use private methods to perform work

that would only be used by the class it is declared in"

}

//MARK: Actions

@IBAction func buttonPressed(_ sender: UIButton) {

// handle the event initiated by the user

}

}

//MARK: - Extensions

//MARK: URLSession

extension ViewController: URLSessionDataDelegate {

// ... URLSession Data Delegate Methods

}

extension ViewController: URLSessionTaskDelegate {

// ... URLSession Task Delegate methods

}

//MARK: - View Controller Protocols

extension ViewController: myProtocol {

// add default implementation specific to ViewController

}

extension ViewController: myOtherProtocol {

// add default implementation specific to ViewController

}

//MARK:- Protocols

extension myProtocol {

//add default implementation where used

}

extension myOtherProtocol {

//add default implementation where used

}

```

### Breaking things up 
- view controller files will start to getting pretty long when you have so much functionality written it to it. 
- so we can start to separate them into files that in the same group folder. you can also use the naming convention as follows. `ViewController+NameOfDelegate.swift` or `ViewController+NameOfDataSource.swift`.
- for example i can declare all the global enums into a one file and use MARK: word to separate them into sections. if the enums are use in the same view controller group we can keep the enum file under the same group. 
- also its better to move protocols and their extensions to their own separate file unless the protocol define inside the model 
- structs can also rip out from the main file and put them under a group models
- class and all of its methods should be in the same file.
- constants are special. what we can do for this is put all the constant the use through out the project into one single file. so if there is a change we don't need to change it every place. only change in the constant file will do the trick. 

below is a example regarding a constant file 

```swift 
//Contants.swift

// This file contains all constants used by my app

struct SomeWebAPI {

static let myURL: String = "http://api.contoso.com"

}

// Call it later like this

let url = URL(string: SomeWebAPI.myURL)!

// or safely using

if let url = URL(string: SomeWebAPI.myURL) {

// do stuff with URL

}
```
more details regarding the constant file : https://stackoverflow.com/questions/26252233/global-constants-file-in-swift 

> always use descriptive variable names 

