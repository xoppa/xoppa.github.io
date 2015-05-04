---

layout: blogpost

comments: true
title: Behind the 3D scenes - part1
tags: libGDX 3D Graphics
author: Xoppa

---

In the [previous tutorial]({% post_url 2013-05-29-loading-a-scene-with-libgdx %}) we've seen how to load a 3D scene using LibGDX. Now we're going to take a quick look at what is actually going on behind the scenes. You don't have to know this information to actually use the 3D api and if you don't care how things work, you can safely ignore this. But I think it is good practice to know what is going on and how you can benefit from that. Therefor, this tutorial will cover the very basics, which I think you should know when working with the 3D api.
<more />

This is a two part tutorial. The first part will cover the g3db/g3dj file format, the Model class and how this affects creating your scene in the modeling application. The second part will cover the rendering pipeline, from loading the Model up to the actual rendering using a Shader.

We'll start with fbx-conv. Make sure to download the latest version of <a href="https://github.com/libgdx/fbx-conv" title="fbx-conv" target="_blank">fbx-conv</a> and let's convert the invaderscene.fbx file we created earlier. But instead of using the default binary format, we add a command line option to get the easier to read textual json format:

```bash
fbx-conv -o G3DJ invaderscene.fbx
```

This will produce a file called invaderscene.g3dj. Open the file using your favorite plain text editor (notepad will do). Don't be overwhelmed by all those numbers, and look at the overall structure:

```json
{
	"version": [  0,   1], 
	"id": "", 
	"meshes": [
...
	], 
	"materials": [
...
	], 
	"nodes": [
...
	], 
	"animations": []
}
```

I assume you are familiar with the json file format. Here we see that the file contains an object with six members. The first one being <em>version</em>, so the model loader knows what to expect. The second one is the <em>id</em> (the name some modeling applications allow you to specify), we will not use that for now. Then four arrays <em>meshes</em>, <em>materials</em>, <em>nodes</em> and <em>animations</em>. Now if you take a quick look at the <a href="https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/g3d/Model.java" title="Model.java">LibGDX Model class</a> we used in the previous tutorials, you'll see this at the top of the class:

```java
public class Model implements Disposable {
	public final Array<Mesh> meshes = new Array<Mesh>();
	public final Array<MeshPart> meshParts = new Array<MeshPart>();
	public final Array<Material> materials = new Array<Material>();
	public final Array<Node> nodes = new Array<Node>();
	public final Array<Animation> animations = new Array<Animation>();
	...
}
```

The Model class has also a meshes, materials, nodes and animations array. It also has a meshParts array, which we will cover soon. But for now we can say that the g3dj (and g3db) file is an one on one representation of what the model will actually contain. In fact that's the whole purpose of fbx-conv, it converts a fbx file to a runtime format ready to be rendered. This also means that by inspecting the g3dj file, we're also inspecting what the model class represents. This is extremely useful for debugging and testing. Take a look at the meshes array in the g3dj file. Again don't be overwhelmed by all those numbers, we just look at the overall structure:

```json
{
	"version": [  0,   1], 
	"id": "", 
	"meshes": [
		{
			"attributes": ["POSITION", "NORMAL", "TEXCOORD0"], 
			"vertices": [
				 25.000017, -95.105652, -18.163574, -0.269870,  0.942723,  0.196072,  0.050000,  0.900000, 
				 ...
			], 
			"parts": [
				{
					"id": "mpart1", 
					"type": "TRIANGLES", 
					"indices": [
						  0,   1,   2,   1,   0,   3,   3,   4,   5,   4,   3,   0, 
						 ...
					]
				}, 
				{
					"id": "mpart2", 
					"type": "TRIANGLES", 
					"indices": [
						 ...
					]
				}, 
				{
					"id": "mpart4", 
					"type": "TRIANGLES", 
					"indices": [
						 ...
					]
				}
			]
		}, 
		{
			"attributes": ["POSITION", "NORMAL"], 
			"vertices": [
				...
			], 
			"parts": [
				{
					"id": "mpart3", 
					"type": "TRIANGLES", 
					"indices": [
						 ...
					]
				}
			]
		}
	], 
	"materials": [
		...
	], 
	"nodes": [
		...
	], 
	"animations": []
}
```

Here we see that the meshes array contains two items (two meshes). Each item contains three arrays: <em>attributes</em>, <em>vertices</em> and <em>parts</em>. The attributes array specifies which vertex attributes the mesh contains. If you've followed the <a href="http://blog.xoppa.com/basic-3d-using-libgdx-2/" title="Basic 3D using LibGDX">Basic 3D using LibGDX tutorial</a>, you've already seen the Usage.Position and Usage.Normal of VertexAttributes when we created the box using ModelBuilder. The "TEXCOORD0" attribute specifies that the mesh contains texture coordinates.

The vertices array is just a huge array of floating point values representing the mesh. Notice how each vertex is printed on a single line. On every line the first three values represent the position, the next three values represent the normal and finally the last two values represent the texture coordinates (UV).

The parts array of the first mesh, contains three objects. These are known as mesh-parts. We've seen above that the Model class has a separate meshParts array. That array holds all the parts within all meshes. Now look at the first part. It contains three members. The first is <em>id</em>, which is an unique identifier, internally used to identify the part. Next is the <em>type</em>, which defines how the part should be rendered (the primitive type). Theoretically this can be a different value, but in practice it will always be "TRIANGLES", meaning that the part is made up of triangles where each triangle is specified by three vertices. Finally there is the <em>indices</em> array. Again this is a huge array of numbers where each number is used to identify a vertex within the vertices array. So e.g. a value of 0 means the first line in the vertices array and a value of 1 means the second line. Since the type is set to TRIANGLES, the first three values (0, 1, 2) specify the first triangle, the second three values (1, 0, 3) specify the second triangle, and so on. Note that each line is made up of twelve values, meaning there are four triangles on one line.

Now if you take a quick look at the <a href="https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/Mesh.java" title="Mesh.java">LibGDX Mesh class</a>, you'll see that it contains the following lines:

```java
public class Mesh implements Disposable {
	...
	final VertexData vertices;
	final IndexData indices;
	...
```

Here VertexData can be seen as a huge array of floating point values, which will match up with the vertices array of the mesh within the g3dj file. IndexData can also be seen as a huge array, but now with short values. This array will be filled with the indices of every part of the mesh within the g3dj file, flattening out the parts. So in case of the first mesh, it will contain the indices of the first mesh part (mpart1), directly followed by the indices of the second mesh part (mpart2) and then followed by the indices of third mesh part (mpart4). To identify the part within the mesh, we need to know where it's located within the IndexData. Now let's take a look at the <a href="https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/graphics/g3d/model/MeshPart.java" title="MeshPart.java" target="_blank">MeshPart class</a>:

```java
public class MeshPart {
	/** unique id within model **/
	public String id;
	/** the primitive type, OpenGL constant like GL_TRIANGLES **/
	public int primitiveType;
	/** the offset into a Mesh's indices array **/
	public int indexOffset;
	/** the number of vertices that make up this part **/
	public int numVertices;
	/** the Mesh the part references, also stored in {@link Model} **/
	public Mesh mesh;
}
```

Well that's exactly what we need, the indexOffset and numVertices values tell us which part of IndexData is used for this mesh part. So in the LibGDX 3d api we don't render meshes, we render meshparts. This is useful, because now multiple different ModelInstances can share the same mesh. This practically reduces the number of times the mesh needs to be bound, which increases performance. In fact, notice how we started with four models (ship, block, invader and spacesphere) and now instead only have two meshes, but with a total of four mesh parts. Fbx-conv has merged the meshes which share the same attributes for us. The second mesh doesn't contain TEXCOORD0, because the block model didn't had a texture on it. We can optimize this by adding texture coordinates to the block model and simply not applying a texture. This will reduce the number of mesh binds to only once. I will not walk through that process now, because it is modeling application specific. But keep in mind that having the same vertex attributes will help merging meshes. You can always convert your scene to G3DJ to quickly check these kind of optimizations.

Let's continue and have a look add the materials array:

```json
	"materials": [
		{
			"id": "sphere2_auv1", 
			"diffuse": [ 1.000000,  1.000000,  1.000000], 
			"textures": [
				{
					"id": "file3", 
					"filename": "invader.png", 
					"type": "DIFFUSE"
				}
			]
		}, 
		{
			"id": "lambert2", 
			"diffuse": [ 1.000000,  1.000000,  1.000000], 
			"textures": [
				{
					"id": "file1", 
					"filename": "space.jpg", 
					"type": "DIFFUSE"
				}
			]
		}, 
		{
			"id": "cube1_auv1", 
			"diffuse": [ 1.000000,  1.000000,  1.000000], 
			"textures": [
				{
					"id": "file2", 
					"filename": "ship.png", 
					"type": "DIFFUSE"
				}
			]
		}, 
		{
			"id": "block_default1", 
			"diffuse": [ 0.000000,  0.000000,  1.000000]
		}
	],
```

Here we see that the file contains four materials. Every material has an unique id, which is the same as the name of the material within the modeling application. As you can see the id's don't make a lot of sense. This is because we imported them from the obj files. However, I would encourage you to give your materials a useful name. It allows you to identify the material within the materials array of the Model class. Next, the material contains a <em>diffuse</em> value, which represents the diffuse color of the material as array of the red, green and blue components ranging from 0 to 1. So a value of [0.5, 0.5, 0.5] would be gray and a value of [1.0, 0.0, 0.0] would be red. Note that the last material, which is the material for the block model, has a blue diffuse color. Finally the first three materials also have a <em>textures</em> array, in which the textures are defined that should be applied. Again, the id of the texture is the same as the name of the texture within the modeling application, which I would encourage you to give a useful name. The <em>filename</em> value is obviously the filename of the texture. And the <em>type</em> value specifies how the texture should be applied, which is "DIFFUSE" in this case, but another value could be e.g. "NORMALMAP". We'll not dive deeper into materials for now, but notice that every value except the id, is optional. If a value is absent, it will simply not be applied.

Now let's take a look at the nodes array:

```json
	"nodes": [
		{
			"id": "space", 
			"parts": [
				{
					"meshpartid": "mpart1", 
					"materialid": "lambert2", 
					"uvMapping": [[  0]]
				}
			]
		}, 
		{
			"id": "ship", 
			"rotation": [ 0.000000,  1.000000,  0.000000,  0.000000], 
			"translation": [ 0.000000,  0.000000,  6.000000], 
			"parts": [
				{
					"meshpartid": "mpart2", 
					"materialid": "cube1_auv1", 
					"uvMapping": [[  0]]
				}
			]
		}, 
		{
			"id": "block1", 
			"translation": [-5.000000,  0.000000,  3.000000], 
			"parts": [
				{
					"meshpartid": "mpart3", 
					"materialid": "block_default1"
				}
			]
		}, 
		...
		{
			"id": "block6", 
			"translation": [ 5.000000,  0.000000,  3.000000], 
			"parts": [
				{
					"meshpartid": "mpart3", 
					"materialid": "block_default1"
				}
			]
		}, 
		{
			"id": "invader1", 
			"translation": [-5.000000,  0.000000,  0.000000], 
			"parts": [
				{
					"meshpartid": "mpart4", 
					"materialid": "sphere2_auv1", 
					"uvMapping": [[  0]]
				}
			]
		}, 
		...
		{
			"id": "invader30", 
			"translation": [ 5.000000,  0.000000, -8.000000], 
			"parts": [
				{
					"meshpartid": "mpart4", 
					"materialid": "sphere2_auv1", 
					"uvMapping": [[  0]]
				}
			]
		}
	], 
```

Well, that looks familiar. It contains every model instance we created within the modeling application. Each node has an id which matches the name we used within the modeling application and we've used in the <a href="http://blog.xoppa.com/loading-a-scene-with-libgdx/" title="Loading a scene with LibGDX">previous tutorial</a> to create each ModelInstance. Some nodes also have a translation or rotation value. Also for these values the rule applies that if they are absent, they're simply not applied. So the "space" node e.g. is not translated or rotated at all, while the "ship" node is both rotated and translated. Just like we did while modeling. The next value is the <em>parts</em> array, which is the place where everything comes together. It describes how the node should be rendered. Each item, called node-part, contains the id of both the mesh part and material, meaning that the specified mesh part should be rendered with the specified material. The <em>uvMapping</em> value is used to specify which texture coordinates should be used for which texture. Remember we had a "TEXCOORD0" vertex attribute and a "DIFFUSE" texture. Now consider the scenario where we would have both "TEXCOORD0" and "TEXCOORD1" vertex attributes and both a "DIFFUSE" and a "NORMALMAP" texture. The uvMapping array specifies which texture should be used for TEXCOORD0 (e.g. DIFFUSE) and which texture should be used for TEXCOORD1 (e.g. NORMALMAP).

In this case the <em>parts</em> array only contains one node-part. But think about e.g. a car model, where you want to render the whole model with a texture applied, except for the windows, which should be rendered with a black color. In that case you would have two parts, one containing the mesh part of the windows and a black material and one containing the mesh part of the remaining of the car and a material with the texture. You should always keep this in mind. Whenever you apply two or more different materials to a model within you modeling application, it will be split up into multiple node parts. In case of the car example, we could optimize this by adding a small black rectangle to the texture and setting the texture coordinates of the windows so that they will only cover the middle of that small black rectangle.

On a quick side note. If you used the old 3d model class of LibGDX, this is where the two can be compared. The old StillModel class had an array of SubMesh classes, which contained a Material, Mesh and primitive-type. The NodePart class contains about the same information, but with the difference of referencing a MeshPart (which contains the primitive-type) instead of directly referencing a Mesh.

Again, the nodes within the g3dj file one on one match up with the nodes within the Model class. So the nodes within the Model match up with the nodes within the modeling application. Most modeling application allow those nodes to be hierarchical and so does the Model class. Every Node has a children array which contains it's child nodes. We'll not look further into that for now, but remember that if you use hierarchy within you modeling application, it will also be reflected within the model class.

That leaves us with the last array within our g3dj file, named <em>animations</em>, which is obviously used for animated models. We will cover animations later, but for now just note that it's in there.

Now, if you like, you can copy the invaderscene.g3dj file to the data folder within the assets folder and load that file instead of the invaderscene.g3db file. This allows you to easily change some values and see how it affects the scene. E.g. try to change the diffuse color of the ship's material, or applying a different material to it.

[Next: Behind the 3D scenes - part2]({% post_url 2013-06-23-behind-the-3d-scenes-part2 %})
