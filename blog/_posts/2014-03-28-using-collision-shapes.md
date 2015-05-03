---

layout: blogpost

comments: true
title: Using collision shapes
tags: libGDX 3D Graphics

---

Previously we've used a bounding box and bounding sphere to check whether or not an object is [visible to the camera](http://blog.xoppa.com/3d-frustum-culling-with-libgdx) or [touched and dragged](http://blog.xoppa.com/interacting-with-3d-objects). We've also seen that this can cause false positives. Sometimes you need a more precise method to check intersection. Collision shapes can be used to get a very accurate intersection check at almost no performance costs. They are also fundamental for collision detection and physics.
<more />

We'll start where we left off in the <a href="http://blog.xoppa.com/interacting-with-3d-objects/" title="Interacting with 3D objects">previous tutorial</a> and use that code as base for this tutorial. The code and assets are also available on <a href="https://github.com/xoppa/blog" title="Xoppa - blog" target="_blank">github</a>, along with a runnable test.

Currently we use the methods <code>isVisible</code> and <code>getObject</code> to compare a 3D object (a <code>GameObject</code>) against either a frustum or a ray. Let's move those checks to the <code>GameObject</code> class to make the code a bit cleaner:
[java]
public class ShapeTest extends InputAdapter implements ApplicationListener {
	public static class GameObject extends ModelInstance {
		public final Vector3 center = new Vector3();
		public final Vector3 dimensions = new Vector3();
		public final float radius;

		private final static BoundingBox bounds = new BoundingBox();
		private final static Vector3 position = new Vector3();

		public GameObject (Model model, String rootNode, boolean mergeTransform) {
			super(model, rootNode, mergeTransform);
			calculateBoundingBox(bounds);
			bounds.getCenter(center);
			bounds.getDimensions(dimensions);
			radius = dimensions.len() / 2f;
		}
		
		public boolean isVisible(Camera cam) {
			return cam.frustum.sphereInFrustum(transform.getTranslation(position).add(center), radius);
		}

		/** @return -1 on no intersection, or when there is an intersection: the squared distance between the center of this 
		 * object and the point on the ray closest to this object when there is intersection. */
		public float intersects(Ray ray) {
			transform.getTranslation(position).add(center);
			final float len = ray.direction.dot(position.x-ray.origin.x, position.y-ray.origin.y, position.z-ray.origin.z);
			if (len < 0f)
				return -1f;
			float dist2 = position.dst2(ray.origin.x+ray.direction.x*len, ray.origin.y+ray.direction.y*len, ray.origin.z+ray.direction.z*len);
			return (dist2 <= radius * radius) ? dist2 : -1f;
		}
	}
	...
}
[/java]
I renamed the test to <code>ShapeTest</code> and added a <code>static</code> variable called <code>position</code> to be used in the calculation. The <code>isVisible</code> is basically a copy of the method we've seen in the <a href="http://blog.xoppa.com/3d-frustum-culling-with-libgdx/" title="3D frustum culling with libgdx">previous tutorial</a>, although I modified it a bit to a single line using method chaining. The <code>intersects</code> methods is basically a copy of the relevant part (the body of the <code>for</code>-loop) of the <code>getObject</code> method. In these tutorials I don't discuss much about comments, but this is on of those methods that really need at least some comment to explain what its result means, so I also added that.

Now we can remove the <code>isVisible(final Camera cam, final GameObject instance)</code> method and change the <code>render</code> and <code>getObject</code> accordingly.
[java]
	@Override
	public void render () {
		...
		for (final GameObject instance : instances) {
			if (instance.isVisible(cam)) {
				modelBatch.render(instance, environment);
				visibleCount++;
			}
		}
		...
	}
	...
	public int getObject (int screenX, int screenY) {
		Ray ray = cam.getPickRay(screenX, screenY);
		int result = -1;
		float distance = -1;
		for (int i = 0; i < instances.size; ++i) {
			final float dist2 = instances.get(i).intersects(ray);
			if (dist2 >= 0f && (distance < 0f || dist2 <= distance)) { 
				result = i;
				distance = dist2;
			}
		}
		return result;
	}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/shapes/step1/ShapeTest.java" target="_blank">View on github</a>

These changes should be straight forward. We now use <code>instance.isVisible(cam)</code> and <code>instances.get(i).intersects(ray)</code>.

If you run this, you'll see it does exactly the same as it did before. Including the inaccuracy of the bounding sphere intersection check. Our goal is to increase this accuracy and we will use collision shapes to achieve this. A collision shape is a basically a small set of mathematical methods to check a shape against for example the frustum or pick ray. The idea is to use a different shape for each differently shaped object. The easiest way to implement that is to use a small <code>interface</code> and use that to identify the shape.
[java]
public class ShapeTest extends InputAdapter implements ApplicationListener {
	public interface Shape {
		public abstract boolean isVisible(Matrix4 transform, Camera cam);
		/** @return -1 on no intersection, or when there is an intersection: the squared distance between the center of this 
		 * object and the point on the ray closest to this object when there is intersection. */
		public abstract float intersects(Matrix4 transform, Ray ray);
	}
	
	public static class GameObject extends ModelInstance {
		public Shape shape;

		public GameObject (Model model, String rootNode, boolean mergeTransform) {
			super(model, rootNode, mergeTransform);
		}
		
		public boolean isVisible(Camera cam) {
			return shape == null ? false : shape.isVisible(transform, cam);
		}
		
		public float intersects(Ray ray) {
			return shape == null ? -1f : shape.intersects(transform, ray);
		}
	}
	...
}
[/java]
We added a simple <code>interface</code> called <code>Shape</code>, which we will implement next. Because a Shape can be (re)used for multiple objects, I've also added an additional argument to specify the transformation matrix (the location, rotation and scale) of the object we want to check. The <code>GameObject</code> now holds a reference to a shape and simple calls the appropriate method (<code>isVisible</code> or <code>intersects</code>) of the shape. I made the shape optional (it can be null), if the object doesn't have a shape it will simple return a default value. Finally, I've removed the center, dimension, radius and <code>static</code> bounds and position variables, since it's now the shape takes care of that.

In practice almost every shapes needs to know the center and dimension of the shape. So we might as well make a small base class from which to derive each shape implementation.
[java]
	public static abstract class BaseShape implements Shape {
		protected final static Vector3 position = new Vector3();
		public final Vector3 center = new Vector3();
		public final Vector3 dimensions = new Vector3();
		
		public BaseShape(BoundingBox bounds) {
			bounds.getCenter(center);
			bounds.getDimensions(dimensions);
		}
	}
[/java]
The <code>abstract BaseShape</code> class holds the center and dimensions vectors. I've also added a <code>static protected position</code> vector, since all shapes will need to calculate the center position of the shape. To set the center and dimensions vector I've added a constructor which takes the <code>BoundingBox</code> and sets these variables accordingly.

Note that depending on your needs it might be better to only use an abstract base class and no interface, or to add additional methods to the interface to get the center or dimensions.

Now, to test these changes we'll need to implement at least one shape. Let's use the sphere shape for that, since that's the one we previously used:
[java]
	public static class Sphere extends BaseShape {
		public float radius;
		
		public Sphere(BoundingBox bounds) {
			super(bounds);
			radius = bounds.getDimensions().len() / 2f;
		}
		
		@Override
		public boolean isVisible(Matrix4 transform, Camera cam) {
			return cam.frustum.sphereInFrustum(transform.getTranslation(position).add(center), radius);
		}
		
		@Override
		public float intersects(Matrix4 transform, Ray ray) {
			transform.getTranslation(position).add(center);
			final float len = ray.direction.dot(position.x-ray.origin.x, position.y-ray.origin.y, position.z-ray.origin.z);
			if (len < 0f)
				return -1f;
			float dist2 = position.dst2(ray.origin.x+ray.direction.x*len, ray.origin.y+ray.direction.y*len, ray.origin.z+ray.direction.z*len);
			return (dist2 <= radius * radius) ? dist2 : -1f;
		}
	}
[/java]
Basically all this is moving code around. The `Sphere` class extends the `BaseShape` abstract class. In the constructor it calls the constructor of the `BaseShape` and sets the radius like we did previously. Likewise the <code>isVisible</code> and <code>intersects</code> methods are copied from the GameObject class, we only added the <code>transform</code> argument.

Now all we need to do to test this, is update the test class to assign a shape to each object. Since we have three different shapes (ship, block and invader) in our scene, we only need three shapes which we will reuse for each object.
[java]
public class ShapeTest extends InputAdapter implements ApplicationListener {
	...
	protected Shape blockShape;
	protected Shape invaderShape;
	protected Shape shipShape;
	...
	private BoundingBox bounds = new BoundingBox();
	private void doneLoading () {
		Model model = assets.get(data + "/invaderscene.g3db", Model.class);
		for (int i = 0; i < model.nodes.size; i++) {
			String id = model.nodes.get(i).id;
			GameObject instance = new GameObject(model, id, true);

			if (id.equals("space")) {
				space = instance;
				continue;
			}

			instances.add(instance);

			if (id.equals("ship")) {
				if (shipShape == null) {
					instance.calculateBoundingBox(bounds);
					shipShape = new Sphere(bounds);
				}
				instance.shape = shipShape;
				ship = instance;
			}
			else if (id.startsWith("block")) {
				if (blockShape == null) {
					instance.calculateBoundingBox(bounds);
					blockShape = new Sphere(bounds);
				}
				instance.shape = blockShape;
				blocks.add(instance);
			}
			else if (id.startsWith("invader")) {
				if (invaderShape == null) {
					instance.calculateBoundingBox(bounds);
					invaderShape = new Sphere(bounds);
				}
				instance.shape = invaderShape;
				invaders.add(instance);
			}
		}

		loading = false;
	}
	...
}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/shapes/step2/ShapeTest.java" target="_blank">View on github</a>

We've added three variables: blockShape, invaderShape and shipshape. In the <code>doneLoading</code> method we already checked the id of each object to see if it is a ship, block or invader. Now we check if the shape for the object is already created. If it isn't we calculate the bounding box of the object and create the shape as <code>Sphere</code>. And finally we assign it to <code>instance.shape</code>.

Now, if you run this, you will again see that nothing has changed. All we did is moving code around and adding an interface and two classes. However, we gained a lot more flexibility and possibilies regarding collision shapes. All we have to is create a shape for each differently shapes object to get a better accuracy at almost no performance cost. Let's start with the shape for the block models, for this we can basically use the bounding box check we've used earlier.
[java]
	public static class Box extends BaseShape {		
		public Box(BoundingBox bounds) {
			super(bounds);
		}
		
		@Override
		public boolean isVisible(Matrix4 transform, Camera cam) {
			return cam.frustum.boundsInFrustum(transform.getTranslation(position).add(center), dimensions);
		}
		
		@Override
		public float intersects(Matrix4 transform, Ray ray) {
			transform.getTranslation(position).add(center);
			if (Intersector.intersectRayBoundsFast(ray, position, dimensions)) {
				final float len = ray.direction.dot(position.x-ray.origin.x, position.y-ray.origin.y, position.z-ray.origin.z);
				return position.dst2(ray.origin.x+ray.direction.x*len, ray.origin.y+ray.direction.y*len, ray.origin.z+ray.direction.z*len);
			}
			return -1f;
		}
	}
[/java]
Here we added the <code>Box</code> shape. The center and dimensions are already set in the <code>BaseShape</code>, so we only have to implement the required methods. For the <code>isVisible</code> method, we use the <code>cam.frustum.boundsInFrustum</code> method, we've used earlier. Likewise for the <code>intersects</code> method we use the <code>Intersector.intersectRayBoundsFast</code> method to check for the intersection. If there's an intersection, we calculate the point on the ray closest to the center of the object and return the squared distance between them. Otherwise we return -1f, just like we do in the Sphere shape.

Well that was relatively easy, let's update the <code>doneLoading</code> method to use this shape instead of the sphere shape.
[java]
	private void doneLoading () {
	...
			else if (id.startsWith("block")) {
				if (blockShape == null) {
					instance.calculateBoundingBox(bounds);
					blockShape = new Box(bounds);
				}
				instance.shape = blockShape;
				blocks.add(instance);
			}
	...
	}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/shapes/step3/ShapeTest.java" target="_blank">View on github</a>

Now if you run this, you'll notice that the method is very accurate for the blocks.

Let's also add a shape for the invaders. If you look closely at the invaders, you'll that's roughly something like a disc shape. It's not really round (more like an octagon), but it approximates it. For the shape it is fine to assume it is round. If needed, you could add a small offset to the radius to compensate for this assumption, although in most cases this wont be necessary. The top and bottom part of the invaders aren't very height and steep to the center. For simplicity of this tutorial, we'll use only the disc (circle) for this shape. If needed, you could take the height into account to get a better intersection check, for example by using a cylinder shape.
[java]
	public static class Disc extends BaseShape {
		public float radius;
		public Disc(BoundingBox bounds) {
			super(bounds);
			radius = 0.5f * (dimensions.x > dimensions.z ? dimensions.x : dimensions.z);
		}
		
		@Override
		public boolean isVisible (Matrix4 transform, Camera cam) {
			return cam.frustum.sphereInFrustum(transform.getTranslation(position).add(center), radius);
		}
		
		@Override
		public float intersects (Matrix4 transform, Ray ray) {
			transform.getTranslation(position).add(center);
			final float len = (position.y - ray.origin.y) / ray.direction.y;
			final float dist2 = position.dst2(ray.origin.x + len * ray.direction.x, ray.origin.y + len * ray.direction.y, ray.origin.z + len * ray.direction.z);
			return (dist2 < radius * radius) ? dist2 : -1f;
		}
	}
[/java]
In the constructor we calculate the radius of the disc. Unlike the sphere (which we used for all possible rotations, more on that shortly), we simply take the biggest of the width and depth to calculate the radius. Like said before, you could add a small offset (e.g. multiply it by 1.1f) to compensate the radius. In the <code>isVisible</code> method we simply use the sphere frustum check, since this method doesn't have to be very precise.

The <code>intersects</code> is based on the calculation within the <code>touchDragged</code> we implemented earlier. First we calculate the center of the object. Next we calculate the location on the ray where the y-coordinate is the same as the y-coordinate of the center of the object. Next we calculate the squared distance between this point and the center of the object. If the distance is more than the radius then we return -1f otherwise we return the squared distance.

Next, the <code>doneLoading</code> method needs to be updated to use this shape for the invaders:
[java]
	private void doneLoading () {
	...
			else if (id.startsWith("invader")) {
				if (invaderShape == null) {
					instance.calculateBoundingBox(bounds);
					invaderShape = new Disc(bounds);
				}
				instance.shape = invaderShape;
				invaders.add(instance);
			}
	...
	}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/shapes/step4/ShapeTest.java" target="_blank">View on github</a>

Now if you run this, you'll notice that selecting the invaders is much more accurate. You might even want to update the code to use the object closest to the camera, instead of the one closest to the ray. Just like we did before, but later changed to compensate for the inaccuracy.

As you now hopefully understand, the collision shape you choose affects the algorithm used to check for example against a ray or frustum. There's always a trade-off between accuracy and performance impact. For example, what if we really want to check against the octagon of the invader, instead of a disc shape? Well, that would be a more complex implementation than we used now and it may be doubtful if that's worth it. Of course there are also methods to optimize such check. For example first check against bounding sphere (which is very cheap and has no false-negatives) and only if that succeeds, use the more accurate and costly algorithm.

In this tutorial we only looked at the checks against the frustum and the pick ray. These are commonly called: <code>sphere-frustum, sphere-ray, box-frustum, box-ray, disc-frustum</code> and <code>disc-ray</code>. It is possible to add more checks, for example <code>sphere-sphere, sphere-box, box-disc</code> etc. Most commonly this is done in a separate class (comparable to the <code>Intersector</code> class we used earlier). This makes collision detection very easy to implement.

It is possible to use collision shapes for more complex shapes (for example the ship). And even if it is not possible to represent a model using a single shape, it is possible to use a combination of shapes (for example create a wing shape and combine two of them to create the ship). When it's also not possible to use basic shapes, then a convex shape might be an option (basically a bag tightly wrapped around the object). And if you really need more precision than that, you could always fall back to a concave shape or even the actual mesh data (which might be very slow).

You might be wondering why LibGDX doesn't contain these shape classes. While it would be great to have some good shape classes, the actual required implementation might be different depending on the needs. However, Libgdx does contain a 3d physics extension, which contains most common shapes. In practice, when you need collision detection and/or physics, you might want to use that extension, including its shapes. We will look into this soon.

[Previously](http://blog.xoppa.com/3d-frustum-culling-with-libgdx) we decided to use spheres because bounds don't work for rotation. Likewise, the shapes we just created will fail when the objects are rotated or scaled. It is relatively easy to solve this (for example transform the ray into "shape-space"). However, in practice you probably don't have to care about that math and simply use for example the 3d physics extension.

[Next: Using the libGDX 3D physics Bullet wrapper - part1](http://blog.xoppa.com/using-the-libgdx-3d-physics-bullet-wrapper-part1)

