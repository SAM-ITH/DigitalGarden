- one of the way is force unwrapping. this is not a safe way to unwrap because 
> you can not unwrap a nil value.

``` swift
var parsedInt: Int? = 10

print(parsedInt!)
```

- do not use this way unless you are sure that there aren't any nil values. 