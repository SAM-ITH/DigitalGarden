- variadic functions means, accepting any number of parameters of the same type.
- we can make function variadic by writing ... after its type.

``` swift
func square(numbers: Int...) {
    for number in numbers {
        print("\(number) squared is \(number * number)")
    }
}
```

- inside the function, swift converts the values that were passed in to an array of integers, so we can loop over them as needed. 

``` swift
square(numbers: 1, 2, 3, 4, 5)
```