## SKNode class 
- this class doesn't render any visual content.
- but its consider as the building block of SpriteKit because every node in a spriteKit scene is a subclass of SKNode.
- each subclass has a specific function or purpose. 
- To draw content, you need to use a visual node
	-   SKSpriteNode: Perhaps the most widely used type, this node draws a rectangle texture, image, or color.
    
	-   SKShapeNode: This type of node is used along with a Core Graphics path to draw custom shapes.
    
	-   SKLabelNode: When you need text, this type of node is used to draw a text label.
    
	-   SKVideoNode: With this type of node, you can display video content.
    
	-   SKReferenceNode: Although technically not considered a visual node, this is a special node in which you can create reusable content.

- these nodes are used to modify draw behavior 
	-   SKEffectNode: This type of node is used for caching or for applying Core Image filters for special effects.
    
	-  SKCropNode: When you need to mask pixels, you can use this type of node.

Although the SKNode class does not allow for visual content, it does provide some standard properties that its subclasses inherit. These properties include:

Position—

-   frame: The rectangle within the parent’s coordinate system.
-   position: The position within the parent’s coordinate system.
-   zPosition: The depth within the scene relative to its parent. Using this setting, you can have content overlap in a specific order.

Scale and Rotation—

-   xScale: The scaling factor of the x-axis (width).
-   yScale: The scaling factor of the y-axis (height).
-   zRotation: The Euler rotation around the z-axis, specified in radians[[21]](https://learning.oreilly.com/library/view/apple-game-frameworks/9781680508543/f_0031.xhtml#FOOTNOTE-21) (a standard unit for measuring angles).