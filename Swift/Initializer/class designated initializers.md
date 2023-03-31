### fixing errors with default values
- when we create a instance of class, no property can be left in a undetermined state. 
- for example following code snippet gives a error saying class samith has no initializer. 

``` swift
class samith {

var color : String

var age : Int

var hair : String?

var gf = "Saku"

}
```
- in here there are two properties that not determine.
- first one is color and the second one is age. 
- hair variable is a optional and gf variable has a value. 
- this error can be mitigate by creating a initializer for properties that do not have a default value or optional

#### what is an intializer?
- its an object that runs every time when you creating a new instance of object. 
- primary role of the initializer is to ensure that all stored properties will be assigned a value when instance is created. 

###   Create an Initializor for those properties that do not have a default value, or which are optional.


### more initializer concepts.
