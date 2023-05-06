### layers of ARKit
- ARKit can be divided into 3 layers. these layers are working together simultaneously. 
	- Tracking 
	- Scene understanding 
	- Rendering 

#### Tracking 
Tracking is the key function of ARKit. it allows us to track a device's position, location and orientation in the real world. 

#### Scene Understanding 
Understanding the scene means that ARKit analyses the environment presented by the camera’s view, then adjust the scene or provide information on it. This is what enables the detection of all the surfaces in the physical world such as the floor or a flat surface. Then, it will allow us to place a virtual object on it. Also, the light estimation can be integrated to lit a virtual object simulating a light source in the physical world. 

#### Rendering 
ARKit uses technologies to handle the processing of the 3D models and present them in our scene. such as 
- Metal 
- SceneKit 
- Third party tools like Unity of Unreal Engine 

