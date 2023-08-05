## Dealing with texts 

[[Labels]]
[[Secure Field]]
[[Getting user input]]

### modifying the font weight 
``` swift
Text("hey samith").fontWeight(.medium)
```
### changing the background color 
``` swift
.foregroundColor(Color.blue)
```

### changing space between individual letters or characters
``` swift
Text("Use kerning to change space between lines
           of text").kerning(7)
```

### Other modifications that can be done for the text views 
``` swift 
 Text("Changing baseline offset")
          .baselineOffset(100)

      Text("Strikethrough")
           .strikethrough()

      Text("This is a multiline text implemented in
           SwiftUI. The trailing modifier was added 
           to the text. This text also implements
           multiple modifiers")
            .background(Color.yellow)
            .multilineTextAlignment(.trailing)
            .lineSpacing(10)
```


