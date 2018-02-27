# CPU renderer
This is a 3D renderer running solely on the CPU. 

### But why would you do this
Usually when we want to render some pretty graphics we turn to our GPUs since a lot of smart people have spent a lot of time thinking about how to make them work optimally just for this purpose. That means using an API such as OpenGL or Direct3D (or Vulkan if you want to go hard-core). GPU vendors such as NVIDIA provide an implementation of these APIs in drivers which utilize the graphic card's processing power. The rasterization of primitives is handled behind the scene and the nitty-gritty details of the process are hidden from the API's user. It might then be interesting to replicate the process on the CPU where it is fully under our control.

### What the above entails
Even a basic implementation of a renderer needs to have the following:
* Logic for rasterizing primitives (triangles in our case) producing fragments
* Logic for interpolating vertex attributes over the produced fragments
* The ability to run shaders on vertices passed to the pipeline and on fragments generated by the rasterization
* Texture sampling logic
* Texture memory units
* Uniform memory units
* A frame buffer for storing pixels
* A depth buffer storing values for depth test
* Finally some way to actually display the frame buffer to the monitor for which we can use SDL

### What the especially tricky parts are
* Perspective correct interpolation
  * The interpolation of vertex attributes over fragments must be *perspective correct*. The rasterizer receives a triangle containing three vertices and each vertex contains attributes such as position, color or a normal vector. After rasterization the triangle may end up covering many (or few) pixels on the screen depending on the view angle and camera position relative to it. The values of the attributes must be interpolated for each fragment from the vertices. The resulting value of the fragment's attribute is a weighted sum of the values in the vertices based on the relative distance of the fragment from each vertex.
  * But since vertices undergo perspective projection the interpolation cannot be done naively but must take into account the depth (z-distance) from the camera of the vertices. Those that are further from the viewer have their weight reduced appropriately. The equations are described in detail in the [OpenGL spec](https://www.khronos.org/registry/OpenGL/specs/gl/glspec44.core.pdf). Figure from [scratchapixel](https://www.scratchapixel.com/lessons/3d-basic-rendering/rasterization-practical-implementation/perspective-correct-interpolation-vertex-attributes).
  
  ![](https://www.scratchapixel.com/images/upload/rasterization/persp-correct-interpo4.png?)
  
* Triangle clipping
  * Since we only want to render parts of the triangles which happen to be in the camera's view frustum (defined by the near plane, far plane and its field of view) we need to clip them against the six planes defined by the frustum. New vertices are produced in the process and in some cases more than one triangle. 
  
  ![Figure from scratchapixel](https://i.imgur.com/JqIoysL.png)
  
* Speeding up the rasterization loop
  * Each triangle is projected from the 3D space onto the 2D canvas(window) and we need to determine which pixels in the framebuffer are potentially affected by it. That means determining for each pixel whether it actually happens to be within the projected triangle's area. The [edge function](https://www.cs.drexel.edu/~david/Classes/Papers/comp175-06-pineda.pdf) (the original linked article is quite readable) is used for this and we can take advantage of its linearity so that we do not need to recompute it completely for each pixel but just slightly modify it for each movement in the x or y direction in pixel space. Additionally, we need to find out whether the passes the depth test (it is not occluded by a closer object). 
