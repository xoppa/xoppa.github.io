---

layout: blogpost

comments: true
title: Loading a 3D scene with libGDX
tags: libGDX 3D Graphics
author: Xoppa

---

In the [previous tutorial]({% post_url 2013-05-25-loading-models-using-libgdx %}) we've seen how to convert, load and display a model with LibGDX. Now we're going to take a look at how to load a complete 3D scene.
<more />

The full <a href="https://github.com/xoppa/blog/tree/master/tutorials/src/com/xoppa/blog/libgdx/g3d/loadscene" title="View source on github" target="_blank">source</a>, <a href="https://github.com/xoppa/blog/tree/master/tutorials/assets/loadscene/data" title="View assets on github" target="_blank">assets</a> and a runnable tests of this tutorial can be found on <a href="https://github.com/xoppa/blog" title="Xoppa - Blog - Github" target="_blank">this github repository</a>. 

We use the same code a before as our base, although I've renamed the class to SceneTest to keep things clean. In that class we have an array of ModelInstance instances, which we will use to define the scene. We already have seen how to load the ship model, so let's add a few more models. You can download the models I've used <a href="http://blog.xoppa.com/wp-content/uploads/invaders.zip">over here</a>. It contains 4 models (obj files): the ship model we used previously, the invader.obj and block.obj models from gdx-invaders and a spacesphere model I quickly put together. The space sphere is simply a large sphere with a texture on it and reversed normals (so the texture is visible from the inside).

Previously we converted the models using fbx-conv. We will do that for these models also, but for now let's just use the obj files. So make sure to copy the files to the data folder within the assets folder and load them like we did previously. For reference, here's the complete code, we will discuss the changes below it:

```java
public class LoadSceneTest implements ApplicationListener {
	public PerspectiveCamera cam;
	public CameraInputController camController;
	public ModelBatch modelBatch;
	public AssetManager assets;
	public Array<ModelInstance> instances = new Array<ModelInstance>();
	public Environment environment;
	public boolean loading;
	
	public Array<ModelInstance> blocks = new Array<ModelInstance>();
	public Array<ModelInstance> invaders = new Array<ModelInstance>();
	public ModelInstance ship;
	public ModelInstance space;
	
	@Override
	public void create () {
		modelBatch = new ModelBatch();
		environment = new Environment();
		environment.set(new ColorAttribute(ColorAttribute.AmbientLight, 0.4f, 0.4f, 0.4f, 1f));
		environment.add(new DirectionalLight().set(0.8f, 0.8f, 0.8f, -1f, -0.8f, -0.2f));
		
		cam = new PerspectiveCamera(67, Gdx.graphics.getWidth(), Gdx.graphics.getHeight());
		cam.position.set(0f, 7f, 10f);
		cam.lookAt(0,0,0);
		cam.near = 1f;
		cam.far = 300f;
		cam.update();

		camController = new CameraInputController(cam);
		Gdx.input.setInputProcessor(camController);
		
		assets = new AssetManager();
		assets.load("data/ship.obj", Model.class);
		assets.load("data/block.obj", Model.class);
		assets.load("data/invader.obj", Model.class);
		assets.load("data/spacesphere.obj", Model.class);
		loading = true;
	}

	private void doneLoading() {
		ship = new ModelInstance(assets.get("data/ship.obj", Model.class));
		ship.transform.setToRotation(Vector3.Y, 180).trn(0, 0, 6f);
		instances.add(ship);

		Model blockModel = assets.get("data/block.obj", Model.class);
		for (float x = -5f; x <= 5f; x += 2f) {
			ModelInstance block = new ModelInstance(blockModel);
			block.transform.setToTranslation(x, 0, 3f);
			instances.add(block);
			blocks.add(block);
		}
		
		Model invaderModel = assets.get("data/invader.obj", Model.class);
		for (float x = -5f; x <= 5f; x += 2f) {
			for (float z = -8f; z <= 0f; z += 2f) {
				ModelInstance invader = new ModelInstance(invaderModel);
				invader.transform.setToTranslation(x, 0, z);
				instances.add(invader);
				invaders.add(invader);
			}
		}
		
		space = new ModelInstance(assets.get("data/spacesphere.obj", Model.class));
		
		loading = false;
	}
	
	@Override
	public void render () {
		if (loading && assets.update())
			doneLoading();
		camController.update();
		
		Gdx.gl.glViewport(0, 0, Gdx.graphics.getWidth(), Gdx.graphics.getHeight());
		Gdx.gl.glClear(GL20.GL_COLOR_BUFFER_BIT | GL20.GL_DEPTH_BUFFER_BIT);

		modelBatch.begin(cam);
		modelBatch.render(instances, environment);
		if (space != null)
			modelBatch.render(space);
		modelBatch.end();
	}
	
	@Override
	public void dispose () {
		modelBatch.dispose();
		instances.clear();
		assets.dispose();
	}
	
	@Override
	public void resume () {
	}
 
	@Override
	public void resize (int width, int height) {
	}
 
	@Override
	public void pause () {
	}
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/loadscene/step1/LoadSceneTest.java" title="View full source code on github" target="_blank">View full source code on github</a>

Now let's go through the changes:

``` java
	public Array<ModelInstance> blocks = new Array<ModelInstance>();
	public Array<ModelInstance> invaders = new Array<ModelInstance>();
	public ModelInstance ship;
	public ModelInstance space;
``` 

Here we added an array to hold our blocks and invaders. And a single ModelInstance for the ship and space. We will still use the instances array for rendering, but this also allows us to easily access each part of the scene. So, if we want to move the ship, we can just use the ship instance.

``` java
	public void create () {
		modelBatch = new ModelBatch();
		...
		cam.position.set(0f, 7f, 10f);
		...
		assets.load("data/ship.obj", Model.class);
		assets.load("data/block.obj", Model.class);
		assets.load("data/invader.obj", Model.class);
		assets.load("data/spacesphere.obj", Model.class);
		loading = true;
	}
```

We set the camera to a location more suitable for the scene we are going to load. And next we tell the assetmanager to load all the models.

``` java
	private void doneLoading() {
		ship = new ModelInstance(assets.get("data/ship.obj", Model.class));
		ship.transform.setToRotation(Vector3.Y, 180).trn(0, 0, 6f);
		instances.add(ship);

		Model blockModel = assets.get("data/block.obj", Model.class);
		for (float x = -5f; x <= 5f; x += 2f) {
			ModelInstance block = new ModelInstance(blockModel);
			block.transform.setToTranslation(x, 0, 3f);
			instances.add(block);
			blocks.add(block);
		}
		
		Model invaderModel = assets.get("data/invader.obj", Model.class);
		for (float x = -5f; x <= 5f; x += 2f) {
			for (float z = -8f; z <= 0f; z += 2f) {
				ModelInstance invader = new ModelInstance(invaderModel);
				invader.transform.setToTranslation(x, 0, z);
				instances.add(invader);
				invaders.add(invader);
			}
		}
		
		space = new ModelInstance(assets.get("data/spacesphere.obj", Model.class));
		
		loading = false;
	}
```

Now here is where things are becoming interesting. On the first line we fetch the ship model and create the ship ModelInstance from it. On the next line we rotate it 180 degrees, so it is facing away from the camera and then move it on the Z axis towards the camera by 6 units. Finally on the third line we add the ship to the instances array, so it will actually be rendered.

Next we do the same for the block and invader models. But now instead of creating just one instance we create multiple instances. The block instances will be aligned in a row on the x axis and we add them to both the instances and the blocks array. Again, this is just for easy access, so we can e.g. check if ship collides a block. The invader instances will be placed on a grid on the XZ plane.

Finally we instantiate the space ModelInstance. Notice that we don't add this one to instances array. This is because we don't want to apply lighting to this model:

``` java
	public void render () {
		...
		modelBatch.begin(cam);
		modelBatch.render(instances, environment);
		if (space != null)
			modelBatch.render(space);
		modelBatch.end();
	}
```

In the render method we render the instances just like before. But now we also separately render the space instance and apply no lights. We need to check if space is set before we do that, because it is loaded asynchronous.

Let's see how it looks:

<a href="scenetest1.png"><img src="scenetest1.png" alt="scenetest1" width="300" /></a>

Well, that's quite nice. We could now just implement some gameplay and call it a day. In fact, I bet there are quite a few games out there that are created this way. But that will not work for bigger scenes, so let's optimize it.

First of, we load four models and in a bigger scene you would likely be loading much more models. Let's optimize that.

Open your favorite modeling application and begin with a new empty scene. I use Maya, but for this example any modeling application will do. Now import all four models from our test (ship, block, invader and spacesphere) into the scene. If you're an inexperienced modeler just like I am, you might want to do this one at the time and make sure each model is displayed correctly. E.g. I had to manually assign the textures and flip the texture coordinates. Also make sure to give each model a simple name and don't apply any transformation on them. It should look something like this:

<a href="scenetest2.png"><img src="scenetest2.png" alt="scenetest2" width="300" /></a>

I enabled X-ray for easy editing, that's why the models look transparent. In the background you see the space sphere, which I called "space". In the front you see the "ship", "block" and "invader" models blended into each other, because they are all set at position (0,0,0). If all models look correct and have the correct name, you can export the scene to FBX, I named it "invaders.fbx". You might also want to save the scene in the modeling application's own format, we're going to need it later on.

Now we're going to convert the FBX file to G3DB. Make sure to grab the latest version of <a href="https://github.com/libgdx/fbx-conv" title="fbx-conv" target="_blank">fbx-conv</a> and convert the FBX file:

```bash
fbx-conv invaders.fbx
```

If you had to flip the texture coordinates while creating the FBX file, you'll probably need to flip them now also. In which case you need to add a command line option:

```bash
fbx-conv -f invaders.fbx
```

If you want to see all command line options, you can run fbx-conv without arguments.

Now copy the just created file invaders.g3db to the data folder within the assets folders and let's load the file:

``` java
public class LoadSceneTest extends GdxTest implements ApplicationListener {
	...
	@Override
	public void create () {
		...
		assets = new AssetManager();
		assets.load("data/invaders.g3db", Model.class);
		loading = true;
	}

	private void doneLoading() {
		Model model = assets.get("data/invaders.g3db", Model.class);
		ship = new ModelInstance(model, "ship");
		ship.transform.setToRotation(Vector3.Y, 180).trn(0, 0, 6f);
		instances.add(ship);

		for (float x = -5f; x <= 5f; x += 2f) {
			ModelInstance block = new ModelInstance(model, "block");
			block.transform.setToTranslation(x, 0, 3f);
			instances.add(block);
			blocks.add(block);
		}
		
		for (float x = -5f; x <= 5f; x += 2f) {
			for (float z = -8f; z <= 0f; z += 2f) {
				ModelInstance invader = new ModelInstance(model, "invader");
				invader.transform.setToTranslation(x, 0, z);
				instances.add(invader);
				invaders.add(invader);
			}
		}
		
		space = new ModelInstance(model, "space");
		
		loading = false;
	}
...
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/loadscene/step2/LoadSceneTest.java" title="View full source code on github" target="_blank">View full source code on github</a>

In the create() method we removed the loading of each individual model and replaced it by the single model invaders.g3db. In the doneLoading() method we fetch that model from the assetmanager. And when creating the ModelInstances we supply that model and the name we used when creating the FBX. So e.g. for the ship ModelInstance we are telling ModelInstance to only represent the model called "ship". Ofcourse the name provided to the ModelInstance must exactly match the name you gave the model in the FBX. We will get deeper into this later, but for now let's run it and see that it's exactly the same as before.

Well that's useful. We can provide all models we need for a scene in a single file. Even more, because we now have a single Model, the ModelInstances share resources. This allows ModelBatch to make optimizations for performance (more on that later). Of course you can still use multiple files if needed, in fact sometimes (e.g. with skinned or animated models) it's easier to have them as a separate file.

But there's more. Let's go back to the modeling application and make sure we have the same scene open as we created before. Now grab the ship model, rotate it 180 degrees around it's Y axis and translate it 6 units along the Z axis. Just like we did in Java code. Next grab the block model and translate it 3 units on Z and -5 on X, then rename "block" to "block1". Now grab the invader model and translate it -5 units on the X axis and rename it to "invader1".

Next duplicate (instance) the block1 model five times and translate each instance 2 units on X from the previous instance, so you end up with six blocks next to each other. Make sure they are named "block1" to "block6". Duplicate instance (which might be called different depending on the modeling application) makes sure that each instance share the same vertex data. Now do the same for invader1, but also along the Z axis so you end up with the same grid we previously created with code. It should look something like this:

<a href="scenetest3.png"><img src="scenetest3.png" alt="scenetest3" width="300" /></a>

Note that the grid spacing I used in the modeling application is 5 units.

Let's export this scene to invaderscene.fbx and convert it to g3db:

```bash
fbx-conv -f invaderscene.fbx
```

Again, the -f command line option is to flip the texture coordinates, which is sometimes required depending on the modeling application. Now let's load the scene using LibGDX:

```java
	public void create () {
		...
		assets = new AssetManager();
		assets.load("data/invaderscene.g3db", Model.class);
		loading = true;
	}

	private void doneLoading() {
		Model model = assets.get("data/invaderscene.g3db", Model.class);
		for (int i = 0; i < model.nodes.size; i++) {
			String id = model.nodes.get(i).id;
			ModelInstance instance = new ModelInstance(model, id);
			Node node = instance.getNode(id);
			
			instance.transform.set(node.globalTransform);
			node.translation.set(0,0,0);
			node.scale.set(1,1,1);
			node.rotation.idt();
			instance.calculateTransforms();
			
			if (id.equals("space")) {
				space = instance;
				continue;
			}
			
			instances.add(instance);
			
			if (id.equals("ship"))
				ship = instance;
			else if (id.startsWith("block"))
				blocks.add(instance);
			else if (id.startsWith("invader"))
				invaders.add(instance);
		}

		loading = false;
	}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/loadscene/step3/LoadSceneTest.java" title="View full source code on github" target="_blank">View full source code on github</a>

Here's what we changed:
First we fetch the invaders model, just like before. As we've seen earlier this contains all models including their name we had in the modeling application. These are stored in the nodes. So we iterate over the nodes and fetch the name (id) of each node. Then we create a ModelInstance using the model and id as arguments, just like we did earlier. On the next line we fetch the node from the instance, which is basically a copy of the node within the model.

Next we set the transformation of the ModelInstance to the transformation of the node. Practically this reads the transformation (like rotation and translation) we set earlier within the modeling application. Then we need to reset the node's transformation, because we now use the ModelInstance transform. For translation this is (0,0,0), for scale this is (1,1,1) and for rotation we set the quaternion to identity. This is followed by a call to calculateTransforms() to make sure the ModelInstance is updated with these new values.

Now that we have the ModelInstance just like it was in the modeling application, we need to add it to our scene. If the id is space, we simply assign the space ModelInstance and continue, because we manually render that. Otherwise we add it to the instances array, so it gets properly rendered. Finally we check the id if it starts with either "ship", "block" or "invader" and accordingly assign the ship ModelInstance or add it to the appropriate Array.

Let's run it and see that it is exactly the same as we had before. But instead, now we have fully designed our scene in the modeling application, making it a lot easier to design.

[Next: Behind the 3D scenes - part1]({% post_url 2013-06-05-behind-the-3d-scenes-part1 %})

You might also want to read: [3D frustum culling with libgdx]({% post_url 2014-03-02-3d-frustum-culling-with-libgdx %})
