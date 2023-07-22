## Spacer
--
adding this property will add a space between the two elements. 
``` swift
Text("VStack Item 1")
Spacer()
Text("VStack Item 2")
```
- Adding the **Spacer()** forces the view to use the maximum amount of vertical space. This is because the **Spacer()** is the most flexible view—it fills the remaining space after all other views have been displayed.
- The **HStack** is like the **VStack** but its contents are displayed horizontally from left to right. Adding a **Spacer()** in an **HStack** thus causes it to fill all available horizontal space, and a divider draws a vertical line between components in the **HStack**.