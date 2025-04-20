
## Frame
You can also use the **.frame** modifier to adjust the width and height of a component. Try deleting the **Spacer()** and **Divider()** from the **HStack** and then apply the following modifier to the **HStack**:
``` swift
        .frame(
                  maxWidth: .infinity,
                  maxHeight: .infinity,
                  alignment: .topLeading
                )
```