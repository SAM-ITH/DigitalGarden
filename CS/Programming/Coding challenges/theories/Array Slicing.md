- array slicing involves taking subset from an array and allocating it to a new array. 

````swift
let myArray = ["samith","saku","Medha","hasith"]

let slicer : ArraySlice<String>
slicer = myArray[0 ..< 3]
print(slicer)
````

- cut if the array is more than certain amount of elements 

````swift
let first3Elements : [String] // An Array of up to the first 3 elements.
if mentions.count >= 3 {
    first3Elements = Array(mentions[0 ..< 3])
} else {
    first3Elements = mentions
}
````