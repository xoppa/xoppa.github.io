---

layout: blogpost

comments: true
title: Using materials with libGDX
tags: libGDX 3D Graphics
author: Xoppa

---

In this tutorial we will see how to use materials with <a href="http://libgdx.badlogicgames.com/" title="http://libgdx.badlogicgames.com/" target="_blank">libGDX</a>. Materials are used by shaders, therefor this tutorial continues where we left of in the [previous tutorial]({% post_url 2013-07-19-creating-a-shader-with-libgdx %}), in which we created a custom shader. If you haven't done so, I'd suggest to read that tutorial prior to this one.
<more />

The full <a href="https://github.com/xoppa/blog/tree/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials" title="View source on github" target="_blank">source</a>, <a href="https://github.com/xoppa/blog/tree/master/tutorials/assets/usingmaterials/data" title="View assets on github" target="_blank">assets</a> and a runnable tests of this tutorial can be found on <a href="https://github.com/xoppa/blog" title="Xoppa - Blog - Github" target="_blank">this github repository</a>.

Previously we've only used a Renderable and a Shader to test the shader that we've created. This is great for testing, because it's very easy to see what's going on. But eventually you'd want to go back to the ModelInstance and ModelBatch we've used earlier, so you can easily use multiple shaders and models. Luckily this is very easy to accomplish, so let's change the code a bit. For reference, here is the full ShaderTest.java code, we'll discuss the changes below:

```java
public class ShaderTest implements ApplicationListener {
   public PerspectiveCamera cam;
   public CameraInputController camController;
   public Shader shader;
   public Model model;
   public Array<ModelInstance> instances = new Array<ModelInstance>();
   public ModelBatch modelBatch;
    
   @Override
   public void create () {
       cam = new PerspectiveCamera(67, Gdx.graphics.getWidth(), Gdx.graphics.getHeight());
       cam.position.set(0f, 8f, 8f);
       cam.lookAt(0,0,0);
       cam.near = 1f;
       cam.far = 300f;
       cam.update();
       
       camController = new CameraInputController(cam);
       Gdx.input.setInputProcessor(camController);

       ModelBuilder modelBuilder = new ModelBuilder();
       model = modelBuilder.createSphere(2f, 2f, 2f, 20, 20,
      	 new Material(),
      	 Usage.Position | Usage.Normal | Usage.TextureCoordinates);
       
       for (int x = -5; x <= 5; x+=2) {
      	 for (int z = -5; z<=5; z+=2) {
             instances.add(new ModelInstance(model, x, 0, z));
      	 }
       }

       shader = new TestShader();
       shader.init();
       
       modelBatch = new ModelBatch();
   }

   @Override
   public void render () {
   	camController.update();
        
   	Gdx.gl.glViewport(0, 0, Gdx.graphics.getWidth(), Gdx.graphics.getHeight());
   	Gdx.gl.glClear(GL10.GL_COLOR_BUFFER_BIT | GL10.GL_DEPTH_BUFFER_BIT);

   	modelBatch.begin(cam);
   	for (ModelInstance instance : instances)
   		modelBatch.render(instance, shader);
   	modelBatch.end();
   }
    
   @Override
   public void dispose () {
       shader.dispose();
       model.dispose();
       modelBatch.dispose();
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

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step1/MaterialTest.java" title="View full source code on github" target="_blank">View full source code on github</a>

First we've moved the camera a bit further away from the origin, so it's covers the scene we're going to create. Next, we've removed the renderable object. Instead we use an array of ModelInstances which we fill with spheres placed on a grid on the XZ plane. We've also removed the RenderContext and now create a ModelBatch instead. The ModelBatch must be disposed, so we also added a line to our dispose method. If you followed the previous tutorials, all of these changes should be really straight forward. The only new aspect of this code, is in the render method:

```java
modelBatch.begin(cam);
for (ModelInstance instance : instances)
	modelBatch.render(instance, shader);
modelBatch.end();
```

Here we specify the shader that we want the ModelBatch to use for the supplied ModelInstance. Which is the easiest method to supply a custom shader to the ModelBatch. So let's run it and see what it renders:

<a href="shadertest8.png"><img src="shadertest8.png" alt="shadertest8" width="300" /></a>

Now we're going to apply a different color to each sphere. To do that, we must first change the shader. As we've seen previously the shader consists of a CPU part (the TestShader.java file) and a GPU part. That GPU part is made up of code that's run for each vertex (test.vertex.glsl) and code that's run for each fragment (test.fragment.glsl).

Here is the modified test.vertex.glsl file:

```glsl
attribute vec3 a_position;
attribute vec3 a_normal;
attribute vec2 a_texCoord0;

uniform mat4 u_worldTrans;
uniform mat4 u_projTrans;

void main() {
	gl_Position = u_projTrans * u_worldTrans * vec4(a_position, 1.0);
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/assets/usingmaterials/data/color.vertex.glsl" title="View full source code on github" target="_blank">View full source code on github</a>

Not much changed, we've only removed the v_texCoord0 varying, because the fragment shader will not be needing it anymore. Here's the test.fragment.glsl file:

```glsl
#ifdef GL_ES 
precision mediump float;
#endif

uniform vec3 u_color;

void main() {
	gl_FragColor = vec4(u_color, 1.0);
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/assets/usingmaterials/data/color.fragment.glsl" title="View full source code on github" target="_blank">View full source code on github</a>

Here we've also removed the `v_texCoord0` varying and instead added a uniform `u_color`. This uniform is used to specify the color of the fragment. So we'll need to set that uniform from within the TestShader class. For reference, here is the complete TestShader code:

```java
public class TestShader implements Shader {
	ShaderProgram program;
	Camera camera;
	RenderContext context;
	int u_projTrans;
	int u_worldTrans;
	int u_color;
	
	@Override
	public void init () {
		String vert = Gdx.files.internal("data/test.vertex.glsl").readString();
		String frag = Gdx.files.internal("data/test.fragment.glsl").readString();
		program = new ShaderProgram(vert, frag);
		if (!program.isCompiled())
			throw new GdxRuntimeException(program.getLog());
		u_projTrans = program.getUniformLocation("u_projTrans");
		u_worldTrans = program.getUniformLocation("u_worldTrans");
		u_color = program.getUniformLocation("u_color");
	}
	
	@Override
	public void dispose () {
		program.dispose();
	}
	
	@Override
	public void begin (Camera camera, RenderContext context) {
		this.camera = camera;
		this.context = context;
		program.begin();
		program.setUniformMatrix(u_projTrans, camera.combined);
		context.setDepthTest(true, GL20.GL_LEQUAL);
		context.setCullFace(GL20.GL_BACK);
	}
	
	@Override
	public void render (Renderable renderable) {
		program.setUniformMatrix(u_worldTrans, renderable.worldTransform);
		program.setUniformf(u_color, MathUtils.random(), MathUtils.random(), MathUtils.random());
		renderable.mesh.render(program,
			renderable.primitiveType,
			renderable.meshPartOffset,
			renderable.meshPartSize);
	}
	
	@Override
	public void end () {
		program.end();
	}
	
	@Override
	public int compareTo (Shader other) {
		return 0;
	}
	@Override
	public boolean canRender (Renderable instance) {
		return true;
	}
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step2/TestShader.java" title="View full source code on github" target="_blank">View full source code on github</a>

The only change here is that we added an u_color value which holds the uniform location and in the render method we set it to a random color. Let's run it:

<a href="shadertest9.png"><img src="shadertest9.png" alt="shadertest9" width="300" /></a>

Using a random color doesn't give us much control. We need a way to inform the shader which color should be used for each renderable. The very most basic method to do this is using the userData value of ModelInstance. Here is how we could do that in ShaderTest:

```java
   public void create () {
       ...
       for (int x = -5; x <= 5; x+=2) {
      	 for (int z = -5; z<=5; z+=2) {
      		 ModelInstance instance = new ModelInstance(model, x, 0, z);
      		 instance.userData = new Color((x+5f)/10f, (z+5f)/10f, 0, 1);
      		 instances.add(instance);
      	 }
       }
       ...
   }
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step3/MaterialTest.java" title="View full source code on github" target="_blank">View full source code on github</a>

Here we simply set userData to the color we want to use in the shader, which in this example depends on the location of the instance. Next we need to use that value within the shader:

```java
	public void render (Renderable renderable) {
		...
		Color color = (Color)renderable.userData;
		program.setUniformf(u_color, color.r, color.g, color.b);
		...
	}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step3/TestShader.java" title="View full source code on github" target="_blank">View full source code on github</a>

<a href="shadertest10.png"><img src="shadertest10.png" alt="shadertest10" width="300" /></a>

So, the userData value can be used to pass data down to the shader. But if you have multiple uniforms this can get really messy and if you use multiple shaders, it will be even a more pain to keep track of the uniform values. We need a better way to set the uniform values at the model instance.

First, have a quick look at what we are actually trying to accomplish over here. Our shader above has three uniforms (`u_projTrans`, `u_worldTrans` and `u_color`). The first depends on the camera, the second (renderable.worldTransform) and third depend on the renderable. In general you can divide your uniforms into three specific groups:

* <u><i>Global:</i></u> these are all values you can set in the shader's begin method. They are the same for all renderables and never changed between the begin() and end() call. For example the u_projTrans is a global uniform.
* <u><i>Environmental:</i></u> these are all values which aren't global, but also not directly related to the ModelInstance. Most commonly these depend on the location of the ModelInstance within the scene. For example the lights that are applied (renderable.lights) are typical environmental values.
* <u><i>Specific:</i></u> these are all values which are specific to the ModelInstance (NodePart). Independent of the scene or location within the scene, these values should always be applied. For example the u_worldTrans and u_color are specific values.

<span style="font-size:10px">Here, the term <i>specific</i> refers to values specific to the part of the ModelInstance, you could also refer to them as <i>local</i> or <i>model</i> values.</span>

Note that not only uniforms are included in those groups. For example the vertex attributes are values always specific for the renderable (MeshPart). The depth test and cull face values of the render context, like we set in the begin() method of our shader, are global values for the shader. All these settings and values together define the context in which the GLSL is run.

When creating a Shader (the CPU part), you should always keep in mind which value belongs to which group and how often you expect them to be changed. For example a ShadowMap texture can be a global value or environmental value, but it's unlikely to be a specific value.

For this tutorial we will only look at the specific values, like the u_color uniform above. This is where materials come in.

A material contains only specifc values. The MeshPart of the renderable defines the shape that should be rendered. Likewise the material defines how the shape should be rendered (e.g. which color to apply on the shape), regardless of it's environment. A renderable always has a material (it must never be null). A material basically is an array of material attributes:

```java
class Material {
	Array<Material.Attribute> attributes = new Array<Attribute>();
	...
}
```

<span style="font-size:10px">Note that while the material must never be null, the material itself can be empty.</span>

In it's simplest form a Attribute describes <i>which value</i> should be set to <i>which uniform</i>. Of course the type of value can vary, e.g. an uniform can be a color, float or texture. Therefor Attribute must be extended for each specific type. LibGDX comes with most basic types, for example:

```java
package com.badlogic.gdx.graphics.g3d.materials;
...
public class ColorAttribute extends Attribute {
    public final Color color = new Color();
    ...
}
```

Likewise there is FloatAttribute and TextureAttribute (and a few others). So to specify <i>the value</i> is as simple as:

```java
colorAttribute.color.set(Color.RED);
```

To specify <i>the uniform</i>, each Attribute has a <i>type</i> value:

```java
public class Material implements Iterable<Attribute>, Comparator<Attribute> {
	public static abstract class Attribute {
		public final long type;
		...
	}
	...
}
```

Just like a shader can only contain one uniform with the same name, a material can only contain one attribute with the same type value. But while an uniform name is only used for one shader, a material attribute can be used by many shaders. Therefor the material attribute is independent of the shader. For example, in our shader above, we have a uniform called u_color. If we would have an attribute that defines just a color, it would be ambiguous. Instead we need to better define what is actually the purpose of the material attriute, e.g. <i>"this attribute specifies the overall fully opaque diffuse color"</i>. LibGDX already contains such a material attribute, which can be constructed as follows:

```java
ColorAttribute attribute = new ColorAttribute(ColorAttribute.Diffuse, Color.RED);
```

For convenience, you can also create the attribute like this:

```java
ColorAttribute attribute = ColorAttribute.createDiffuse(Color.RED);
```

Here `ColorAttribute.DiffuseColor` is the type value. So getting the attribute from the material can be done in a similar way:

```java
Material.Attribute attribute = material.get(ColorAttribute.Diffuse);
```

Note that by definition the attribute with the type of ColorAttribute.Diffuse must always be castable to ColorAttribute. (e.g. you cannot call `new TextureAttribute(ColorAttribute.Diffuse);`, that wouldn't make sense). So the attribute can safely be cast back to ColorAttribute:

```java
ColorAttribute attribute = (ColorAttribute)material.get(ColorAttribute.Diffuse);
```

Let's bring that in practice and modify the ShaderTest:

```java
   public void create () {
       ...
       for (int x = -5; x <= 5; x+=2) {
      	 for (int z = -5; z<=5; z+=2) {
      		 ModelInstance instance = new ModelInstance(model, x, 0, z);
      		 ColorAttribute attr = ColorAttribute.createDiffuse((x+5f)/10f, (z+5f)/10f, 0, 1);
      		 instance.materials.get(0).set(attr);
      		 instances.add(instance);
      	 }
       }
       ...
   }
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step4/MaterialTest.java" title="View full source code on github" target="_blank">View full source code on github</a>

Here we create a new ColorAttribute of type ColorAttribute.Diffuse with a color depending on the place on the grid. Next we add the color to the first (and only) material of the instance. Note that the method is called <em>set</em> (and not add), because it will override any existing attribute with the same type.

Now change TestShader to use that value to set the `u_color` uniform:

```java
	public void render (Renderable renderable) {
		program.setUniformMatrix(u_worldTrans, renderable.worldTransform);
		Color color = ((ColorAttribute)renderable.material.get(ColorAttribute.Diffuse)).color;
		program.setUniformf(u_color, color.r, color.g, color.b);
		...
	}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step4/TestShader.java" title="View full source code on github" target="_blank">View full source code on github</a>

Here we fetch material attribute of type `ColorAttribute.Diffuse`, cast it to ColorAttribute and fetch it's color value. Next that color value is used to set the u_color uniform. If you run this code, you will see it renders exactly the same as before. But now we use material instead of userData.

If you look closely at the render method we just changed, you will see that it will terribly fail if the material doesn't contain an attribute of type ColorAttribute.Diffuse. We could add a check, for example:

```java
ColorAttribute attr = (ColorAttribute)renderable.material.get(ColorAttribute.Diffuse);
if (attr != null)
	... set the uniform
else
	... fall back to a default color
```

In some cases that can be very useful. But in other cases (like our shader), it probably would be better to use another shader (or the default shader) for that specific renderable. We can make sure that our shader is only used for a renderable containing a specific material attribute, by implementing the canRender method. Let's add that to the TestShader:

```java
	public boolean canRender (Renderable renderable) {
		return renderable.material.has(ColorAttribute.Diffuse);
	}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step5/TestShader.java" title="View full source code on github" target="_blank">View full source code on github</a>

Now the shader will only be used if the material contains a ColorAttribute.Diffuse attribute. Otherwise ModelBatch will fall back to the default shader.

When creating a more complex shader, you'll probably need material attributes which aren't included by default. So let's create a more complex (but still very simple) shader. First, bring back the v_texCoord0 varying to vertex shader (test.vertex.glsl):

```glsl
attribute vec3 a_position;
attribute vec3 a_normal;
attribute vec2 a_texCoord0;

uniform mat4 u_worldTrans;
uniform mat4 u_projTrans;

varying vec2 v_texCoord0;

void main() {
	v_texCoord0 = a_texCoord0;
	gl_Position = u_projTrans * u_worldTrans * vec4(a_position, 1.0);
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/assets/usingmaterials/data/uvcolor.vertex.glsl" title="View full source code on github" target="_blank">View full source code on github</a>

Next, change the fragment shader to use the texture coordinates to specify the color (test.fragment.glsl):

```glsl
#ifdef GL_ES 
precision mediump float;
#endif

uniform vec3 u_colorU;
uniform vec3 u_colorV;

varying vec2 v_texCoord0;

void main() {
	gl_FragColor = vec4(v_texCoord0.x * u_colorU + v_texCoord0.y * u_colorV, 1.0);
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/assets/usingmaterials/data/uvcolor.fragment.glsl" title="View full source code on github" target="_blank">View full source code on github</a>

Here, instead of one uniform, we use two uniforms to create the pixel color. One color depending on the X (U) texture coordinate and one depending on the Y (V) texture coordinate. Finally we need to alter TestShader to set both these uniforms:

```java
public class TestShader implements Shader {
	ShaderProgram program;
	Camera camera;
	RenderContext context;
	int u_projTrans;
	int u_worldTrans;
	int u_colorU;
	int u_colorV;
	
	@Override
	public void init () {
		...
		u_worldTrans = program.getUniformLocation("u_worldTrans");
		u_colorU = program.getUniformLocation("u_colorU");
		u_colorV = program.getUniformLocation("u_colorV");
	}
	...
	@Override
	public void render (Renderable renderable) {
		program.setUniformMatrix(u_worldTrans, renderable.worldTransform);
		Color colorU = ((ColorAttribute)renderable.material.get(ColorAttribute.Diffuse)).color;
		Color colorV = Color.BLUE;
		program.setUniformf(u_colorU, colorU.r, colorU.g, colorU.b);
		program.setUniformf(u_colorV, colorV.r, colorV.g, colorV.b);
		renderable.mesh.render(program,
			renderable.primitiveType,
			renderable.meshPartOffset,
			renderable.meshPartSize);
	}
	...
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step6/TestShader.java" title="View full source code on github" target="_blank">View full source code on github</a>

We fetch the uniform location of both `u_colorU` and `u_colorV` uniforms. Then we set `u_colorU` to the color of the diffuse color attribute. And we set `u_colorV` to `Color.BLUE`.

<a href="shadertest11.png"><img src="shadertest11.png" alt="shadertest11" width="300" /></a>

In the shader we've now used the diffuse color attribute for the color depending on the X texture coordinate. And we've use Color.BLUE as the color depending on the Y texture coordinate. But it would be better if both would be configurable using a material attribute. So we need to create two material attributes, one for diffuse color based on the U value and one for the diffuse color based on the V value. The easiest way to accomplish that is to extend to ColorAttribute class and register additional type values, so let's do that:

```java
public class TestShader implements Shader {
	public static class TestColorAttribute extends ColorAttribute {
		public final static String DiffuseUAlias = "diffuseUColor";
		public final static long DiffuseU = register(DiffuseUAlias);

		public final static String DiffuseVAlias = "diffuseVColor";
		public final static long DiffuseV = register(DiffuseVAlias);

		static {
			Mask = Mask | DiffuseU | DiffuseV;
		}
		
		public TestColorAttribute (long type, float r, float g, float b, float a) {
			super(type, r, g, b, a);
		}
	}
	...
}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step7/TestShader.java" title="View full source code on github" target="_blank">View full source code on github</a>

Since this is such a small class, I didn't create a new java file, but instead created a static subclass of TestShader. The class extends ColorAttribute, because that's the type of attribute we need to register. Within that class we have a few public final static members. The first is DiffuseUAlias, which is the name of the attribute type we're going to define. This value e.g. is returned when calling <em>attribute.toString()</em>. On the next line we register that name as an attribute type. The register method returns the type value (which is globally unique) and we use that to initialize the DiffuseU value. This allows us to use the <em>TestColorAttribute.DiffuseU</em> as attribute type, just like the <em>ColorAttribute.Diffuse</em> value. Next we do the same for DiffuseVAlias and register is a the type DiffuseV.

We now have registered two new material attributes DiffuseU and DiffuseV. But since we're extending the ColorAttribute class, we also need to inform that class the accept those attributes. This is done on the next line (<em>Mask = Mask | DiffuseU | DiffuseV</em>). Finally we implement a constructor, so we can actually construct the newly created material attribute.

Let's use these two new material attributes, first change the ShaderTest class:

```java
   public void create () {
       ...
       for (int x = -5; x <= 5; x+=2) {
      	 for (int z = -5; z<=5; z+=2) {
      		 ModelInstance instance = new ModelInstance(model, x, 0, z);
      		 ColorAttribute attrU = new TestColorAttribute(TestColorAttribute.DiffuseU, (x+5f)/10f, 1f - (z+5f)/10f, 0, 1);
      		 instance.materials.get(0).set(attrU);
      		 ColorAttribute attrV = new TestColorAttribute(TestColorAttribute.DiffuseV, 1f - (x+5f)/10f, 0, (z+5f)/10f, 1);
      		 instance.materials.get(0).set(attrV);
             instances.add(instance);
      	 }
       }
       ...
   }
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step7/MaterialTest.java" title="View full source code on github" target="_blank">View full source code on github</a>

Here we now create two TestColorAttributes, one of type DiffuseU and one of type DiffuseV. And we set the color of them both depending on the location of the grid. Finally we add them to the material, just like we did before. Now we need to change the TestShader to actually use them:

```java
	public void render (Renderable renderable) {
		program.setUniformMatrix(u_worldTrans, renderable.worldTransform);
		Color colorU = ((ColorAttribute)renderable.material.get(TestColorAttribute.DiffuseU)).color;
		Color colorV = ((ColorAttribute)renderable.material.get(TestColorAttribute.DiffuseV)).color;
		program.setUniformf(u_colorU, colorU.r, colorU.g, colorU.b);
		program.setUniformf(u_colorV, colorV.r, colorV.g, colorV.b);
		renderable.mesh.render(program,
			renderable.primitiveType,
			renderable.meshPartOffset,
			renderable.meshPartSize);
	}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step7/TestShader.java" title="View full source code on github" target="_blank">View full source code on github</a>

In the render method we fetch the color of both attributes and set the uniforms accordingly. Pretty much the same as before. Now if you run this, it will look something like this:

<a href="shadertest12.png"><img src="shadertest12.png" alt="shadertest12" width="300" /></a>

Well that's not what we expected. That's because we haven't updated the canRender method yet. Since the material now doesn't contain a ColorAttribute.Diffuse attribute anymore, ModelBatch falls back to the default shader. Even while we explicitly specified the shader when calling <em>modelBatch.render(instance, shader);</em>, ModelBatch guards us from using a shader which can't be used. We need to change the canRender method of the TestShader class, to inform the ModelBatch it now accepts the new material attributes:

```java
	public boolean canRender (Renderable renderable) {
		return renderable.material.has(TestColorAttribute.DiffuseU | TestColorAttribute.DiffuseV);
	}
```

<a href="https://github.com/xoppa/blog/blob/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step8/TestShader.java" title="View full source code on github" target="_blank">View full source code on github</a>

Let's run it and see how it renders:

<a href="shadertest13.png"><img src="shadertest13.png" alt="shadertest13" width="300" /></a>

That's better, we can now control our uniforms with custom material attributes.

You might have noticed that I used bitwise operators to combine multiple material attribute types. For example:

```java
return renderable.material.has(TestColorAttribute.DiffuseU | TestColorAttribute.DiffuseV);
```

This will only return true if the material has both the DiffuseU attribute and the DiffuseV attribute. For now, we will not look further into that, but keep in mind that material attributes can be combined for faster comparison.

On a final note, material attributes don't have to specify an uniform. For example, LibGDX comes with the attribute type `IntAttribute.CullFace`, which can be used together with `context.setCullFace(...)` and doesn't involve setting an uniform value. Likewise an attribute doesn't have to specify a single value. For example, in the shader above we've used two color attributes, DiffuseU and DiffuseV. A better approach would be to create a single attribute containing two color values, e.g.:

```java
public class DoubleColorAttribute extends Attribute {
	public final static String DiffuseUVAlias = "diffuseUVColor";
	public final static long DiffuseUV = register(DiffuseUVAlias);
   
	public final Color color1 = new Color();
	public final Color color2 = new Color();
		
	protected DoubleColorAttribute (long type, Color c1, Color c2) {
		super(type);
		color1.set(c1);
		color2.set(c2);
	}

	@Override
	public Attribute copy () {
		return new DoubleColorAttribute(type, color1, color2);
	}

	@Override
	protected boolean equals (Attribute other) {
		DoubleColorAttribute attr = (DoubleColorAttribute)other;
		return type == other.type && color1.equals(attr.color1) && color2.equals(attr.color2);
	}

	@Override
	public int compareTo (Attribute other) {
		if (type != other.type)
			return (int) (type - other.type);
		DoubleColorAttribute attr = (DoubleColorAttribute) other;
		return color1.equals(attr.color1)
				? attr.color2.toIntBits() - color2.toIntBits()
				: attr.color1.toIntBits() - color1.toIntBits();
	}
}
```

<a href="https://github.com/xoppa/blog/tree/master/tutorials/src/com/xoppa/blog/libgdx/g3d/usingmaterials/step9" title="View full source code on github" target="_blank">View full source code on github</a>

Next: [3D frustum culling with libgdx]({% post_url 2014-03-02-3d-frustum-culling-with-libgdx %})
