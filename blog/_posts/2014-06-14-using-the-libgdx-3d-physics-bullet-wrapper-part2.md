---

layout: blogpost

comments: true
title: Using the libGDX 3D physics Bullet wrapper - part2
tags: libGDX 3D Graphics

---

<a href="http://http://bulletphysics.org" target="_blank">Bullet</a> is a Collision Detection and Rigid Body Dynamics Library. In this second part of this tutorial we will learn how to use the <a href="http://libgdx.com" target="_blank">LibGDX</a> Bullet wrapper for <em>Rigid Body Dynamics</em>. This is the part where we simulate real world physics like applying forces, responding to collisions and so forth.
<more />

In the [first part](http://blog.xoppa.com/using-the-libgdx-3d-physics-bullet-wrapper-part1) of this tutorial we've seen what the Bullet wrapper is and how we can use it for collision detection. I assume that you've read that prior to this. We'll use its code as the base for this part of the tutorial.

If you have a look at the <a href="https://github.com/libgdx/libgdx/tree/master/extensions/gdx-bullet/jni/swig-src" target="_blank">source</a> of the Bullet wrapper, you'll see that it consists of five parts, each with its own java package. These are:
<ul>
	<li>linearmath</li>
	<li>collision</li>
	<li>dynamics</li>
	<li>softbody</li>
	<li>extras</li>
</ul>
The <code>linearmath</code> package contains some generic classes and methods, which are not directly related to physics. This is for example the <code>btVector3</code> class and the bridge to the LibGDX <code>Vector3</code> class. The <code>collision</code> package contains everything related to collision detection as we've seen before, like the collision shapes, collision world and near and broad phase classes. The <code>dynamics</code> package contains everything related to rigid body dynamics which we'll look at in this tutorial. The <code>softbody</code> package contains every related to soft-body and cloth simulation (in contrast to rigid bodies). And finally, the <code>extra</code> package contains some useful helper classes/tools, at the moment this only contains importing previously saved bullet world (e.g. from Blender).

Note that the order of these packages is important. The collision package uses classes from the linearmath package, but doesn't use classes from the dynamics package. Likewise the dynamics package relies on the collision package, but is independent of the softbody package. Or, in other words: <code>dynamics</code> is built on top of <code>collision</code>. When working with <code>dynamics</code>, you are still able to use the functionality of the <code>collision</code> package.

<strong>Add dynamic properties</strong>
Before we can apply dynamics, we need to set some properties required for Bullet to perform some calculations. E.g. when you push against (apply a force to) an object, then the weight (mass) of the object is important on what happens. If the object is very heavyweight it might not move at all. But when it's lightweight, it might move quite some distance. Other properties, like the friction between the object and the surface it's on are also relevant. Because a <code>btCollisionObject</code> doesn't contain such properties, we need to use a subclass called <code>btRigidBody</code>. As the name suggests this class contains all properties of a rigid body.
[java]
public class BulletTest implements ApplicationListener {
	...
	static class GameObject extends ModelInstance implements Disposable {
		public final btRigidBody body;
		public boolean moving;

		public GameObject (Model model, String node, btRigidBody.btRigidBodyConstructionInfo constructionInfo) {
			super(model, node);
			body = new btRigidBody(constructionInfo);
		}

		@Override
		public void dispose () {
			body.dispose();
		}

		static class Constructor implements Disposable {
			public final Model model;
			public final String node;
			public final btCollisionShape shape;
			public final btRigidBody.btRigidBodyConstructionInfo constructionInfo;
			private static Vector3 localInertia = new Vector3();

			public Constructor (Model model, String node, btCollisionShape shape, float mass) {
				this.model = model;
				this.node = node;
				this.shape = shape;
				if (mass > 0f)
					shape.calculateLocalInertia(mass, localInertia);
				else
					localInertia.set(0, 0, 0);
				this.constructionInfo = new btRigidBody.btRigidBodyConstructionInfo(mass, null, shape, localInertia);
			}

			public GameObject construct () {
				return new GameObject(model, node, constructionInfo);
			}

			@Override
			public void dispose () {
				shape.dispose();
				constructionInfo.dispose();
			}
		}
	}
	...
	@Override
	public void create () {
		...
		constructors = new ArrayMap<String, GameObject.Constructor>(String.class, GameObject.Constructor.class);
		constructors.put("ground", new GameObject.Constructor(model, "ground", new btBoxShape(new Vector3(2.5f, 0.5f, 2.5f)), 0f));
		constructors.put("sphere", new GameObject.Constructor(model, "sphere", new btSphereShape(0.5f), 1f));
		constructors.put("box", new GameObject.Constructor(model, "box", new btBoxShape(new Vector3(0.5f, 0.5f, 0.5f)), 1f));
		constructors.put("cone", new GameObject.Constructor(model, "cone", new btConeShape(0.5f, 2f), 1f));
		constructors.put("capsule", new GameObject.Constructor(model, "capsule", new btCapsuleShape(.5f, 1f), 1f));
		constructors.put("cylinder", new GameObject.Constructor(model, "cylinder", new btCylinderShape(new Vector3(.5f, 1f, .5f)), 1f));
		...
	}
}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/bullet/dynamics/step1/BulletTest.java" target="_blank">View full source on github</a>

Here we replaced the <code>btCollisionObject</code> of the <code>GameObject</code> class with the <code>btRigidBody</code> class, which extends <code>btCollisionObject</code>. To construct the body we now use a <code>btRigidBodyConstructionInfo</code> class. This class contains the <code>btCollisionShape</code> like we've used previously, but also contains other properties like the <code>mass</code>, <code>friction</code>, <code>damping</code> and others. The <code>Constructor</code> class now also contains this <code>constructionInfo</code>, this allows for a very convenient way to construct multiple instances of the same object, like we've seen before.

When creating the <code>btRigidBodyConstructionInfo</code>, we need to specify the mass (the weight of the object), we use <code>null</code> for the second argument (we'll see soon why), next we supply the <code>btCollisionShape</code> as third argument and finally we need to specify the <code>localInertia</code>. <code>localInertia</code> is a static <code>Vector3</code>, so that it will be reused for each <code>Constructor</code>. If the mass is equal or less than zero, we simply set the local inertia also to zero. Otherwise we need to calculate the local intertia. Luckily the collision shape has a nice helper method to calculate it.

Note that the <code>Constructor</code> class still holds the <code>btCollisionShape</code>, even while the <code>btRigidBodyConstructionInfo</code> also contains this information. This is because we still <em>own</em> the <code>shape</code> and need to <code>dispose</code> it when it's no longer needed. Of course the same goes for the <code>constructionInfo</code> member, so I added a line to the <code>dispose</code> method to dispose that as well.

The constructors that we use for our test are mostly still the same, but we now have to include the mass when creating the <code>Constructor</code>. For now we'll just use a mass of <code>1f</code> for each object, except for the ground, where we use a mass of <code>0f</code>.

A zero mass isn't physically possible. It is used to indicate that the ground should not respond to any forces applied to it. It should always stay at the same location (and rotation), regardless of any forces or collisions that may apply to it. This is called a "static" object. The other objects (with a mass greater than zero) are called "dynamic" objects.

<strong>A small note about units</strong>
As you might know, most physics properties need to be specified using units. For example the mass is commonly specified in kilograms. The size of an object is specified in meters and time is specified in seconds. These are called <a href="http://en.wikipedia.org/wiki/Si_units" target="_blank">SI units</a>. It is advised to use them when possible. But sometimes it is just not practical to use them. For example when your objects are very huge or very small.

Bullet performs best when values are around one (<code>1f</code>). There are a lot of factors (like floating point precision), but in practice it's best to keep the values of the properties of the the <code>btRigidBody</code> around one. So, if you are making a space game where the game objects are likely to be much larger than one meter, you might want to decide to use dekameters (*10), hectometers (*100) or kilometers (*1000) instead. Or, if you are making a game where the objects are likely to be much smaller than one meter, you might want to decide to use decimeters (*0.1), centimeters (*0.01) or millimeters (*0.001). Likewise, if your objects are likely to be much heavier or lighter than one kilogram, you might want to scale the mass also.

Scaling your units is generally no problem, as long as you keep consistent. So if you decide to use inches instead of meter, then you should also use inches per second for velocity, inches per second per second for acceleration (like gravity) and kilogram inches per second per second for forces. Read <a href="http://bulletphysics.org/mediawiki-1.5.8/index.php/Scaling_The_World" target="_blank">this article</a> for more information.

Keep in mind that in most cases the size of the (visual) model is the same as the size of the physics body and that the transformation matrix of the model instance can't contain a scaling component. Therefor the size of your model should match the desired units and any scaling applied should be "baked" prior to exporting the model from within your modeling application.

<strong>Add the dynamics world</strong>
To actually apply dynamics, we need to add a dynamics world (<code>btDynamicsWorld</code>). This will replace (and extends) the <code>btCollisionWorld</code> that we've used before:
[java]
public class BulletTest implements ApplicationListener {
	...
	btDynamicsWorld dynamicsWorld;
	btConstraintSolver constraintSolver;

	@Override
	public void create () {
		...
		collisionConfig = new btDefaultCollisionConfiguration();
		dispatcher = new btCollisionDispatcher(collisionConfig);
		broadphase = new btDbvtBroadphase();
		constraintSolver = new btSequentialImpulseConstraintSolver();
		dynamicsWorld = new btDiscreteDynamicsWorld(dispatcher, broadphase, constraintSolver, collisionConfig);
		dynamicsWorld.setGravity(new Vector3(0, -10f, 0));
		contactListener = new MyContactListener();
		
		instances = new Array<GameObject>();
		GameObject object = constructors.get("ground").construct();
		instances.add(object);
		dynamicsWorld.addRigidBody(object.body, GROUND_FLAG, ALL_FLAG);
	}

	public void spawn () {
		GameObject obj = constructors.values[1 + MathUtils.random(constructors.size - 2)].construct();
		obj.transform.setFromEulerAngles(MathUtils.random(360f), MathUtils.random(360f), MathUtils.random(360f));
		obj.transform.trn(MathUtils.random(-2.5f, 2.5f), 9f, MathUtils.random(-2.5f, 2.5f));
		obj.body.setWorldTransform(obj.transform);
		obj.body.setUserValue(instances.size);
		obj.body.setCollisionFlags(obj.body.getCollisionFlags() | btCollisionObject.CollisionFlags.CF_CUSTOM_MATERIAL_CALLBACK);
		instances.add(obj);
		dynamicsWorld.addRigidBody(obj.body, OBJECT_FLAG, GROUND_FLAG);
	}
	...
	@Override
	public void dispose () {
		...
		dynamicsWorld.dispose();
		constraintSolver.dispose();
		...
	}
}
[/java]
I've renamed the <code>collisionWorld</code> member to <code>dynamicsWorld</code> and made it a type of <code>btDynamicsWorld</code> (a subclass of <code>btCollisionWorld</code>). To construct the dynamics world we'll need a <code>btConstraintSolver</code>. We'll not look into constraints into this tutorial, but as you can imagine, this class is used to solve constraints (simply said: constraints can be used to attach objects to each other). We use the <code>btDiscreteDynamicsWorld</code> implementation for the dynamics world.

After we've created the dynamics world, we also set the gravity of the world. This will cause all dynamic objects (but not the static ground) to have gravity be applied to them. For the sake of our test we'll use a gravity of minus 10 meters along the Y axis per second per second. This is close to earth gravity.

Instead of using the <code>addCollisionObject</code> method, we now use the <code>addRigidBody</code> method to add the objects to the world. Note that the <code>addCollisionObject</code> method is still available, but the <code>addRigidBody</code> method makes sure that for example gravity is correctly applied to each object.

Since the objects will now fall down because of gravity, we don't have to move the objects manually. Instead we need to instruct the world to apply the gravity and update the transform of the body.
[java]
	@Override
	public void render () {
		final float delta = Math.min(1f / 30f, Gdx.graphics.getDeltaTime());
		
		dynamicsWorld.stepSimulation(delta, 5, 1f/60f);
		
		for (GameObject obj : instances)
			obj.body.getWorldTransform(obj.transform);
		...
	}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/bullet/dynamics/step2/BulletTest.java" target="_blank">View full source on github</a>

Instead of the <code>performDiscreteCollisionDetection</code> method we now call the <code>stepSimulation</code> method of the world. This will internally still call the <code>performDiscreteCollisionDetection</code> method (including the collision callbacks).

The discrete dynamics world uses a fixed time step. This basically means that it will always use the same delta value to perform calculations. This fixed delta value is supplied as the third argument of <code>stepSimulation</code>. If the actual delta value (the first argument) is greater than the desired fixed delta value, then the calculation will be done multiple times. The maximum number of times that this will be done (the maximum number of sub-steps) is specified by the second argument. Note that we still cap the delta to <code>1f/30f</code>, so the actual number of sub-steps will never exceed 2. If you want to know more about it, there are quite a few resources on fixing the time-step. However, in practice Bullet takes care of it for us, as long as you understand the arguments, it should be fine.

Since Bullet now transforms (translates and rotates) our objects, we need to ask Bullet for the new transformation and set it to ModelInstance#transform. This is done using the call to <code>obj.body.getWorldTransform(obj.transform);</code>.

If you run this, you'll see that it actually already does what we want. The objects now fall down with the force of gravity on the static ground. Since we've used a collision filter, as seen in the previous part of this tutorial, the objects don't respond to each other (they fall through each other).
<a href="http://blog.xoppa.com/wp-content/uploads/2014/05/bullet6.png"><img src="http://blog.xoppa.com/wp-content/uploads/2014/05/bullet6-300x225.png" alt="bullet6" width="300" height="225" class="alignnone size-medium wp-image-558" /></a>

Since the world now fully controls the moving objects, there's no need for the <code>moving</code> member of the GameObject class anymore. Therefor we might as well remove that member. I'll not show that change (it's only removing the lines using the <code>moving</code> member), but it does leave us with an empty contact listener. To verify that the contact listener still is triggered, let's change the color of the objects when they hit the ground.
[java]
public class BulletTest implements ApplicationListener {
	...
	class MyContactListener extends ContactListener {
		@Override
		public boolean onContactAdded (int userValue0, int partId0, int index0, int userValue1, int partId1, int index1) {
			if (userValue0 != 0)
				((ColorAttribute)instances.get(userValue0).materials.get(0).get(ColorAttribute.Diffuse)).color.set(Color.WHITE);
			if (userValue1 != 0)
				((ColorAttribute)instances.get(userValue1).materials.get(0).get(ColorAttribute.Diffuse)).color.set(Color.WHITE);
			return true;
		}
	}
	...
}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/bullet/dynamics/step3/BulletTest.java" target="_blank">View full source on github</a>
<a href="http://blog.xoppa.com/wp-content/uploads/2014/05/bullet7.png"><img src="http://blog.xoppa.com/wp-content/uploads/2014/05/bullet7-300x225.png" alt="bullet7" width="300" height="225" class="alignnone size-medium wp-image-561" /></a>

<strong>Using motion states</strong>
Have a look at following code that we've just added:
[java]
	@Override
	public void render () {
		...
		for (GameObject obj : instances)
			obj.body.getWorldTransform(obj.transform);
		...
	}
[/java]
Basically this asks bullet the current location and rotation of every object. As you'd imagine, if you have a large world this can be quite some objects that are polled for their location and rotation on every <code>render</code> call. However, in practice, there are commonly only just a few objects which are actually moved and/or rotated, if any. Luckily Bullet provides a mechanism to inform us when an object is transformed, so we don't have to iterate through all objects. For this it uses a little callback class called <code>btMotionState</code>.

The <code>btMotionState</code> class has two methods, which you both must override. The <code>setWorldTransform</code> is called by Bullet whenever it has transformed a dynamic object. The <code>getWorldTransform</code> is called by Bullet whenever it needs to know the current transformation of the object, for example when you add the object to the world.
[java]
	static class MyMotionState extends btMotionState {
		Matrix4 transform;
		@Override
		public void getWorldTransform (Matrix4 worldTrans) {
			worldTrans.set(transform);
		}
		@Override
		public void setWorldTransform (Matrix4 worldTrans) {
			transform.set(worldTrans);
		}
	}
[/java]
Here we have implemented a very basic motion state, simply updating a <code>Matrix4</code> instance. Of course it is possible to perform other operations as well. In our test, for example, if an object would fall off the ground (the <code>y</code> value of the location is below zero or a certain threshold) we could remove the object from the world.

Now we need to inform Bullet to use this motion state.
[java]
	static class GameObject extends ModelInstance implements Disposable {
		public final btRigidBody body;
		public final MyMotionState motionState;

		public GameObject (Model model, String node, btRigidBody.btRigidBodyConstructionInfo constructionInfo) {
			super(model, node);
			motionState = new MyMotionState();
			motionState.transform = transform;
			body = new btRigidBody(constructionInfo);
			body.setMotionState(motionState);
		}

		@Override
		public void dispose () {
			body.dispose();
			motionState.dispose();
		}
		...
	}
[/java]
Here we construct the motion state and set its transform to the <code>transform</code> member of the ModelInstance. Note that this is done by reference, so the motion state is directly updating the <code>transform</code> member of the MotionState. The call to <code>setMotionState</code> will inform Bullet to use this motion state. This will also cause Bullet to call the <code>getWorldTransform</code> on the motion state to get the current transform of the object.

We also need to modify the <code>spawn</code> method a bit.
[java]
	public void spawn () {
		GameObject obj = constructors.values[1 + MathUtils.random(constructors.size - 2)].construct();
		obj.transform.setFromEulerAngles(MathUtils.random(360f), MathUtils.random(360f), MathUtils.random(360f));
		obj.transform.trn(MathUtils.random(-2.5f, 2.5f), 9f, MathUtils.random(-2.5f, 2.5f));
		obj.body.proceedToTransform(obj.transform);
		obj.body.setUserValue(instances.size);
		obj.body.setCollisionFlags(obj.body.getCollisionFlags() | btCollisionObject.CollisionFlags.CF_CUSTOM_MATERIAL_CALLBACK);
		instances.add(obj);
		dynamicsWorld.addRigidBody(obj.body, OBJECT_FLAG, GROUND_FLAG);
	}
[/java]
The only change here is that we removed the call to <code>setWorldTransform</code> on the body and instead now call the <code>proceedToTransform</code> method. This instructs Bullet not only to update the world transformation matrix of the object, but to update all other related members as well. 

Finally we don't have to poll each object for its transform in the render method anymore, so let's remove that code.
[java]
	@Override
	public void render () {
		final float delta = Math.min(1f / 30f, Gdx.graphics.getDeltaTime());

		dynamicsWorld.stepSimulation(delta, 5, 1f/60f);

		if ((spawnTimer -= delta) < 0) {
			spawn();
			spawnTimer = 1.5f;
		}
		...
	}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/bullet/dynamics/step4/BulletTest.java" target="_blank">View full source on github</a>

As you hopefully understand, motion states are very powerful and very easy to work with. There's, however, another advantage of using motion states which might not seem that obvious. To understand this, consider what happens when you call <code>dynamicsWorld.stepSimulation(delta, 5, 1f/60f);</code>. Bullet will perform as many (but not more than the specified maximum) calculations using the specified <code>1f/60f</code> delta time, until it reaches the specified actual elapsed delta time. Obviously, it's very unlikely that the <code>delta</code> value is always exactly a multiple of <code>1f/60f</code>. In fact, it is in some cases possible that the value of <code>delta</code> is less than the specified <code>1f/60f</code>, causing Bullet not to perform any calculations at all.

Therefor, the transformation which you get using the <code>obj.body.getWorldTransform()</code> method is not the transformation at the current time, but the transformation at the last calculated fixed time step. To compensate for this, Bullet approximates the transformation at the current time by interpolating the calculated transformation. This interpolated transformation is provided to you in the <code>setWorldTransform</code> method of the motion state.

So, using a motion state will give you a smoother visual transition between time steps. Note the word <em>visual</em> though, the actual collision detection and dynamics calculation (including callbacks) are only done at the fixed time steps.

<strong>A peek behind the scenes</strong>
It is great that Bullet provides us the interpolated transformation matrix and while this almost seems trivial it might get you wondering about some of the internals of Bullet. For example, let's have a look at how it has to calculate this interpolated transformation. Consider a ball falling down, just like we have in our test class, but without any other objects. Let's say that the ball starts at location x=0, y=0, z=0, but for simplicity we'll only look at y-coordinate in this example. At the start of the simulation the ball has no velocity (it "falls" down with zero meter per second). We only apply gravity to the ball, which is an acceleration of 10 meter per second per second downwards:
[java]
position(0) = 0f;
velocity(0) = 0f;
acceleration = -10f;
[/java]
As you probably know, with this information we can calculate the location of the ball at any given time. After one second the velocity is increased with 10 meter per second and the ball has fallen 5 meters downwards, and so on:
[java]
position(t) = 0.5 * acceleration * t * t;
position(0) = 0;
position(1) = -5;
position(2) = -20;
position(3) = -45;
position(4) = -80;
[/java]
<span style="font-size:10px">While you don't have to understand this equation for this example, I'd recommend to read into it if it doesn't look familiar to you.</span>
Now consider that 3.5 seconds have elapsed and we want to know the new location. The most obvious method would be to calculate the new location using above equation: <code>position(3.5) = 0.5 * -10 * 3.5 * 3.5 = -61.25;</code>. While that's the most accurate approach. it isn't feasible in most situations. Practically, this is the same as using a variable time step, which has many disadvantages. For example it misses all collisions that occur between time=0 and time=3.5. And because the delta time will be different each time, the results are not reproducible (you could not replay a physics simulation). Note that you can instruct Bullet to take this approach by providing the value zero for maxSubsteps, which is the second argument of the <code>stepSimulation</code> method, but obviously it is not advised to do so.

Like discussed before, we're using a fixed time-step. Let's say we have a fixed time step of 1 second. Thus, we calculate the location at time = 1 (and perform collision detection and respond to it). Next, we'll calculate the location at time = 2 (again perform collision detection and respond to it). Finally, we'll calculate the location at time = 3 (and perform collision detection and respond to it). Then we'll somehow have to approximate the location at time = 3.5. Let's see what options that we have:

1. We could calculate the location at time = 4 and simply interpolate between them:
[java]
position(3.5) = position(3) + 0.5 * (position(4) - position(3)) = -45 + 0.5 * -35 = -62.5
[/java]
Well, that's pretty close. But it has one big disadvantage: we can't look into the future. To calculate the location at time = 4, we'd have to perform the collision detection and respond to it before it actually happens. This would cause for example the ContactListener to be called while there isn't a contact yet. There are some practical problems with this approach as well, for example interpolation between two transformations can cause strange results

2. We know the velocity of the ball at time = 3, we could use that to approximate the location at time = 3.5:
[java]
velocity(3) = 3 * -10 = -30;
position(3.5) = position(3) + (3.5 - 3) * velocity(3) = -45 + 0.5 * -30 = -60
[/java]
That's pretty close as well. In fact, it used to be the default in Bullet for quite a while. The main problem with this approach, however, is that the approximated transformation doesn't have to line up with the actual location at time = 4. For example if a collision is ahead or for some other reason the velocity is changed.

3. Since the main problem of the previous two approaches is that you can't look into the future, the alternative approach deals with this by literally looking into the past. Instead of approximating the location at time = 3.5, this approach is to approximate the location at time = 2.5:
[java]
velocity(3) = 3 * -10 = -30;
position(2.5) = position(3) - (3 - 2.5) * velocity(3) = -45 - 0.5 * -30 = -30
[/java]
The key factor to this approach is consistency. The visual representation is always exactly one time step behind. Although this is in most scenario's not noticeable, you could compensate for this in your game logic.

Bullet allows you to choose between approach number 2 and 3 using the <code>setLatencyMotionStateInterpolation</code> method. For example to choose approach 2:
[java]
((btDiscreteDynamicsWorld)dynamicsWorld).setLatencyMotionStateInterpolation(false);
[/java]
In these approaches I simplified the calculation for readability. In reality, Bullet uses a much more complex and accurate calculation to estimate the desired transformation. The input for this calculation is stored in dedicated members of the collision object, which are called <code>InterpolationWorldTransform</code>, <code>InterpolationLinearVelocity</code> and <code>InterpolationAngularVelocity</code>. You can get and set each of these using the respective methods, for example: <code>object.setInterpolationWorldTransform(transform);</code>. Remember how we changed the <code>spawn()</code> method to use the <code>proceedToTransform</code> method instead of <code>setWorldTransform</code>? The <code>proceedToTransform</code> method will update the interpolation values as well.

<strong>Using contact callback filtering</strong>
In the previous part of this tutorial we've used collision filtering to make the objects only collide with the ground. Let's remove this filter and make the objects collide with each other as well.
[java]
	public void create () {
		...
		dynamicsWorld.addRigidBody(object.body);
	}

	public void spawn () {
		...
		dynamicsWorld.addRigidBody(obj.body);
	}
[/java]
This change should be straight forward, we've only removed the <code>GOUND_FLAG, ALL_FLAG</code> arguments and <code>OBJECT_FLAG, GROUND_FLAG</code> arguments from the call to <code>addRigidBody</code>. If you run this, you'll see that the objects now also collide with each other:
<a href="http://blog.xoppa.com/wp-content/uploads/2014/06/bullet8.png"><img src="http://blog.xoppa.com/wp-content/uploads/2014/06/bullet8-300x224.png" alt="bullet8" width="300" height="224" class="alignnone size-medium wp-image-593" /></a>
But the objects now also change to a white color when they collide against another object, while we only want to change the color when they collide with the ground. We can simply solve this by slightly modifying the contact listener:
[java]
	class MyContactListener extends ContactListener {
		@Override
		public boolean onContactAdded (int userValue0, int partId0, int index0, int userValue1, int partId1, int index1) {
			if (userValue1 == 0)
				((ColorAttribute)instances.get(userValue0).materials.get(0).get(ColorAttribute.Diffuse)).color.set(Color.WHITE);
			if (userValue0 == 0)
				((ColorAttribute)instances.get(userValue1).materials.get(0).get(ColorAttribute.Diffuse)).color.set(Color.WHITE);
			return true;
		}
	}
[/java]
Here we change the color of the first object only if the second object is the ground and, likewise, change the color of the second object only if the first object is the ground. Remember that we use the <code>userValue</code> here which we've set to the index of the object within the <code>instances</code> array. Since the first object is the ground, it has an index of <code>0</code>.
<a href="http://blog.xoppa.com/wp-content/uploads/2014/06/bullet9.png"><img src="http://blog.xoppa.com/wp-content/uploads/2014/06/bullet9-300x224.png" alt="bullet9" width="300" height="224" class="alignnone size-medium wp-image-594" /></a>
While this works for our test, it isn't optimal. Bullet does not only detect collisions for us, but it now also handles how objects respond to those collisions as well. In most cases, you want Bullet to do it's thing without you being notified for every collision. Only for a small amount of collisions you actually need to be notified. Obviously you can check the collision in the callback, just like we do now in this modified ContactListener. But in that case the wrapper still has to bridge between the native bullet code and your java code for every collision. Luckily the wrapper allows you to specify for which collisions you'd like it to bridge to your ContactListener. This is known as contact callback filtering (sometimes shortened to contact filtering or callback filtering) and is specific to the Bullet wrapper.

Contact callback filtering works very much like (but is not related to) collision filtering. You need to define a bitwise flag for each collision object and a bitwise filter (mask) for which objects you want the callback to be called.
[java]
	public void create () {
		...
		dynamicsWorld.addRigidBody(object.body);
		object.body.setContactCallbackFlag(GROUND_FLAG);
		object.body.setContactCallbackFilter(0);
	}

	public void spawn () {
		...
		dynamicsWorld.addRigidBody(obj.body);
		obj.body.setContactCallbackFlag(OBJECT_FLAG);
		obj.body.setContactCallbackFilter(GROUND_FLAG);
	}
[/java]
Here we tell the wrapper that the ground has the bitwise flag <code>GROUND_FLAG</code> (which is the ninth bit as we've seen in the previous part of this tutorial) and that we don't need to be informed when the ground collides with another object (the filter of the ground is zero). Next we tell the wrapper in the <code>spawn</code> method that each object has the bitwise flag <code>OBJECT_FLAG</code> and that we want to be informed when it collides with the ground (the filter of the object is <code>GROUND_FLAG</code>). When needed, you can combine multiple flags to create a filter, for example: <code>obj.body.setContactCallbackFilter(GROUND_FLAG | WALL_FLAG);</code>.

Note that this is different compared to collision filtering. For collision filtering the filters of both objects need to match the flag of the other object for a collision to occur, while for contact callback filtering only one of the filters has to match with the flag of the other object for the callback to be called.

Now we need to inform the wrapper to actually use contact callback filtering. We can do this by using another signature for the callback method. Like we've seen in the previous part the ContactListener class has several methods that we can override and depending on the signature of the method that we override the wrapper will optimize where possible.
[java]
	@Override
	public boolean onContactAdded (int userValue0, int partId0, int index0, boolean match0, 
					int userValue1, int partId1, int index1, boolean match1) {
		if (match0)
			((ColorAttribute)instances.get(userValue0).materials.get(0).get(ColorAttribute.Diffuse)).color.set(Color.WHITE);
		if (match1)
			((ColorAttribute)instances.get(userValue1).materials.get(0).get(ColorAttribute.Diffuse)).color.set(Color.WHITE);
		return true;
	}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/bullet/dynamics/step5/BulletTest.java" target="_blank">View full source on github</a>

Here we've changed the method signature to include two <code>boolean</code> arguments: <code>match0</code> and <code>match1</code>. The wrapper will "see" this and therefor apply contact callback filtering. Note that by default the contact callback filter will be set to zero, so overriding this method without setting the contact callback flag and filter values, will cause the callback never to be triggered. Also note that you can choose whether or not to use contact callback filtering per callback method. For example, the following would use contact callback filtering for the <code>onContactAdded</code> callback, but not for the <code>onContactProcessed</code> callback:
[java]
	class MyContactListener extends ContactListener {
		@Override
		public boolean onContactAdded (int userValue0, int partId0, int index0, boolean match0,
						int userValue1, int partId1, int index1, boolean match1) {
			...
		}
		@Override
		public void onContactProcessed(int userValue0, int userValue1) {
			...
		}
	}
[/java]
The <code>match0</code> and <code>match1</code> values are used to indicate the filter of which object matches. So in our case, the match variable will only be set for an object colliding with the ground, but not for the ground itself.

<strong>Kinematic bodies</strong>
Let's make our test a bit more interesting by moving the ground up and down. Since the ground is a static body it will not be affected by physics, but the physics will be affected by the ground. Obviously a moving ground isn't really that static anymore. Such an object that does move, but does not respond to collisions, is called a kinematic body. In practice a kinematic body is very much like a static object, except that you can change its location and rotation through code. Before we can start moving the ground, we'll have to inform Bullet that the ground is now a kinematic body.
[java]
	public void create () {
		...
		instances = new Array<GameObject>();
		GameObject object = constructors.get("ground").construct();
		object.body.setCollisionFlags(object.body.getCollisionFlags()
			| btCollisionObject.CollisionFlags.CF_KINEMATIC_OBJECT);
		instances.add(object);
		dynamicsWorld.addRigidBody(object.body);
		object.body.setContactCallbackFlag(GROUND_FLAG);
		object.body.setContactCallbackFilter(0);
	}
[/java]
The only change here is the call to the <code>setCollisionFlags</code> method, where we add the <code>CF_KINEMATIC_OBJECT</code> flag to the ground body. We've seen this method earlier in the <code>spawn</code> method, where we've used it to add the <code>CF_CUSTOM_MATERIAL_CALLBACK</code> flag to inform Bullet that we want to receive <code>onContactAdded</code> callbacks. Much alike, the <code>CF_KINEMATIC_OBJECT</code> informs Bullet that the ground is a kinematic body and that we might want to change its transformation.

So let's move the ground:
[java]
	float angle, speed = 90f;
	@Override
	public void render () {
		final float delta = Math.min(1f / 30f, Gdx.graphics.getDeltaTime());
		
		angle = (angle + delta * speed) % 360f;
		instances.get(0).transform.setTranslation(0, MathUtils.sinDeg(angle) * 2.5f, 0f);
		instances.get(0).body.setWorldTransform(instances.get(0).transform);

		dynamicsWorld.stepSimulation(delta, 5, 1f/60f);
		...
	}
[/java]
Here we've added an <code>angle</code> and <code>speed</code> variable. The <code>angle</code> variable will hold the current angle on which we'll base the location of the ground. The <code>speed</code> variable holds the speed in degrees per second at which the ground will move. In the <code>render</code> method we update the <code>angle</code> variable accordingly. Next we set the y coordinate of the location of the ground to the sine of this angle. This will cause the ground to smoothly move up and down. And finally we set the world transform of the physics body of the ground accordingly, causing Bullet to use this new location to perform the next simulation.

If you run this, you'll see that it actually already does what we want. The ground is moving smoothly up and down and the dynamic objects respond accordingly. But after a while, you'll see that the simulation has flaws. Objects are floating in mid-air or are bouncing on the ground.
<a href="http://blog.xoppa.com/wp-content/uploads/2014/06/bullet10.png"><img src="http://blog.xoppa.com/wp-content/uploads/2014/06/bullet10-300x223.png" alt="bullet10" width="300" height="223" class="alignnone size-medium wp-image-601" /></a>
To understand why this is happening, imagine a very large static ground with thousands of dynamic objects lying still on that ground. Obviously there's a collision between each dynamic object and the ground, so when collision detection is performed both broad phase and near phase collision algorithms including the contact callbacks will have to be executed for all objects. And this will have to be done every frame for all objects, even while there's no dynamics to be performed.

As you might understand, this isn't very effective. That's why Bullet let you specify which objects should be checked for collision. So instead of checking <em>all objects against all objects</em> for collision, Bullet will only check <em>the specified objects against all objects</em> for collision. The objects that should be checked for collision are called <em>active</em> objects. Likewise, objects that shouldn't be checked are called <em>sleeping</em> (or deactivated) objects.

While this on itself is a pure collision detection functionality, the dynamics layer makes it extra powerful by automatically activating and deactivating objects as needed. It does this by monitoring the velocity of the objects. When a body is added to the world or when it is moving, it is made active. As soon as its speed gets below a certain threshold (which you can set using the <code>setSleepingThresholds</code> method) for a certain amount of time (which you can set using the <code>setDeactivationTime</code> method), it is candidate for getting deactivated. If all adjacent objects are also not considered active then it is deactivated. This is done through the activation state:
<ul>
	<li><code>ACTIVE_TAG</code>: <em>the object is active and should be checked for collisions</em>
This is the state that all bodies get when they are added to the world and as soon as they are moving.</li>
	<li><code>WANTS_DEACTIVATION</code>: <em>the velocity of the object has been below the threshold for the required time</em>
This state is used internally by Bullet, you should never have to use it.</li>
	<li><em><code>ISLAND_SLEEPING</code>: This object and all adjacent objects are deactivated</em>
An <em>island</em> is a group of bodies that are adjacent, for example a stack of boxes all belong to the same island.</li>
</ul>
So, in our test class: 
<ol>
	<li>an object is added to the world and is set to the <code>ACTIVE_TAG</code> state</li>
	<li>gravity applies to the object causing it to fall down, its velocity is above the required threshold thus it is kept in the <code>ACTIVE_TAG</code> state</li>
	<li>the object hits the ground causing its velocity to drop below the required threshold, bullet starts counting the time</li>
	<li>the velocity has been below threshold for the required time, the state is set to <code>WANTS_DEACTIVATION</code></li>
	<li>all adjacent objects also don't have the <code>ACTIVE_TAG</code> state, thus the state is set to <code>ISLAND_SLEEPING</code></li>
	<li>the object isn't checked for collision anymore</li>
	<li>the ground is moved, but it doesn't have the <code>ACTIVE_TAG</code> state, causing the object to float in mid-air</li>
</ol>
Because the location and rotation of a kinematic body is set by code and it doesn't have a velocity applied, it isn't automatically activated by Bullet. So basically we have to set the state of the kinematic body our self.
[java]
	public void render () {
		...
		instances.get(0).body.setWorldTransform(instances.get(0).transform);
		instances.get(0).body.setActivationState(Collision.ACTIVE_TAG);
		...
	}
[/java]
For our test this suffices, but this is not the preferred method to make a body active. This is because a timer is used to detect how long the velocity of the body is below the threshold. When activating the body through code, we'd need to reset that timer as well. Luckily bullet has a nice helper method for that called <code>activate()</code>:
[java]
	public void render () {
		...
		instances.get(0).body.setWorldTransform(instances.get(0).transform);
		instances.get(0).body.activate();
		...
	}
[/java]
So, in case we've moved a body, we need to activate that body to make sure that Bullet checks it for collisions. After a short while Bullet will then automatically deactivate the body again. But in our test we're constantly moving the ground, so there's no need for Bullet to automatically deactivate that body. Bullet has two additional activation states for just that:
<ul>
	<li><code>DISABLE_DEACTIVATION</code>: the body will never be (automatically) deactivated</li>
	<li><code>DISABLE_SIMULATION</code>: the body will never be (automatically) activated</li>
</ul>
Thus by setting the activation state to <code>DISABLE_DEACTIVATION</code> right after creating the ground, we don't have to manually activate it in render method anymore.
[java]
	public void create () {
		...
		object.body.setActivationState(Collision.DISABLE_DEACTIVATION);
	}
	...
	public void render () {
		...
		instances.get(0).body.setWorldTransform(instances.get(0).transform);

		dynamicsWorld.stepSimulation(delta, 5, 1f/60f);
	}
[/java]
Earlier we've seen how to use motion states to synchronize the world transformation matrices of the bullet physics bodies and our visual game objects. The <code>getWorldTransform</code> of the motion state of an <em>active</em> kinematic body is automatically called by Bullet to get the latest transformation. So we don't have to manually update the body after moving it:
[java]
	public void render () {
		final float delta = Math.min(1f / 30f, Gdx.graphics.getDeltaTime());
		
		angle = (angle + delta * speed) % 360f;
		instances.get(0).transform.setTranslation(0, MathUtils.sinDeg(angle) * 2.5f, 0f);

		dynamicsWorld.stepSimulation(delta, 5, 1f/60f);
		...
	}
[/java]
<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/bullet/dynamics/step6/BulletTest.java" target="_blank">View full source on github</a>

Be careful: as long as a kinematic body is active, the <code>getWorldTransform</code> method of its motion state is called every time. You should only keep the body activated if you're actually moving or rotating it.

<strong>What's next</strong>
This concludes this second part of the tutorial about the LibGDX Bullet wrapper. Obviously it is impossible to cover every aspect of bullet physics and the libgdx bullet wrapper. For example, it is possible to <a href="https://github.com/libgdx/libgdx/blob/master/tests/gdx-tests/src/com/badlogic/gdx/tests/bullet/InternalTickTest.java" target="_blank">hook onto the internal time steps</a> of Bullet to get even more control on the simulation. You can use bullet <a href="https://github.com/libgdx/libgdx/blob/master/tests/gdx-tests/src/com/badlogic/gdx/tests/bullet/CharacterTest.java" target="_blank">to control your players character</a>. Or you can instruct bullet to use <a href="http://www.bulletphysics.org/mediawiki-1.5.8/index.php/Collision_Callbacks_and_Triggers#Triggers" target="_blank">objects as triggers</a> so you're notified about collision, but it doesn't respond to it. And there's a lot more that Bullet has to offer!

If you haven't done so already, I'd highly suggest to read the [Bullet manual](https://github.com/erwincoumans/bullet2/blob/master/Bullet_User_Manual.pdf?raw=true), 
it is a good read and the place to start when using the Bullet library. It isn't targeted to the LibGDX Bullet wrapper, but much of it still applies. 
Also, the [Bullet website](http://bulletphysics.org/wordpress)Bullet website</a> and especially the [Bullet wiki](http://bulletphysics.org/mediawiki-1.5.8/index.php/Main_Page)
provides a lot of practical information. [The forum](http://www.bulletphysics.org/Bullet/phpBB3) is also quite active.

<a href="https://github.com/libgdx/libgdx/wiki/Bullet-physics" target="_blank">This wiki page</a> provides a lot of information specifically for the LibGDX Bullet wrapper.
