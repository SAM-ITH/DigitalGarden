- this is the safest way to unwrap a value. 

``` swift
      
var parsedInt: Int? = 10

if let parsedInt = parsedInt {
 print(parsedInt)
}else{
 print("nil")
}
```

- this can also used to unwrap multiple values.

``` swift
var parsedInt: Int? = nil

var parsedString: String? = " "


if let parsedInt = parsedInt,

 let parsedString = parsedString{

 print(parsedInt)

 print(parsedString)

}else{

 print("nil")

}
```