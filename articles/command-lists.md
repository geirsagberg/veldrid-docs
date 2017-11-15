---
uid: command-lists
---

# Command Lists

Using Veldrid, no graphics commands can be executed without the use of a [CommandList](xref:Veldrid.CommandList). A CommandList is a special device resource (created by a [GraphicsDevice](xref:Veldrid.GraphicsDevice)) that records a variety of commands. These commands can then be submitted for execution on the device using the [GraphicsDevice.ExecuteCommands](xref:Veldrid.GraphicsDevice#Veldrid_GraphicsDevice_ExecuteCommands_Veldrid_CommandList_) method.

Several types of commands are available, depending on what work is being done.

## Resource Update

[Texture](xref:Veldrid.Texture) and [Buffer](xref:Veldrid.Buffer) objects can be updated directly in a CommandList, using the UpdateBuffer and UpdateTexture methods.

The color and depth targets of the active [Framebuffer](xref:Veldrid.Framebuffer) can be cleared.

Multisampled Textures can be resolved down into a regular non-multisampled Texture using the ResolveTexture method.

## State Changes

There are several state-change methods available, which control various pieces of state influencing Draw commands. The following state can be changed:
* [Framebuffer](xref:Veldrid.Framebuffer)
* [Viewports](xref:Veldrid.Viewport)
* Scissor rectangles
* [VertexBuffers](xref:Veldrid.VertexBuffer)
* [IndexBuffer](xref:Veldrid.IndexBuffer)
* [Pipeline](xref:Veldrid.Pipeline)
* [ResourceSets](xref:Veldrid.ResourceSet)

## Drawing

There is a single [Draw](xref:Veldrid.CommandList#Veldrid_CommandList_Draw_System_UInt32_System_UInt32_System_UInt32_System_Int32_System_UInt32_) method which can be invoked to record a draw command into the CommandList. The effect of this Draw is controlled by the active state of the CommandList -- which Pipeline, ResourceSets, and Framebuffer are bound, and what Viewport and scissor rectangles are set. Regular indexed and instanced draw commands are both supported by this method.