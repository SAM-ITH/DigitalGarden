## Divider 
this will add a divider between the two elements.
``` swift
Text("VStack Item 1")
Divider()
Text("VStack Item 2")
```
- The **Divider()** component is used to draw a horizontal line across the width of its parent view. That is why adding a **Divider()** view stretched the **VStack** background from just around the **Text** views to the entire width of the **VStack.** By default, the divider line does not have a color. To set the divider color, we add the **.background(Color.black)** modifier. Modifiers are methods that can be applied to a view to return a new view. In other words, it applies changes to a view. Examples include **.background(Color. black)**, **.padding()**, and **.offsets(…)**.