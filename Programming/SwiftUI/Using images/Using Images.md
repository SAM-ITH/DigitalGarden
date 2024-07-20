## Adding image to a view.

- adding image view is done by using Image key word.
``` swift
Image("name of the image goes here")
```
- SwiftUI will add this image to original proportions.
- this can be fix by using 
``` swift
.resizable()
```
- modifier, this modifier cause the image to fit within its view.
- to keep the aspect ratio following modifier can be used.
``` swift
.aspectRatio(contentMode: .fit)
```
- for specifying width and height of the image.
``` swift
.frame(width: 200, height: 200)
```

for details about other modifiers please look into project file and https://developer.apple.com/documentation/swiftui/image%0D