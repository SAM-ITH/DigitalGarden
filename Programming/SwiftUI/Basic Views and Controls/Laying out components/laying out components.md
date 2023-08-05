## Laying out components 
- swiftUI use three basic layout components
	- VStack - arrange components in the vertical axis 
	- HStack - arrange components in the horizontal axis
	- ZStack - arrange components along the vertical and horizontal axis. 
	- body view only returns a single view.

---

## How the Stack view determine to show the display content
1.  Figure out its internal spacing and subtract that from the size proposed by its parent view.
2.  Divide the remaining space into equal parts.
3.  Process the size of its least flexible view.
4.  Divide the remaining unclaimed space by the unallocated space, and then repeat _Step 2_.
5.  The stack then aligns its content and chooses its own size to exactly enclose its children.

---
- [[Spacer]]
- [[Divider]]
- [[Offset]]
- [[Frame]]





