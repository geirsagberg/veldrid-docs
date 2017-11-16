---
uid: getting-started-part2
---

# Part 2

## Creating Graphics Resources

It's time to create some Veldrid objects which we will need to render our multi-colored quad. Let's set up some fields in our Program class.

```C#
private static CommandList _commandList;
private static VertexBuffer _vertexBuffer;
private static IndexBuffer _indexBuffer;
private static Shader _vertexShader;
private static Shader _fragmentShader;
private static Pipeline _pipeline;
```

Before our application's main `while` loop, let's call a new method, called `CreateResources`.

```C#
CreateResources();

while (window.Exists)
{
    window.PumpEvents();
}
```

We will create our resources one-by-one, using the ResourceFactory property of our GraphicsDevice. Inside of our new CreateResources method, let's add the following:

```C#
ResourceFactory factory = _graphicsDevice.ResourceFactory;
```

### CommandList

A [CommandList](xref:Veldrid.CommandList) is a device resource that lets you record and execute graphics commands. You can't do anything interesting in Veldrid without one, and advanced techniques make use of many in parallel. In this program, we will use a single CommandList in two ways: to initialize some data, and to issue our rendering commands. Creating a CommandList is simple:

```C#
_commandList = factory.CreateCommandList();
```

### VertexBuffer and IndexBuffer

Next, let's create an array to store our vertex data. For this demo, we just need simple vertices containing a normalized position, and a color. Let's define a structure which will represent each vertex:

```C#
struct VertexPositionColor
{
    public const uint SizeInBytes = 24;
    public Vector2 Position;
    public RgbaFloat Color;
    public VertexPositionColor(Vector2 position, RgbaFloat color)
    {
        Position = position;
        Color = color;
    }
}
```

Let's create an array of these, representing the four corners of our multi-colored quad.

```C#
VertexPositionColor[] quadVertices =
{
    new VertexPositionColor(new Vector2(-.75f, .75f), RgbaFloat.Red),
    new VertexPositionColor(new Vector2(.75f, .75f), RgbaFloat.Green),
    new VertexPositionColor(new Vector2(-.75f, -.75f), RgbaFloat.Blue),
    new VertexPositionColor(new Vector2(.75f, -.75f), RgbaFloat.Yellow)
};
```

We will render these vertices as a Triangle Strip, so we need four indices as well. We will use 16-bit indices:

```C#
ushort[] indexData = { 0, 1, 2, 3 };
```

We need somewhere to store this vertex and index data that the GraphicsDevice can use for rendering. This is accomplished with a [VertexBuffer](xref:Veldrid.VertexBuffer) and an [IndexBuffer](xref:Veldrid.IndexBuffer).

A VertexBuffer is created with a [BufferDescription](xref:Veldrid.BufferDescription) object. The only info we need to provide is the size of our buffer. We have four vertices, and each are 24 bytes in size.

```C#
_vertexBuffer = factory.CreateVertexBuffer(new BufferDescription(4 * VertexPositionColor.SizeInBytes));
```

An IndexBuffer is created with an [IndexBufferDescription](xref:Veldrid.IndexBuffer) object, which is identical to a BufferDescription, except it also needs to know what the format of the index data is. In our case, the data is 16-bit unsigned integers.

```C#
_indexBuffer = factory.CreateIndexBuffer(new IndexBufferDescription(4 * sizeof(ushort), IndexFormat.UInt16));
```

We've created our buffers, but they are empty at the moment. We need to fill them with the data contained in our `quadVertices` and `indexData` arrays. Resource updates are done through our `CommandList`. Before we can do that, we need to call [Begin](xref:Veldrid.CommandList#Veldrid_CommandList_Begin):

```C#
_commandList.Begin();
```

We will use the [UpdateBuffer](xref:Veldrid.CommandList#Veldrid_CommandList_UpdateBuffer__1_Veldrid_Buffer_System_UInt32___0___) method to upload our data:

```C#
_commandList.UpdateBuffer(_vertexBuffer, 0, quadVertices);
_commandList.UpdateBuffer(_indexBuffer, 0, indexData);
```

These commands are simply recorded into the CommandList. To execute them on the GraphicsDevice, we need to [End](xref:Veldrid.CommandList#Veldrid_CommandList_End) our CommandList, and then call [GraphicsDevice.ExecuteCommands](xref:Veldrid.GraphicsDevice#Veldrid_GraphicsDevice_ExecuteCommands_Veldrid_CommandList_).

```C#
_commandList.End();
_graphicsDevice.ExecuteCommands(_commandList);
```

`_vertexBuffer` and `_indexBuffer` now contain all of the data from our arrays.

Creating a Pipeline requires that we know the layout of the VertexBuffer that will be used. Let's create a description for our VertexBuffer now.

```C#
VertexLayoutDescription vertexLayout = new VertexLayoutDescription(
    new VertexElementDescription("Position", VertexElementSemantic.Position, VertexElementFormat.Float2),
    new VertexElementDescription("Color", VertexElementSemantic.Color, VertexElementFormat.Float4));
```

Our vertex data has only two elements: a 2-float position, and a 4-float color.

To create a Pipeline, we also need a set of shaders. Shader code is a bit outside of the scope of this tutorial, so I have provided some pre-written shaders which can be used to draw our multi-colored quad. Copy [these assets](https://github.com/mellinoe/veldrid-samples/tree/master/src/GettingStarted/Shaders) into your project and set them to Copy to Output upon build. Included are shaders for all graphics backends. To load our [Veldrid.Shader](xref:Veldrid.Shader) objects, we will use a helper function which simply selects the appropriate file from the "Shaders" subdirectory and loads it into a Shader. For simplicity, the filename is assumed to be the name of the shader stage (vertex or fragment).

```C#
private static Shader LoadShader(ShaderStages stage)
{
    string extension = null;
    switch (_graphicsDevice.BackendType)
    {
        case GraphicsBackend.Direct3D11:
            extension = "hlsl.bytes";
            break;
        case GraphicsBackend.Vulkan:
            extension = "spv";
            break;
        case GraphicsBackend.OpenGL:
            extension = "glsl";
            break;
        default: throw new InvalidOperationException();
    }

    string path = Path.Combine(AppContext.BaseDirectory, "Shaders", $"{stage.ToString()}.{extension}");
    byte[] shaderBytes = File.ReadAllBytes(path);
    return _graphicsDevice.ResourceFactory.CreateShader(new ShaderDescription(stage, shaderBytes));
}
```

Next, we load our vertex and fragment shaders and create a [ShaderSetDescription](xref:Veldrid.ShaderSetDescription).

```C#
_vertexShader = LoadShader(ShaderStages.Vertex);
_fragmentShader = LoadShader(ShaderStages.Fragment);
ShaderStageDescription[] shaderStages =
{
    new ShaderStageDescription(ShaderStages.Vertex, _vertexShader, "VS"),
    new ShaderStageDescription(ShaderStages.Fragment, _fragmentShader, "FS")
};

ShaderSetDescription shaderSet = new ShaderSetDescription();
shaderSet.VertexLayouts = new VertexLayoutDescription[] { vertexLayout };
shaderSet.ShaderStages = shaderStages;
```

### Pipeline

The last object we need is a [Pipeline](xref:Veldrid.Pipeline). This is an object which encapsulates all of the necessary graphics state for drawing primitives. One piece of information is the set of shaders that will be used -- we have that already. There are several other pieces of information we need.

```C#
PipelineDescription pipelineDescription = new PipelineDescription();
pipelineDescription.BlendState = BlendStateDescription.SingleOverrideBlend;
```

The `BlendState` controls how the results from rendering are blended into the output textures. This is a simple example -- we only have a single output texture, and we want our new results to completely overwrite the existing data in our output.

```C#
pipelineDescription.DepthStencilState = new DepthStencilStateDescription(
    depthTestEnabled: true,
    depthWriteEnabled: true,
    comparisonKind: DepthComparisonKind.LessEqual);
```

In this example, the depth state is not very important -- we're only drawing a static 2D object. We've set up the depth-stencil state such that all depth testing and writing is enabled, anyways.

```C#
pipelineDescription.RasterizerState = new RasterizerStateDescription(
    cullMode: FaceCullMode.Back,
    fillMode: PolygonFillMode.Solid,
    frontFace: FrontFace.Clockwise,
    depthClipEnabled: true,
    scissorTestEnabled: false);
```

The rasterizer state lets you control various properties of the fixed-function rasterizer: which face gets culled, how that face is determined, how polygons are filled, and whether depth clipping and scissor testing are enabled. For this example, we've chosen the regular default values for everything.

```C#
pipelineDescription.PrimitiveTopology = PrimitiveTopology.TriangleStrip;
```

We are rendering our quad as a triangle strip. If we wanted to use a different topology, we would need to modify our index data. Index data of `{ 0, 1, 2, 0, 2, 3 }` would work for a triangle list.

```C#
pipelineDescription.ResourceLayouts = Array.Empty<ResourceLayout>();
```

Our shaders do not read from any resources, so we use an empty array here.

```C#
pipelineDescription.ShaderSet = new ShaderSetDescription(
    vertexLayouts: new VertexLayoutDescription[] { vertexLayout },
    shaderStages: shaderStages);
```

We're passing in our previously-created shader stages and vertex layout here. This controls which shaders are used for rendering when the Pipeline is active.

```C#
pipelineDescription.Outputs = _graphicsDevice.SwapchainFramebuffer.OutputDescription;
```

Every `Pipeline` in Veldrid needs to know how many outputs it has, and what the format of each is. Since we are going to be rendering directly to the application's swapchain, we will use the swapchain's [OutputDescription](xref:Veldrid.Framebuffer#Veldrid_Framebuffer_OutputDescription).

Finally, we can create the Pipeline.

```C#
_pipeline = factory.CreatePipeline(ref pipelineDescription);
```

We have successfully created all of the device resources that we need. In the next section, we will draw our quad, and then do some cleanup.

[Next: Part 3](xref:getting-started-part3)