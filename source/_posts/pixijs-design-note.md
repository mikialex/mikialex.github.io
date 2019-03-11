---
title:  a glimpse of pixijs design
date: 2018/1014
---

Pixi.js is a fast opensource html5 2d render engine. My company's product use it for rendering part of the 2d floor plan . I have knew pixi for a long time, but never thought it could be used in many varity purpose.  As I am quite familiar with the world most popular web3d rendering library three.js, maybe I should dig in about how to render 2d content on web effectively by reading the design and implementation of pixi.js.

###  scene structure

Like Threejs's scene tree, to specify what you want to draw, we have same structure in pixi.  To draw an analogy,  in threejs, all things extends from object3d, it has a scene tree node which describe the scene hierarchy. We have same object in pixi called Container. Container extends from DisplayObject.  Some slight difference is DisplayObject only hold the parents reference not children.  Container is specially used for grouping children like three's group, it holds the children reference, and is actually as same duty as the Object3D;

Besides informations like visible, transformation, pixi has some features that 2d render prefered. You can set  alpha, custom filter and filter tectangle per displayobject. You can set the pivot of your displayobject to define the rotation center of your object easily. You can set a musk(any graphics or sprite) on a node to create mask effect. 

Two class derived from Container : Graphics and Sprites. There is a major difference in 3d and 2d rendering is that when we consider bitmap resource for example texture. Texture is something attached to geometry not a standalone render assets in 3D rendering, but In 2d Rendering, texture object is used quite often than any others, and its has it only behaviors. In general, in 2d rendering, we should distinguish our render object to vector graphics and non vector graphics, that is Graphics and Sprites class in pixijs.

##### Sprites

The Sprite object is the base for all textured objects that are rendered to the screen. It can update its texture from an image or from frame buffer.

##### Graphics

The most of traditional canvas2d API is about how to draw a vector shape on canvas. Graphics encapsue and extend same interface. Support additional powerful features like bezierCurve ellipse roundedRect that enables users to express shape with more freedom. The vector shape info stored in GraphicsData class will be used to draw by different render backends.

### render structure

The renderer of pixi is created in PIXI.Application. From it's API reference, it controls basic render configurations such as width, height, antialiasing, resolution, clear color and so on. Two kind of render backend is supported by two kind of renderer: PIXI webgGLRenderer and CanvasRenderer. CanvasRenderer is fallback for webgl not available device, and some features like FXAA, powerPreference can only supported by WebGLRenderer.The two kind of renderer is extends from SystemRenderer for sharing and managing general render options and configurations.

In pixijs, how to render is not implemented in renderer itself, but in the very root base class displayobject. In display object, there are two empty methods renderWebGL and renderCanvas should be implemented by its derived class.  In real implementation, render method is a plugin registered in renderer. Derived class check some state and decide use which plugin to draw it content, because maybe exist some cache or optimization logic like cachetobitmap make the render logic very very complicate.Sprites has it own SpritesCanvasRenderer, SpritesWebGLRenderer, as well as Graphics. These work as plugin in WebglRenderer.

 So, thats maybe a very good design decision in what I found so far. Imaging if you want to extend your graphics to render and maybe you need another render technic already exist, you just need write your only render logic and call the renderer's render plugin. It is a quite good practices in complex render system design.

##### GraphicsRenderer

As I mentioned before, graphics data is stored to support different backend.  webgl/GraphicsRenderer shows how this happened .  In Graphics class, there is a dirty state record wether the underlaying gl data needs update. Every time your change the graphics may reset the dirty flag. In render process, GraphicsRenderer check this flag, and rebuild gl data, rebind and reupload it to context. The corresponding gl resource is stored in Graphics' _webGL object property separated by different context. 

The canvas2d backend renderer is the very initial version of pixijs support. Simply maintain the canvas2d render state machine use simple canvas2d drawing api.

##### SpriteRenderer

SpriteRenderer webgl is not as simple as I thought. Pixi not only supports plain texture sprite, but also supports spine animation. Instead of creating a simple quad for render texture, It creates a full function index buffer seems to be this reason. Another noticeable optimization that is sprite drawing is batched, not when you make a render call it make a instant draw. The real render triggered at each flush call. Flush method is designed in base class ObjectRenderer, and maybe it actually designed for batching.

SpriteCanvas Renderer likes Graphics renderer, not have too much to say .Sprite texture tint feature is not supported by native browser API. Pixi use a CanvasTinter util class make it available. Generally, it use blend mode overlay and multiply or convert texture data directly.