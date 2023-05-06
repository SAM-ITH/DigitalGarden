## How it works 
- binary search is always done on a sorted array.
- first check if the value is in the scope. if not we can be sure searching value is not in the array.
- next check the middle value of the array to find if the value we need is greater than the middle value or not. if its greater that that we can drop the first half of the array.wise versa same thing is happened if less greater than the middle value.


## How to code this thing.

- first we will need setup variables for the index values that we need. 
- remember arrays should be always sorted.
``` swift
 	let minIndex = 0

    let maxIndex = array.count - 1

    let midIndex = maxIndex/2

    let midValue = array[midIndex]
```
- check if the value is in the array by checking max and min index.
``` swift
 	  if key < array[minIndex] || key > array[maxIndex] {
        print("\(key) value is not in the scope")
        return false
    }
```
- checking if the value we need to find is greater than the mid value.
``` swift
 	 if key > midValue {

        let slice = Array(array[midIndex + 1...maxIndex])

        return binarySearch(array: slice, key: key)

    }
```
- now check if the value is less than the mid value.

``` swift
 	  if key < array[minIndex] || key > array[maxIndex] {
        print("\(key) value is not in the scope")
        return false
    }
```
- above steps are continue until we find the value that we are searching for.
``` swift
 	   if key == midValue{
        return true
    }
```