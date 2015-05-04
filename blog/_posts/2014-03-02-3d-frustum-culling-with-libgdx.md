---

layout: blogpost

comments: true
title: 3D frustum culling with libGDX
tags: libGDX 3D Graphics
author: Xoppa
eye_catch: frustumculling2.png

---

When rendering a 3D scene, the actual amount of visible objects is often a lot less than the total amount of objects within the scene. Rendering all object, even those that are not visible, can be a waste of precious GPU time and decrease the speed of the game. Ideally you'd want to render only those objects which are actually visible to the camera and ignore all other objects which for example are behind the camera. This is known as frustum culling and there are several ways to accomplish this. This tutorial will show you the very basics in how to accomplish frustum culling using the 3D api of LibGDX.
<more />

Before we start with the actual frustum culling, we'll need a scene to which apply this technique. Therefor I used the scene and code of the previous tutorial: [Loading a scene with libGDX]({% post_url 2013-05-29-loading-a-scene-with-libgdx %}). I assume you've followed that tutorial and will not go into the details of that code.

It might be useful to have some feedback of how well our frustum culling implementation performs. Therefor let's add a label to indicate the number of object being rendered and while we're at it, let's include the frames per second also. For reference here's the complete listing, we'll discuss the changes below:

```java
public class FrustumCullingTest implements ApplicationListener {
	protected PerspectiveCamera cam;
	protected CameraInputController camController;
	protected ModelBatch modelBatch;
	protected AssetManager assets;
	protected Array instances = new Array();
	protected Environment environment;
	protected boolean loading;
     
	protected Array blocks = new Array();
	protected Array invaders = new Array();
	protected ModelInstance ship;
	protected ModelInstance space;
    
	protected Stage stage;
	protected Label label;
	protected BitmapFont font;
	protected StringBuilder stringBuilder;
     
    @Override
    public void create () {
		stage = new Stage();
		font = new BitmapFont();
		label = new Label(" ", new Label.LabelStyle(font, Color.WHITE));
		stage.addActor(label);
		stringBuilder = new StringBuilder();
    	
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
        assets.load(data+"/invaderscene.g3db", Model.class);
        loading = true;
    }
 
    private void doneLoading() {
        Model model = assets.get(data+"/invaderscene.g3db", Model.class);
        for (int i = 0; i &lt; model.nodes.size; i++) {
            String id = model.nodes.get(i).id;
            ModelInstance instance = new ModelInstance(model, id, true);

            if (id.equals(&quot;space&quot;)) {
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
     
    private int visibleCount;
    @Override
    public void render () {
        if (loading && assets.update())
            doneLoading();
        camController.update();
         
        Gdx.gl.glViewport(0, 0, Gdx.graphics.getWidth(), Gdx.graphics.getHeight());
        Gdx.gl.glClear(GL10.GL_COLOR_BUFFER_BIT | GL10.GL_DEPTH_BUFFER_BIT);
 
        modelBatch.begin(cam);
        visibleCount = 0;
        for (final ModelInstance instance : instances) {
        	if (isVisible(cam, instance)) {
        		modelBatch.render(instance, environment);
        		visibleCount++;
        	}
        }
        if (space != null)
            modelBatch.render(space);
        modelBatch.end();
        
        stringBuilder.setLength(0);
        stringBuilder.append(" FPS: ").append(Gdx.graphics.getFramesPerSecond());
        stringBuilder.append(" Visible: ").append(visibleCount);
        label.setText(stringBuilder);
        stage.draw();
    }
    
    protected boolean isVisible(final Camera cam, final ModelInstance instance) {
    	return true; // FIXME: Implement frustum culling
    }
     
    @Override
    public void dispose () {
        modelBatch.dispose();
        instances.clear();
        assets.dispose();
    }

	@Override
	public void resize(int width, int height) {
		stage.getViewport().update(width, height, true);
	}

	@Override
	public void pause() {
	}

	@Override
	public void resume() {
	}
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/frustumculling/step1/FrustumCullingTest.java">View on github</a>

Just a few changes here, let's discuss them. First I added a Stage, Label, BitmapFont and StringBuilder;

```java
    protected Stage stage;
    protected Label label;
    protected BitmapFont font;
    protected StringBuilder stringBuilder;
```

Next, in the create method we initialize these members:

```java
    @Override
    public void create () {
        stage = new Stage();
        font = new BitmapFont();
        label = new Label(" ", new Label.LabelStyle(font, Color.WHITE));
        stage.addActor(label);
        stringBuilder = new StringBuilder();
        ...
    }
```

Note that in the doneLoading method I used a convenience method to create ModelInstance. The third argument (mergeTransform) does exactly the same as we manually did before, namely set the transformation of the ModelInstance and reset the transformation of the Node.

If you're unfamiliar with using a Stage (scene2d), I'd suggest following a tutorial on that also, since it's a great way to implement your UI. Now let's look at the render method where we actually draw the UI on top of the 3D scene:

```java
    private int visibleCount;
    @Override
    public void render () {
        ...
        modelBatch.begin(cam);
        visibleCount = 0;
        for (final ModelInstance instance : instances) {
        	if (isVisible(cam, instance)) {
        		modelBatch.render(instance, environment);
        		visibleCount++;
        	}
        }
        if (space != null)
            modelBatch.render(space);
        modelBatch.end();
        
        stringBuilder.setLength(0);
        stringBuilder.append(" FPS: ").append(Gdx.graphics.getFramesPerSecond());
        stringBuilder.append(" Visible: ").append(visibleCount);
        label.setText(stringBuilder);
        stage.draw();
    }
```

We don't render all the instances at once anymore, but instead first check each instance if it's visible and only render it if is. Also we increase visibleCount to keep track of the number of instances actually being rendered. Note that the space ModelInstance isn't counted and always drawn, since it's always visible anyway.

Next we use the StringBuilder to build a string containing the frames per second and the number of instances (excluding the space) actually visible. We set the text of the label and finally draw the stage. Note that using a StringBuilder is highly recommended against string concatenation in your render method. The StringBuilder will create less garbage, causing almost no hick-ups due to garbage collection.

```java
	@Override
	public void resize(int width, int height) {
		stage.getViewport().update(width, height, true);
	}
```

We need to update the stage's viewport in the resize method. The last Boolean argument set the origin to the lower left coordinate, causing the label to be drawn at that location.

So, the isVisible method is where the magic happens and where gets decided if a ModelInstance is rendered or not. For now it's just a method stub and always returns true:

```java
    protected boolean isVisible(final Camera cam, final ModelInstance instance) {
    	return true; // FIXME: Implement frustum culling
    }
```

Let's run it and see it in action:

<a href="frustumculling1.png"><img src="frustumculling1.png" alt="frustumculling1" width="300" /></a>

As you can see it says there are 37 visible objects. 1 ship, 6 blocks and 30 invaders (and 1 space which isn't counted). If you move the camera around, you'll notice that the number always stays 37, regardless if all objects are actually visible. Time to implement frustum culling.

If you are not familiar with the term frustum: a frustum can be seen as a shape much like a pyramid in 3d space with the tip at the camera and the body containing everything the camera can see. <a href="http://www.badlogicgames.com/wordpress/?p=1550" title="New Camera Classes in libgdx" target="_blank">Here is a good article on frustum and camera</a>, I'd suggest reading that if you're having trouble understanding or visualizing the frustum of a camera. In practice the frustum is made up of 6 planes, namely the left, right, top, bottom, near and far planes. If an object is between those six planes, then it's visible to the camera, if it's outside (any of) those six planes then it's not visible to the camera.

Luckily libGDX provides some very easy methods to check if an object is inside the frustum. Let's implement that check:

```java
    private Vector3 position = new Vector3();
    protected boolean isVisible(final Camera cam, final ModelInstance instance) {
    	instance.transform.getTranslation(position);
    	return cam.frustum.pointInFrustum(position);
    }
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/frustumculling/step2/FrustumCullingTest.java">View on github</a>

Here we added a Vector3 to hold the position. In the isVisible method we fetch the position of the ModelInstance and next we check if that position is inside the frustum. Let's see how well that performs:

<a href="frustumculling2.png"><img src="frustumculling2.png" alt="frustumculling2" width="300" /></a>

Well that seems about right. But if you look closely, you'll notice that you'll see objects pop in and out as you move/rotate the camera. They are culled too soon. That's because the position we fetch using instance.transform.getTranslation(position) represents the center of the ModelInstance. And while the center of the instance might not be inside the frustum, that doesn't mean that the complete instance isn't inside the frustum. For example, if the center of the ship is not visible to the camera, it's still possible that its right wing is partly visible to the camera.

To solve this we'll need to make sure that the object really isn't visible to the camera before culling it. However, checking each and every vertex (point) of the object against the frustum is going to be very costly and probably counterproductive. We can estimate if the object inside the camera using the dimensions of the object. This will make sure the object is always rendered if it's inside the frustum, but might lead to false-positives (where the object isn't visible, but its dimensions are inside the frustum).

To implement this, we'll have to store the dimensions of the ModelInstance with the ModelInstance. This can be easily achieved by extending ModelInstance and implementing the (only) constructor we need:

```java
public class FrustumCullingTest implements ApplicationListener {
	public static class GameObject extends ModelInstance {
		public final Vector3 center = new Vector3();
		public final Vector3 dimensions = new Vector3();
		
		private final static BoundingBox bounds = new BoundingBox();
		
		public GameObject(Model model, String rootNode, boolean mergeTransform) {
			super(model, rootNode, mergeTransform);
			calculateBoundingBox(bounds);
			bounds.getCenter(center);
			bounds.getDimensions(dimensions);
		}
	}
	...
}
```

Here we calculate the BoundingBox of the ModelInstance. The center of the BoundingBox might not match the origin of the model (= the center of the model in the modeling application). Therefor we store this value in the center Vector3 member. Next we store the dimensions of the ModelInstance in the dimensions member. Note that the BoundingBox used is a static member, meaning it is reused for each GameObject we create.

Since we extended ModelInstance with GameObject, we'll have to replace every occurrence of ModelInstance to GameObject. I won't include those changes, but replacing those should be straight-forward.

Let's update the isVisible method to include the dimensions:

```java
    protected boolean isVisible(final Camera cam, final GameObject instance) {
    	instance.transform.getTranslation(position);
    	position.add(instance.center);
    	return cam.frustum.boundsInFrustum(position, instance.dimensions);
    }
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/frustumculling/step3/FrustumCullingTest.java">View on github</a>

If you run this, you'll notice that the object doesn't pop in and out anymore and still there's a good frustum culling. However, it might not be accurate in all scenario's. For example, if you rotate a GameObject, its dimensions aren't rotated. The easiest (and probably fastest) way to solve this is to think of a sphere (around the center) which contains every possible rotation of the dimensions. Let's include this in the GameObject:

```java
	public static class GameObject extends ModelInstance {
		public final Vector3 center = new Vector3();
		public final Vector3 dimensions = new Vector3();
		public final float radius;
		
		private final static BoundingBox bounds = new BoundingBox();
		
		public GameObject(Model model, String rootNode, boolean mergeTransform) {
			super(model, rootNode, mergeTransform);
			calculateBoundingBox(bounds);
			bounds.getCenter(center);
			bounds.getDimensions(dimensions);
			radius = dimensions.len() / 2f;
		}
	}
```

Here we simple take half the length of the dimension to set it to the radius. Now update the frustum culling method:

```java
    protected boolean isVisible(final Camera cam, final GameObject instance) {
    	instance.transform.getTranslation(position);
    	position.add(instance.center);
    	return cam.frustum.sphereInFrustum(position, instance.radius);
    }
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/frustumculling/step4/FrustumCullingTest.java">View on github</a>

Here we use the convenience method sphereInFrustum to check against the frustum. Note that checking against a radius is a bit faster, but it might cause more false-positives.

In [the next tutorial]({% post_url 2014-03-22-interacting-with-3d-objects %}) we will use this knowledge to interact with 3D objects.
