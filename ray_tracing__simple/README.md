﻿![logo](http://nvidianews.nvidia.com/_ir/219/20157/NV_Designworks_logo_horizontal_greenblack.png)

# NVIDIA Vulkan Ray Tracing Tutorial

By [Martin-Karl Lefrançois](https://devblogs.nvidia.com/author/mlefrancois/),
   [Pascal Gautron](https://devblogs.nvidia.com/author/pgautron/), Neil Bickford

The focus of this document and the provided code is to showcase a basic integration of
ray tracing within an existing Vulkan sample, using the
[`VK_NV_ray_tracing`](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VK_NV_ray_tracing) extension.
This tutorial starts from a basic Vulkan application and provides step-by-step instructions to modify and add methods and functions.
The sections are organized by components, with subsections identifying the modified functions.

![resultRaytraceShadowMedieval](images/resultRaytraceShadowMedieval.png)

## Purpose

This tutorial highlights the steps to add ray tracing to an existing Vulkan application, and assumes a working knowledge
of Vulkan in general. The code verbosity of classical components such as swapchain management, render passes etc. is
reduced using [C++ API helpers](https://github.com/nvpro-samples/shared_sources/tree/master/nvvkpp) and
NVIDIA's [nvpro-samples](https://github.com/nvpro-samples/build_all) framework. This framework contains many advanced examples
and best practices for Vulkan and OpenGL. We also use a helper for the creation of the ray tracing acceleration structures, but we will document
its contents extensively in this tutorial. The code is further simplified by using the
[Vulkan C++ API](https://github.com/KhronosGroup/Vulkan-Hpp), whose type safety and constructors
reduce both its verbosity and its potential for errors.

**Note**:  for  educational purposes all the code is contained in a very small set of files. A real
integration would require additional levels of abstraction.

# Environment Setup

To get support for `VK_NV_ray_tracing`, please install an [NVIDIA driver](http://www.nvidia.com/Download/index.aspx?lang=en-us)
with version 440.97 or later, and the [Vulkan SDK](http://vulkan.lunarg.com/sdk/home) version 1.1.126.0 or later.

You will also need to clone or download the following repositories:

* [shared_sources](https://github.com/nvpro-samples/shared_sources): The primary framework that all samples depend on.
* [shared_external](https://github.com/nvpro-samples/shared_external): Third party libraries that are provided pre-compiled, mostly for Windows x64 / MSVC.

The repository should be looking like:

```bash
-
|- shared_sources
|- shared_external
|- vk_raytracing_tutorial
  |- ray_tracing__before (not this directory, but the one to start from)
```

Run CMake in vk_raytracing_tutorial.

The starting project is a simple framework allowing us to load OBJ files and rasterize them
using Vulkan.

**Note:** The tutorial is based on the modification of ray_tracing__before, the directory ray_tracing__simple is the end result of this tutorial.

![original](images/resultRasterCube.png)

# Ray Tracing Setup

Go to the `main` function of the `main.cpp` file, and find where we request Vulkan extensions with `nvvkpp::ContextCreateInfo`.
To request ray tracing capabilities, we need to explicitly
add the [VK_NV_ray_tracing](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VK_NV_ray_tracing.html)
extension as well as its dependency
[VK_KHR_maintenance3](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VK_KHR_maintenance3.html):

```` C
// #VKRay: Activate the ray tracing extension
contextInfo.addDeviceExtension(VK_NV_RAY_TRACING_EXTENSION_NAME);
contextInfo.addDeviceExtension(VK_KHR_MAINTENANCE3_EXTENSION_NAME);
````

In the `HelloVulkan` class in `hello_vulkan.h`, add an initialization function and a member storing the capabilities of the GPU for ray tracing:

```` C
// #VKRay
void                                      initRayTracing();
vk::PhysicalDeviceRayTracingPropertiesNV  m_rtProperties;
````

At the end of `hello_vulkan.cpp`, add the body of `initRayTracing()`, which will query the ray tracing capabilities
of the GPU using this extension. In particular, it will obtain the maximum recursion depth,
ie. the number of nested ray tracing calls that can be performed from a single ray. This can be seen as the number
of times a ray can bounce in the scene in a recursive path tracer. Note that for performance purposes, recursion
should in practice be kept to a minimum, favoring a loop formulation. The shader header size will be useful when
creating the shader binding table in a later section.

```` C
//--------------------------------------------------------------------------------------------------
// Initialize Vulkan ray tracing
// #VKRay
void HelloVulkan::initRayTracing()
{
  // Requesting ray tracing properties
  auto properties = m_physicalDevice.getProperties2<vk::PhysicalDeviceProperties2,
                                                    vk::PhysicalDeviceRayTracingPropertiesNV>();
  m_rtProperties  = properties.get<vk::PhysicalDeviceRayTracingPropertiesNV>();
}
````

## main

In `main.cpp`, in the `main()` function, we call the initialization method right after `helloVk.updateDescriptorSet();`

```` C
// #VKRay
helloVk.initRayTracing();
````

**Exercise:**
> When running the program, you can put a breakpoint in the `initRayTracing()` method to inspect
    the resulting values. On a Quadro RTX 6000, the maximum recursion depth is 31, and the shader
    group handle size is 16.

# Acceleration Structure

To be efficient, ray tracing requires organizing the geometry into an acceleration structure (AS)
that will reduce the number of ray-triangle intersection tests during rendering.
This structure is divided into a two-level tree. Intuitively, this can directly map to the notion
of a simplified scene graph, in which the internal nodes of the graph have been collapsed into a single
transform matrix for each instance. The geometry of an instance is stored in a bottom-level acceleration structure (BLAS) object,
 which holds the actual vertex data. It is also possible to further simplify the scene graph by combining multiple objects within a single bottom-level AS: for that, a single BLAS can be built from multiple vertex buffers, each with its
 own transform matrix. Note that if an object is instantiated several times within a same BLAS, its geometry
 will be duplicated. This can be particularly useful for improving performance on static, non-instantiated
 scene components (as a rule of thumb, the fewer BLAS, the better).

The top-level AS (TLAS) will contain the object instances, each
with its own transformation matrix and reference to a corresponding BLAS.
We will start with a single bottom-level AS and a top-level AS instancing it once with an identity transform.

![Figure [step]: Acceleration Structure](images/AccelerationStructure.svg)

This sample loads an OBJ file and stores its indices, vertices and material data into an `ObjModel` structure. This
model is referenced by an `ObjInstance` structure which also contains the transformation matrix of that particular instance.
For ray tracing the `ObjModel` and `ObjInstance` will then naturally fit the BLAS and TLAS, respectively.

To simplify the ray tracing setup we use a helper class containing utility functions for acceleration structure builds.
In the header file, include the`raytrace_vkpp` helper

```` C
// #VKRay
#define ALLOC_DEDICATED
#include "nvvkpp/raytrace_vkpp.hpp"
````

so that we can add that helper as a member in the `HelloVulkan` class,

```` C
nvvkpp::RaytracingBuilder m_rtBuilder;
````

and initialize it at the end of `initRaytracing()`:

```` C
m_rtBuilder.setup(m_device, m_physicalDevice, m_graphicsQueueIndex);
````

## Bottom-Level Acceleration Structure

The first step of building a BLAS object consists in converting the geometry data of an `ObjModel` into a `vk::GeometryNV` object than can be used by the AS builder. Add a new method to the `HelloVulkan`
class:

```` C
  vk::GeometryNV objectToVkGeometryNV(const ObjModel& model);
````

Its implementation simply maps the buffers containing the vertex and index data, and specifies its sizes,
offsets and formats. Note that we consider all objects opaque for now, and indicate this to the builder for
potential optimization.

```` C
//--------------------------------------------------------------------------------------------------
// Converting a OBJ primitive to the ray tracing geometry used for the BLAS
//
vk::GeometryNV HelloVulkan::objectToVkGeometryNV(const ObjModel& model)
{
  vk::GeometryTrianglesNV triangles;
  triangles.setVertexData(model.vertexBuffer.buffer);
  triangles.setVertexOffset(0);  // Start at the beginning of the buffer
  triangles.setVertexCount(model.nbVertices);
  triangles.setVertexStride(sizeof(VertexObj));
  triangles.setVertexFormat(vk::Format::eR32G32B32Sfloat);  // 3xfloat32 for vertices
  triangles.setIndexData(model.indexBuffer.buffer);
  triangles.setIndexOffset(0 * sizeof(uint32_t));
  triangles.setIndexCount(model.nbIndices);
  triangles.setIndexType(vk::IndexType::eUint32);  // 32-bit indices
  vk::GeometryDataNV geoData;
  geoData.setTriangles(triangles);
  vk::GeometryNV geometry;
  geometry.setGeometry(geoData);
  // Consider the geometry opaque for optimization
  geometry.setFlags(vk::GeometryFlagBitsNV::eOpaque);
  return geometry;
}
````

In the `HelloVulkan` class declaration, we can now add the `createBottomLevelAS()` method that will generate a `vk::GeometryNV` for each object, and trigger a BLAS build:

```` C
void createBottomLevelAS();
````

The implementation loops over all the loaded models and fills in an array of `vk::GeometryNV` before
triggering a build of all BLAS's in a batch. The resulting acceleration structures will be stored
within the helper in the order of construction, so that they can be directly referenced by index later.

```` C
void HelloVulkan::createBottomLevelAS()
{
  // BLAS - Storing each primitive in a geometry
  std::vector<std::vector<vk::GeometryNV>> blas;
  blas.reserve(m_objModel.size());
  for (const auto& obj : m_objModel)
  {
    auto geo = objectToVkGeometryNV(obj);
    // We could add more geometry in each BLAS, but we add only one for now
    blas.push_back({geo});
  }
  m_rtBuilder.buildBlas(blas, vk::BuildAccelerationStructureFlagBitsNV::ePreferFastTrace);
}
````

Note that the builder takes a vector-of-vectors `std::vector<std::vector<vk::GeometryNV>>`: this is due to the ability of the builder to merge multiple geometries into a single BLAS. In this simple example we only use one OBJ file, and hence generate one BLAS per OBJ.

### Helper Details: `RaytracingBuilder::buildBlas()`

This helper function is already present in `raytracing_vkpp.hpp`: it can be reused in many projects, and is
part of the set of helpers provided by the [nvpro-samples](https://github.com/nvpro-samples). The function
will generate one BLAS for each `std::vector<vk::GeometryNV>`:

```` C
void buildBlas(const std::vector<std::vector<vk::GeometryNV>>& geoms,
                vk::BuildAccelerationStructureFlagsNV           flags =
                    vk::BuildAccelerationStructureFlagBitsNV::ePreferFastTrace)
{
  m_blas.resize(geoms.size());

  vk::DeviceSize maxScratch{0};

  // Iterate over the groups of geometries, creating one BLAS for each group
  for(size_t i = 0; i < geoms.size(); i++)
  {
````

The `vk::GeometryNV` objects are assembled into a creation information object to be passed to the
builder.

```` C
    Blas& blas{m_blas[i]};

    // Set the geometries that will be part of the BLAS
    blas.asInfo.setGeometryCount(static_cast<uint32_t>(geoms[i].size()));
    blas.asInfo.setPGeometries(geoms[i].data());
    blas.asInfo.flags = flags;
    vk::AccelerationStructureCreateInfoNV createinfo{0, blas.asInfo};
````

The creation information is then passed to the allocator, that will internally create an acceleration structure handle. It will also query `vk::Device::getAccelerationStructureMemoryRequirementsNV` to obtain the size of the resulting BLAS, and allocate memory accordingly.

```` C
    // Create an acceleration structure identifier and allocate memory to store the
    // resulting structure data
    blas.as = m_alloc.createAcceleration(createinfo);
    m_debug.setObjectName(blas.as.accel, (std::string("Blas" + std::to_string(i)).c_str()));
````

The acceleration structure builder requires some scratch memory to generate the BLAS. Since we generate all the
BLAS's in a batch, we query the scratch memory requirements for each BLAS, and find the maximum such requirement.

```` C
    // Estimate the amount of scratch memory required to build the BLAS, and update the
    // size of the scratch buffer that will be allocated to sequentially build all BLAS's
    vk::AccelerationStructureMemoryRequirementsInfoNV memoryRequirementsInfo{
        vk::AccelerationStructureMemoryRequirementsTypeNV::eBuildScratch, blas.as.accel};
    vk::DeviceSize scratchSize =
        m_device.getAccelerationStructureMemoryRequirementsNV(memoryRequirementsInfo)
            .memoryRequirements.size;

    maxScratch = std::max(maxScratch, scratchSize);
  }
````

Once that maximum has been found, we allocate a scratch buffer.

```` C
  // Allocate the scratch buffers holding the temporary data of the acceleration structure builder
  nvvkBuffer scratchBuffer =
      m_alloc.createBuffer(maxScratch, vk::BufferUsageFlagBits::eRayTracingNV);
````

We then use a one-time command buffer to launch all the BLAS builds. Note the barrier after each
build call: this is required as we reuse the scratch space across builds, and hence need to ensure
the previous build has completed before starting the next.

```` C
  // Create a command buffer containing all the BLAS builds
  nvvkpp::SingleCommandBuffer genCmdBuf(m_device, m_queueIndex);
  vk::CommandBuffer           cmdBuf = genCmdBuf.createCommandBuffer();
  for(auto& blas : m_blas)
  {
    cmdBuf.buildAccelerationStructureNV(blas.asInfo, nullptr, 0, VK_FALSE, blas.as.accel, nullptr,
                                        scratchBuffer.buffer, 0);
    // Since the scratch buffer is reused across builds, we need a barrier to ensure one build
    // is finished before starting the next one
    vk::MemoryBarrier barrier(vk::AccessFlagBits::eAccelerationStructureWriteNV,
                              vk::AccessFlagBits::eAccelerationStructureReadNV);
    cmdBuf.pipelineBarrier(vk::PipelineStageFlagBits::eAccelerationStructureBuildNV,
                            vk::PipelineStageFlagBits::eAccelerationStructureBuildNV,
                            vk::DependencyFlags(), {barrier}, {}, {});
  }
````

 While this approach has the advantage of keeping all BLAS's independent, building many BLAS's efficiently would
 require allocating a larger scratch buffer, and launch several builds simultaneously. This tutorial also
 does not use compaction, which could reduce significantly the memory footprint of the acceleration structures. Both
 of those aspects will be part of a future advanced tutorial.

We finally execute the command buffer and clean up the allocator's scratch memory and staging buffer:

```` C
  genCmdBuf.flushCommandBuffer(cmdBuf);
  m_alloc.destroy(scratchBuffer);
  m_alloc.flushStaging();
}
````

## Top-Level Acceleration Structure

The TLAS is the entry point in the ray tracing scene description, and stores all the instances. Add a new method
to the `HelloVulkan` class:

```` C
void createTopLevelAS();
````

An instance is represented by a `nvvkpp::RaytracingBuilder::Instance`, which stores its transform matrix (`transform`) and
the identifier of its corresponding BLAS (`blasId`). It also contains an instance identifier that will be available
during shading as `gl_InstanceID`, as well as the index of the hit group that represents the shaders that will be invoked upon hitting the object (`hitGroupId`).
This index and the notion of hit group are tied to the definition of the ray tracing pipeline and the Shader Binding Table, described later in this tutorial. For now
it suffices to say that we will use only one hit group for the whole scene, and hence the hit group index is always 0.
Finally, the instance may indicate culling preferences, such as backface culling, using its `vk::GeometryInstanceFlagsNV flags` member. In our example we decide to disable culling altogether
 for simplicity and independence on the winding of the input models.

Once all the instance objects are created we trigger the TLAS build, directing the builder to prefer generating a TLAS optimized for tracing performance (rather than AS size, for example).

```` C
void HelloVulkan::createTopLevelAS()
{
  std::vector<nvvkpp::RaytracingBuilder::Instance> tlas;
  tlas.reserve(m_objInstance.size());
  for(int i = 0; i < static_cast<int>(m_objInstance.size()); i++)
  {
    nvvkpp::RaytracingBuilder::Instance rayInst;
    rayInst.transform  = m_objInstance[i].transform;  // Position of the instance
    rayInst.instanceId = i;                           // gl_InstanceID
    rayInst.blasId     = m_objInstance[i].objIndex;
    rayInst.hitGroupId = 0;  // We will use the same hit group for all objects
    rayInst.flags      = vk::GeometryInstanceFlagBitsNV::eTriangleCullDisable;
    tlas.emplace_back(rayInst);
  }
  m_rtBuilder.buildTlas(tlas, vk::BuildAccelerationStructureFlagBitsNV::ePreferFastTrace);
}
````

As usual in Vulkan, we need to explicitly destroy the objects we created by adding a call at the end of `HelloVulkan::destroyResources`:

```` C
  // #VKRay
  m_rtBuilder.destroy();
````

### Helper Details: `RaytracingBuilder::buildTlas()`

The helper function for building top-level acceleration structures is part of the [nvpro-samples](https://github.com/nvpro-samples)
and builds a TLAS from a vector of `Instance` objects. We first store some basic information about the TLAS, namely
the number of instances it will hold, and flags indicating preferences for the builder, such as whether to prefer faster builds or better performance.

```` C
void buildTlas(const std::vector<Instance>&          instances,
                 vk::BuildAccelerationStructureFlagsNV flags =
                     vk::BuildAccelerationStructureFlagBitsNV::ePreferFastTrace)
  {
    // Set the instance count required to determine how much memory the TLAS will use
    m_tlas.asInfo.instanceCount = static_cast<uint32_t>(instances.size());
    m_tlas.asInfo.flags         = flags;
    vk::AccelerationStructureCreateInfoNV accelerationStructureInfo{0, m_tlas.asInfo};
````

We then call the allocator, which will create an acceleration structure handle for the TLAS. It will also query the
resulting size of the TLAS using `vk::Device::getAccelerationStructureMemoryRequirementsNV` and allocate that
 amount of memory:

```` C
    // Create the acceleration structure object and allocate the memory required to hold the TLAS data
    m_tlas.as = m_alloc.createAcceleration(accelerationStructureInfo);
    m_debug.setObjectName(m_tlas.as.accel, "Tlas");
````

As with the BLAS, we also query the amount of scratch memory required by the builder to generate the TLAS,
and allocate a scratch buffer. Note that since the BLAS and TLAS both require a scratch buffer, we could also have used
one buffer and thus saved an allocation. However, for the purpose of this tutorial, we keep the BLAS and TLAS builds independent.

```` C
    // Compute the amount of scratch memory required by the acceleration structure builder
    vk::AccelerationStructureMemoryRequirementsInfoNV memoryRequirementsInfo{
        vk::AccelerationStructureMemoryRequirementsTypeNV::eBuildScratch, m_tlas.as.accel};
    vk::DeviceSize scratchSize =
        m_device.getAccelerationStructureMemoryRequirementsNV(memoryRequirementsInfo)
            .memoryRequirements.size;

    // Allocate the scratch memory
    nvvkBuffer scratchBuffer =
        m_alloc.createBuffer(scratchSize, vk::BufferUsageFlagBits::eRayTracingNV);
````

An `Instance` object is nearly identical to a `VkGeometryInstanceNV` object: the only difference is the transform
matrix of the instance. The former uses a $4\times4$ matrix from GLM (column-major), while the latter uses a raw
array of floating-point values representing a row-major $4\times3$ matrix. Using the `Instance` object on the application
side allows us to use the more intuitive $4\times4$ matrices, making the code clearer. When generating the TLAS we then convert
 all the `Instance` objects to `VkGeometryInstanceNV`:

```` C
    // For each instance, build the corresponding instance descriptor
    std::vector<VkGeometryInstanceNV> geometryInstances;
    geometryInstances.reserve(instances.size());
    for(const auto& inst : instances)
    {
      geometryInstances.push_back(instanceToVkGeometryInstanceNV(inst));
    }
````

We then upload the instance descriptions to the device using a one-time command buffer. This command buffer will also be used to generate the TLAS itself, and so we add a barrier after the copy to ensure it has completed before launching the TLAS build.

```` C
    // Building the TLAS
    nvvkpp::SingleCommandBuffer genCmdBuf(m_device, m_queueIndex);
    vk::CommandBuffer           cmdBuf = genCmdBuf.createCommandBuffer();

    // Create a buffer holding the actual instance data for use by the AS builder
    VkDeviceSize instanceDescsSizeInBytes = instances.size() * sizeof(VkGeometryInstanceNV);

    // Allocate the instance buffer and copy its contents from host to device memory
    m_instBuffer =
        m_alloc.createBuffer(cmdBuf, geometryInstances, vk::BufferUsageFlagBits::eRayTracingNV);
    m_debug.setObjectName(m_instBuffer.buffer, "TLASInstances");

    // Make sure the copy of the instance buffer are copied before triggering the
    // acceleration structure build
    vk::MemoryBarrier barrier(vk::AccessFlagBits::eTransferWrite,
                              vk::AccessFlagBits::eAccelerationStructureWriteNV);
    cmdBuf.pipelineBarrier(vk::PipelineStageFlagBits::eTransfer,
                           vk::PipelineStageFlagBits::eAccelerationStructureBuildNV,
                           vk::DependencyFlags(), {barrier}, {}, {});
````

The build is then triggered, and we execute the command buffer before destroying the temporary buffers.

```` C
    // Build the TLAS
    cmdBuf.buildAccelerationStructureNV(m_tlas.asInfo, m_instBuffer.buffer, 0, VK_FALSE,
                                        m_tlas.as.accel, nullptr, scratchBuffer.buffer, 0);

    genCmdBuf.flushCommandBuffer(cmdBuf);

    m_alloc.flushStaging();

    m_alloc.destroy(scratchBuffer);
  }
````

## main

In the `main` function, we can now add the creation of the geometry instances and acceleration structures
 right after initializing ray tracing:

```` C
// #VKRay
helloVk.initRayTracing();
helloVk.createBottomLevelAS();
helloVk.createTopLevelAS();
````

# Ray Tracing Descriptor Set

The ray tracing shaders, like the rasterization shaders, use external resources referenced by a descriptor set. A key difference,
however, is that in a scene requiring several types of shaders, the rasterization would allow each set of shaders to have their own
descriptor set(s). For example, objects with different materials may each have a descriptor set containing the handles of the textures
it needs. This is easily done since for a given material, we would create its corresponding rasterization pipeline and use that pipeline
to render all the objects with that material. On the contrary, with ray tracing it is not possible to know in advance which objects will
be hit by a ray, so any shader may be invoked at any time. The Vulkan ray tracing extension then uses a single set of descriptor sets containing
all the resources necessary to render the scene: for example, it would contain all the textures for all the materials.

To maintain compatibility between rasterization and ray tracing, the ray tracing pipeline will use the same descriptor set containing the scene information,
and will add another descriptor set referencing the TLAS and the buffer in which we store the output image.

In the header, we declare the objects related to this additional descriptor set:

```` C
  void           createRtDescriptorSet();

  std::vector<vk::DescriptorSetLayoutBinding>        m_rtDescSetLayoutBind;
  vk::DescriptorPool                                 m_rtDescPool;
  vk::DescriptorSetLayout                            m_rtDescSetLayout;
  vk::DescriptorSet                                  m_rtDescSet;
````

The acceleration structure will be accessible by the Ray Generation shader, as
we want to call `TraceNV()` from this shader. Later in this document, we will
also make it accessible from the Closest Hit shader, in order to send rays from there as well.
 The output image is the offscreen buffer used by the
rasterization, and will be written only by the RayGen shader.

```` C
//--------------------------------------------------------------------------------------------------
// This descriptor set holds the Acceleration structure and the output image
//
void HelloVulkan::createRtDescriptorSet()
{
  using vkDT   = vk::DescriptorType;
  using vkSS   = vk::ShaderStageFlagBits;
  using vkDSLB = vk::DescriptorSetLayoutBinding;

  m_rtDescSetLayoutBind.emplace_back(
      vkDSLB(0, vkDT::eAccelerationStructureNV, 1, vkSS::eRaygenNV ));  // TLAS
  m_rtDescSetLayoutBind.emplace_back(
      vkDSLB(1, vkDT::eStorageImage, 1, vkSS::eRaygenNV));  // Output image

  m_rtDescPool      = nvvkpp::util::createDescriptorPool(m_device, m_rtDescSetLayoutBind);
  m_rtDescSetLayout = nvvkpp::util::createDescriptorSetLayout(m_device, m_rtDescSetLayoutBind);
  m_rtDescSet       = m_device.allocateDescriptorSets({m_rtDescPool, 1, &m_rtDescSetLayout})[0];

  vk::WriteDescriptorSetAccelerationStructureNV descASInfo;
  descASInfo.setAccelerationStructureCount(1);
  descASInfo.setPAccelerationStructures(&m_rtBuilder.getAccelerationStructure());
  vk::DescriptorImageInfo imageInfo{
      {}, m_offscreenColor.descriptor.imageView, vk::ImageLayout::eGeneral};

  std::vector<vk::WriteDescriptorSet> writes;
  writes.emplace_back(
      nvvkpp::util::createWrite(m_rtDescSet, m_rtDescSetLayoutBind[0], &descASInfo));
  writes.emplace_back(nvvkpp::util::createWrite(m_rtDescSet, m_rtDescSetLayoutBind[1], &imageInfo));
  m_device.updateDescriptorSets(static_cast<uint32_t>(writes.size()), writes.data(), 0, nullptr);
}
````

## Additions to the Scene Descriptor Set

As the ray tracing shaders also have to access the scene description, we need to extend the access flags of the corresponding
 buffers in the original `createDescriptorSetLayout()`. The
RayGen should access the camera matrices to compute ray directions, and the ClosestHit needs access to the materials, scene instances, textures, vertex buffers,
and index buffers. Even though the vertex and index buffers will only be used by the ray tracing shaders we add them to this descriptor
set as they semantically fit the Scene descriptor set.

```` C
  // Camera matrices (binding = 0)
  m_descSetLayoutBind.emplace_back(
      vkDS(0, vkDT::eUniformBuffer, 1, vkSS::eVertex | vkSS::eRaygenNV));
  // Materials (binding = 1)
  m_descSetLayoutBind.emplace_back(
      vkDS(1, vkDT::eStorageBuffer, nbObj, vkSS::eVertex | vkSS::eFragment | vkSS::eClosestHitNV));
  // Scene description (binding = 2)
  m_descSetLayoutBind.emplace_back(  //
      vkDS(2, vkDT::eStorageBuffer, 1, vkSS::eVertex | vkSS::eFragment | vkSS::eClosestHitNV));
  // Textures (binding = 3)
  m_descSetLayoutBind.emplace_back(
      vkDS(3, vkDT::eCombinedImageSampler, nbTxt, vkSS::eFragment | vkSS::eClosestHitNV));
  // Materials (binding = 4)
  m_descSetLayoutBind.emplace_back(
      vkDS(4, vkDT::eStorageBuffer, nbObj, vkSS::eFragment | vkSS::eClosestHitNV));
  // Storing vertices (binding = 5)
  m_descSetLayoutBind.emplace_back(  //
      vkDS(5, vkDT::eStorageBuffer, nbObj, vkSS::eClosestHitNV));
  // Storing indices (binding = 6)
  m_descSetLayoutBind.emplace_back(  //
      vkDS(6, vkDT::eStorageBuffer, nbObj, vkSS::eClosestHitNV));
````

We set the actual contents of the descriptor set by adding those buffers in `updateDescriptorSet()`:

```` C
  // All material buffers, 1 buffer per OBJ
  std::vector<vk::DescriptorBufferInfo> dbiMat;
  std::vector<vk::DescriptorBufferInfo> dbiMatIdx;
  std::vector<vk::DescriptorBufferInfo> dbiVert;
  std::vector<vk::DescriptorBufferInfo> dbiIdx;
  for(size_t i = 0; i < m_objModel.size(); ++i)
  {
    dbiMat.push_back({m_objModel[i].matColorBuffer.buffer, 0, VK_WHOLE_SIZE});
    dbiMatIdx.push_back({m_objModel[i].matIndexBuffer.buffer, 0, VK_WHOLE_SIZE});
    dbiVert.push_back({m_objModel[i].vertexBuffer.buffer, 0, VK_WHOLE_SIZE});
    dbiIdx.push_back({m_objModel[i].indexBuffer.buffer, 0, VK_WHOLE_SIZE});
  }
  writes.emplace_back(nvvkpp::util::createWrite(m_descSet, m_descSetLayoutBind[1], dbiMat.data()));
  writes.emplace_back(
      nvvkpp::util::createWrite(m_descSet, m_descSetLayoutBind[4], dbiMatIdx.data()));
  writes.emplace_back(nvvkpp::util::createWrite(m_descSet, m_descSetLayoutBind[5], dbiVert.data()));
  writes.emplace_back(nvvkpp::util::createWrite(m_descSet, m_descSetLayoutBind[6], dbiIdx.data()));
````

Originally the buffers containing the vertices and indices were only used by the rasterization
pipeline. The ray tracing will need to use those buffers as storage buffers. We update the usage of
the buffers in `loadModel`:

```` C
  model.vertexBuffer =
      m_alloc.createBuffer(cmdBuf, loader.m_vertices, vkBU::eVertexBuffer | vkBU::eStorageBuffer);
  model.indexBuffer =
      m_alloc.createBuffer(cmdBuf, loader.m_indices, vkBU::eIndexBuffer | vkBU::eStorageBuffer);
````

### **Note: Array of Buffers**

> Each model (OBJ) was constructed with a buffer of vertices, indices, and materials. Therefore the
    scene has vectors of those buffers. In the shaders, we access the right buffer using the
    the ObjectID used by the Instance. This is convenient, as we have access to all the data
    of the scene while ray tracing.

## Descriptor Update

As with the rasterization descriptor set, the ray tracing descriptor set needs to be updated if its contents change. This typically
happens when resizing the window, as the output image is recreated and needs to be re-linked to the descriptor set.
The update is performed in a new method of the `HelloVulkan` class:

```` C
void updateRtDescriptorSet();
````

The implementation is straightforward, simply updating the output image reference:

```` C
//--------------------------------------------------------------------------------------------------
// Writes the output image to the descriptor set
// - Required when changing resolution
//
void HelloVulkan::updateRtDescriptorSet()
{
  using vkDT = vk::DescriptorType;

  // (1) Output buffer
  vk::DescriptorImageInfo imageInfo{
      {}, m_offscreenColor.descriptor.imageView, vk::ImageLayout::eGeneral};
  vk::WriteDescriptorSet wds{m_rtDescSet, 1, 0, 1, vkDT::eStorageImage, &imageInfo};
  m_device.updateDescriptorSets(wds, nullptr);
}
````

We can then add the update call to the `onResize()` method to link it to the resizing event:

```` C
  updateRtDescriptorSet();
````

The resources created in this section need to be destroyed when closing the application by adding the following to
`destroyResources`:

```` C
  m_device.destroy(m_rtDescPool);
  m_device.destroy(m_rtDescSetLayout);
````

## main

In the `main` function, we create the descriptor set after the other ray tracing calls:

```` C
  helloVk.createRtDescriptorSet();
````

# Ray Tracing Pipeline

When creating rasterization shaders with Vulkan, the application compiles them into executable shaders, which are bound to the
rasterization pipeline. All objects rendered using this pipeline will use those shaders. To render an image with several types of
shaders, the rasterization pipeline needs to be set to use each before calling the draw commands.

In a ray tracing context, a ray traced through the scene can hit any object and thus trigger the execution of any shader. Instead of using one shader
executable at a time, we now need to have all shaders available at once. The pipeline then contains all the shaders required to render the scene,
 and information on how to execute it. To be able to ray trace some geometry, the Vulkan ray tracing extension typically uses at least these 3 shader programs:

* The **ray generation** shader will be the starting point for ray tracing, and will be called for each pixel. It will typically initialize a ray
  starting at the location of the camera, in a direction given by evaluating the camera lens model at the pixel location.
  It will then invoke `traceNV()`, that will shoot the ray in the scene. Other shaders below will process further events, and return
  their result to the ray generation shader through the ray payload.
* The **miss** shader is executed when a ray does not intersect any geometry. For instance, it might sample an environment map, or return a simple color
  through the ray payload.
* The **closest hit** shader is called upon hitting the geometric instance closest to the starting point of the ray. This shader can for example
  perform lighting calculations and return the results through the ray payload. There can be as many closest hit shaders as needed, much like how a rasterization-based application has multiple pixel shaders depending on its objects.

Two more shader types can optionally be used:

* The **intersection** shader, which allows intersecting user-defined geometry. For example, this can be used to intersect geometry placeholders for on-demand geometry loading, or intersecting
  procedural geometry without tessellating them beforehand.
  Using this shader requires modifying how the acceleration structures are built, and is beyond the scope of this tutorial.
  We will instead rely on the built-in triangle intersection shader provided by the extension, which returns 2 floating-point
  values representing the barycentric coordinates `(u,v)` of the hit point inside the triangle. For a triangle made of vertices `v0`, `v1`, `v2`, the barycentric coordinates define the weights of the vertices as follows:

```` bash
            . u
           / \
          / v1\
         /     \
        /       \
 1-u-v / v0   v2 \ v
      '-----------'
````

* The **any hit** shader is executed on each potential intersection: when searching for the hit point closest to the ray origin, several candidates
  may be found on the way. The any hit shader can frequently be used to efficiently implement alpha-testing. If the alpha test fails, the ray traversal can continue without having to call `traceNV()` again. The built-in any hit shader is simply a pass-through returning the intersection to the traversal engine, which will determine which ray intersection is the closest.

![Figure [step]: The Ray Tracing Pipeline](images/ShaderPipeline.svg)

We will start with a pipeline containing only the 3 main shader programs: a single ray generation shader, a single miss shader, and a single hit group made only of a closest
hit shader. This is done by first compiling each GLSL shader program into SPIR-V. These SPIR-V shaders will be linked
together into a ray tracing pipeline, which will be able to route the intersection calculations to the right hit shaders.

To be able to focus on the pipeline generation, we provide simple shaders:

## Adding Shaders

### Note: [Download Ray Tracing Shaders](files/shaders.zip)

> Download the shaders and extract the content into `src/shaders`. Then rerun CMake, which will add those files to the project.

The `shaders` folder now contains 3 more files:

* `raytrace.rgen` contains the ray generation program. It also declares its access to the ray tracing output buffer `image`, and the ray tracing acceleration structure `topLevelAS`, bound as an `accelerationStructureNV`. For now this shader program simply writes a constant color into the output buffer.
* `raytrace.rmiss` defines the miss shader. This shader will be executed when no geometry is hit, and will
  write a constant color into the ray payload `rayPayloadInNV`, which is provided automatically. Since our current ray generation program does not trace any rays for now, this shader will not be called.
* `raytrace.rchit` contains a very simple closest hit shader. It will be executed upon
  hitting the geometry (our triangles). As the miss shader, it takes the ray payload `rayPayloadInNV`. It also has a second
  input defining the intersection attributes `hitAttributeNV` as provided by the intersection shader, i.e. the barycentric coordinates. This shader simply writes a constant color to the payload.

In the header file, let's add the definition of the ray tracing pipeline building method, and the storage members of the pipeline:

```` C
void                                               createRtPipeline();
std::vector<vk::RayTracingShaderGroupCreateInfoNV> m_rtShaderGroups;
vk::PipelineLayout                                 m_rtPipelineLayout;
vk::Pipeline                                       m_rtPipeline;
````

The pipeline will also use push constants to store global uniform values, namely the background color and
the light source information:

```` C
  struct RtPushConstant
  {
    nvmath::vec4f clearColor;
    nvmath::vec3f lightPosition;
    float         lightIntensity;
    int           lightType;
  } m_rtPushConstants;
````

Our implementation of the ray tracing pipeline generation starts by adding the ray generation and miss shader stages, followed
by the closest hit shader. Note that this order is arbitrary, as the extension allows the developer to set up the pipeline in any order.

All stages are stored in an array of `vk::PipelineShaderStageCreateInfo` objects. Indices within this vector
will be used as unique identifiers for the shaders in the Shader Binding Table. These identifiers are stored in the
`RayTracingShaderGroupCreateInfoNV` structure. This structure first specifies a `type`, which represents the kind of shader group represented in the structure. Ray generation, miss shaders are called 'general' shaders. In this case the type is `eGeneral`, and only the `generalShader` member of the structure is filled. The other ones are set to `VK_SHADER_UNUSED_NV`. This is also the case for the callable shaders, not used in this tutorial.
In our layout the ray generation comes first (0), followed by the miss shader (1).

```` C
//--------------------------------------------------------------------------------------------------
// Pipeline for the ray tracer: all shaders, raygen, chit, miss
//
void HelloVulkan::createRtPipeline()
{
  std::vector<std::string> paths = defaultSearchPaths;

  vk::ShaderModule raygenSM =
      nvvkpp::util::createShaderModule(m_device,  //
                                       nvh::loadFile("shaders/raytrace.rgen.spv", true, paths));
  vk::ShaderModule missSM =
      nvvkpp::util::createShaderModule(m_device,  //
                                       nvh::loadFile("shaders/raytrace.rmiss.spv", true, paths));

  std::vector<vk::PipelineShaderStageCreateInfo> stages;

  // Raygen
  vk::RayTracingShaderGroupCreateInfoNV rg{vk::RayTracingShaderGroupTypeNV::eGeneral,
                                           VK_SHADER_UNUSED_NV, VK_SHADER_UNUSED_NV,
                                           VK_SHADER_UNUSED_NV, VK_SHADER_UNUSED_NV};
  stages.push_back({{}, vk::ShaderStageFlagBits::eRaygenNV, raygenSM, "main"});
  rg.setGeneralShader(static_cast<uint32_t>(stages.size() - 1));
  m_rtShaderGroups.push_back(rg);
  // Miss
  vk::RayTracingShaderGroupCreateInfoNV mg{vk::RayTracingShaderGroupTypeNV::eGeneral,
                                           VK_SHADER_UNUSED_NV, VK_SHADER_UNUSED_NV,
                                           VK_SHADER_UNUSED_NV, VK_SHADER_UNUSED_NV};
  stages.push_back({{}, vk::ShaderStageFlagBits::eMissNV, missSM, "main"});
  mg.setGeneralShader(static_cast<uint32_t>(stages.size() - 1));
  m_rtShaderGroups.push_back(mg);

````

As detailed before, intersections are managed by 3 kinds of shaders: the intersection shader computes the
ray-geometry intersections, the any-hit shader is run for every potential intersection, and the closest hit shader
is applied to the closest hit point along the ray. Those 3 shaders are bound into a hit group. In our case the geometry
is made of triangles, so the `type` of the `RayTracingShaderGroupCreateInfoNV` is `eTrianglesHitGroup`. The intersection
shader is then built-in, and we set the `intersectionShader` member to `VK_SHADER_UNUSED_NV`. We do not use a any-hit shader,
letting the system use a built-in pass-through shader. Therefore, we also leave the `anyHitShader` to `VK_SHADER_UNUSED_NV`.
The only shader we define is then the closest hit shader, by setting the `closestHitShader` member to the index `2` (`stages.size()-1`),
since the `stages` vector already contains the ray generation and miss shaders.

```` C
  // Hit Group - Closest Hit + AnyHit
  vk::ShaderModule chitSM =
      nvvkpp::util::createShaderModule(m_device,  //
                                       nvh::loadFile("shaders/raytrace.rchit.spv", true, paths));

  vk::RayTracingShaderGroupCreateInfoNV hg{vk::RayTracingShaderGroupTypeNV::eTrianglesHitGroup,
                                           VK_SHADER_UNUSED_NV, VK_SHADER_UNUSED_NV,
                                           VK_SHADER_UNUSED_NV, VK_SHADER_UNUSED_NV};
  stages.push_back({{}, vk::ShaderStageFlagBits::eClosestHitNV, chitSM, "main"});
  hg.setClosestHitShader(static_cast<uint32_t>(stages.size() - 1));
  m_rtShaderGroups.push_back(hg);
````

Note that if the geometry were not triangles, we would have set the `type` to `eProceduralHitGroup`, and would have to define an intersection shader.

After creating the shader groups, we need to setup the pipeline layout that will describe how the pipeline
will access external data:

```` C
  vk::PipelineLayoutCreateInfo pipelineLayoutCreateInfo;
````

We first add the push constant range to allow the ray tracing shaders to access the global uniform values:  

```` C
  // Push constant: we want to be able to update constants used by the shaders
  vk::PushConstantRange pushConstant{vk::ShaderStageFlagBits::eRaygenNV
                                         | vk::ShaderStageFlagBits::eClosestHitNV
                                         | vk::ShaderStageFlagBits::eMissNV,
                                     0, sizeof(RtPushConstant)};
  pipelineLayoutCreateInfo.setPushConstantRangeCount(1);
  pipelineLayoutCreateInfo.setPPushConstantRanges(&pushConstant);
````

As described earlier, the pipeline uses two descriptor sets: `set=0` is specific to the ray tracing pipeline (TLAS and output image), and `set=1` is shared with the rasterization (scene data):

```` C
  // Descriptor sets: one specific to ray tracing, and one shared with the rasterization pipeline
  std::vector<vk::DescriptorSetLayout> rtDescSetLayouts = {m_rtDescSetLayout, m_descSetLayout};
  pipelineLayoutCreateInfo.setSetLayoutCount(static_cast<uint32_t>(rtDescSetLayouts.size()));
  pipelineLayoutCreateInfo.setPSetLayouts(rtDescSetLayouts.data());
````

The pipeline layout information is now complete, allowing us to create the layout itself.

```` C  
  m_rtPipelineLayout = m_device.createPipelineLayout(pipelineLayoutCreateInfo);
````

The creation of the ray tracing pipeline is different from the classical graphics pipeline. In the graphics pipeline we simply need to fill in the fixed set of programmable stages (vertex, fragment, etc.). The ray tracing pipeline can contain an arbitrary number of stages depending on the number of active shaders in the scene.

We first provide all the stages that will be used:

```` C
  // Assemble the shader stages and recursion depth info into the ray tracing pipeline
  vk::RayTracingPipelineCreateInfoNV rayPipelineInfo;
  rayPipelineInfo.setStageCount(static_cast<uint32_t>(stages.size()));  // Stages are shaders
  rayPipelineInfo.setPStages(stages.data());
````

Then, we indicate how the shaders can be assembled into groups. A ray generation or miss shader is a group by
itself, but hit groups can comprise up to 3 shaders (intersection, any hit, closest hit).

```` C
  rayPipelineInfo.setGroupCount(
      static_cast<uint32_t>(m_rtShaderGroups.size()));  // 1-raygen, n-miss, n-(hit[+anyhit+intersect])
  rayPipelineInfo.setPGroups(m_rtShaderGroups.data());
````

The ray generation and closest hit shaders can trace rays, making the ray tracing a potentially recursive process.
To allow the underlying RTX layer to optimize the pipeline we indicate the maximum recursion depth used by our shaders. For the simplistic shaders we currently have, we set this depth to 1, meaning that even if the shaders would trigger recursion (ie. a hit shader calling `TraceNV()`), this recursion would be prevented by setting the result of this trace call as a miss. Note that it is preferable to keep the recursion level as low as possible, replacing it by a loop formulation instead.

```` C
  rayPipelineInfo.setMaxRecursionDepth(1);  // Ray depth
  rayPipelineInfo.setLayout(m_rtPipelineLayout);
  m_rtPipeline = m_device.createRayTracingPipelineNV({}, rayPipelineInfo);
````

Once the pipeline has been created we discard the shader modules:

```` C
  m_device.destroy(raygenSM);
  m_device.destroy(missSM);
  m_device.destroy(chitSM);
}
````

The pipeline layout and the pipeline itself also have to be cleaned up upon closing, hence we add this to `destroyResources`:

```` C
  m_device.destroy(m_rtPipeline);
  m_device.destroy(m_rtPipelineLayout);
````

## main

In the `main` function, we call the pipeline construction after the other ray tracing calls:

```` C
  helloVk.createRtPipeline();
````

# Shader Binding Table

In a typical rasterization setup, a current shader and its associated resources are bound prior to drawing the corresponding objects, then another shader and resource set can be bound for some other objects, and so on. Since ray tracing can hit any surface of the scene at any time, all shaders must be available simultaneously.

The Shader Binding Table is the blueprint of the ray tracing process. It indicates which ray generation shader to start with, which miss shader to execute if no intersections are found, and which hit shader groups can be executed for each instance. This association between instances and shader groups is created when setting up the geometry: for each instance we provided a `hitGroupId` in the TLAS. This value represents the index in the SBT corresponding to the hit group for that instance.

## Handles

The SBT is an array containing the handles to the shader groups used in the ray tracing pipeline. In our example, we will create a buffer for the three groups: raygen, miss and closest hit. The size of the handle is given by the `shaderGroupHandleSize` member of the ray tracing properties. We will then allocate a buffer of size `3 * shaderGroupHandleSize` and will consecutively write the handle of each shader group. To retrieve all the handles, we will call `vkGetRayTracingShaderGroupHandlesNV`.

The buffer will have the following information, which will later be used when calling `vkCmdTraceRaysNV`:

```` bash
+--------------+
| RayGen       |
| Handle       |
+--------------+
| Miss         |
| Handle       |
+--------------+
| HitGroup     |
| Handle       |
+--------------+
````

We first add the declarations of the SBT creation method and the SBT buffer itself in the `HelloVulkan` class:

```` C
void           createRtShaderBindingTable();
nvvkBuffer     m_rtSBTBuffer;
````

In this function, we start by computing the size of the binding table from the number of groups and the handle size so that we can allocate the SBT buffer.

```` C
//--------------------------------------------------------------------------------------------------
// The Shader Binding Table (SBT)
// - getting all shader handles and writing them in a SBT buffer
// - Besides exception, this could be always done like this
//   See how the SBT buffer is used in run()
//
void HelloVulkan::createRtShaderBindingTable()
{
  auto groupCount =
      static_cast<uint32_t>(m_rtShaderGroups.size());               // 3 shaders: raygen, miss, chit
  uint32_t groupHandleSize = m_rtProperties.shaderGroupHandleSize;  // Size of a program identifier

  // Fetch all the shader handles used in the pipeline, so that they can be written in the SBT
  uint32_t sbtSize = groupCount * groupHandleSize;
````

We then fetch the handles to the shader groups of the pipeline, and let the allocator allocate the device memory and copy the handles into the SBT:

```` C
  std::vector<uint8_t> shaderHandleStorage(sbtSize);
  m_device.getRayTracingShaderGroupHandlesNV(m_rtPipeline, 0, groupCount, sbtSize,
                                             shaderHandleStorage.data());
  // Write the handles in the SBT
  nvvkpp::SingleCommandBuffer genCmdBuf(m_device, m_graphicsQueueIndex);
  vk::CommandBuffer           cmdBuf = genCmdBuf.createCommandBuffer();

  m_rtSBTBuffer =
      m_alloc.createBuffer(cmdBuf, shaderHandleStorage, vk::BufferUsageFlagBits::eRayTracingNV);
  m_debug.setObjectName(m_rtSBTBuffer.buffer, "SBT");


  genCmdBuf.flushCommandBuffer(cmdBuf);

  m_alloc.flushStaging();
}
````

As with other resources, we destroy the SBT in `destroyResources`:

```` C
  m_alloc.destroy(m_rtSBTBuffer);
````

## main

In the `main` function, we now add the construction of the Shader Binding Table:

```` C
  helloVk.createRtShaderBindingTable();
````

# Ray Tracing

Let's create a function that will call the execution of the ray tracer. First, add the declaration to the header

```` C
void       raytrace(const vk::CommandBuffer& cmdBuf, const nvmath::vec4f& clearColor);
````

We first bind the pipeline and its layout, and set the push constants that will be available throughout the pipeline:

```` C
//--------------------------------------------------------------------------------------------------
// Ray Tracing the scene
//
void HelloVulkan::raytrace(const vk::CommandBuffer& cmdBuf, const nvmath::vec4f& clearColor)
{
  m_debug.beginLabel(cmdBuf, "Ray trace");
  // Initializing push constant values
  m_rtPushConstants.clearColor     = clearColor;
  m_rtPushConstants.lightPosition  = m_pushConstant.lightPosition;
  m_rtPushConstants.lightIntensity = m_pushConstant.lightIntensity;
  m_rtPushConstants.lightType      = m_pushConstant.lightType;

  cmdBuf.bindPipeline(vk::PipelineBindPoint::eRayTracingNV, m_rtPipeline);
  cmdBuf.bindDescriptorSets(vk::PipelineBindPoint::eRayTracingNV, m_rtPipelineLayout, 0,
                            {m_rtDescSet, m_descSet}, {});
  cmdBuf.pushConstants<RtPushConstant>(m_rtPipelineLayout,
                                       vk::ShaderStageFlagBits::eRaygenNV
                                           | vk::ShaderStageFlagBits::eClosestHitNV
                                           | vk::ShaderStageFlagBits::eMissNV,
                                       0, m_rtPushConstants);
````
  
Since the structure of the Shader Binding Table is up to the developer, we need to indicate the ray tracing pipeline how to interpret it. In particular we compute the offsets in the SBT where the ray generation shader, miss shaders and hit groups can be found. Miss shaders and hit groups are stored contiguously, hence we also compute the stride separating each shader. In our case the stride is simply the size of a shader group handle, but more advanced uses may embed shader-group-specific data within the SBT, resulting in a larger stride.

```` C  
  vk::DeviceSize progSize = m_rtProperties.shaderGroupHandleSize;  // Size of a program identifier
  vk::DeviceSize rayGenOffset   = 0u * progSize;  // Start at the beginning of m_sbtBuffer
  vk::DeviceSize missOffset     = 1u * progSize;  // Jump over raygen
  vk::DeviceSize missStride     = progSize;
  vk::DeviceSize hitGroupOffset = 2u * progSize;  // Jump over the previous shaders
  vk::DeviceSize hitGroupStride = progSize;
````

We can finally call `traceRaysNV` that will add the ray tracing launch in the command buffer. Note that the SBT buffer is mentioned several times. This is due to the possibility of separating the SBT into several buffers, one for each type: ray generation, miss shaders, hit groups, and callable shaders (outside the scope of this tutorial). The last three parameters are equivalent to the grid size of a compute launch, and represent the total number of threads. Since we want to trace one ray per pixel, the grid size has the width and height of the output image, and a depth of 1.

```` C  
  // m_sbtBuffer holds all the shader handles: raygen, n-miss, hit...
  cmdBuf.traceRaysNV(m_rtSBTBuffer.buffer, rayGenOffset,                    //
                     m_rtSBTBuffer.buffer, missOffset, missStride,          //
                     m_rtSBTBuffer.buffer, hitGroupOffset, hitGroupStride,  //
                     m_rtSBTBuffer.buffer, 0, 0,                            //
                     m_size.width, m_size.height,                           //
                     1);                                                    // depth

  m_debug.endLabel(cmdBuf);
}
````

# Let's Ray Trace

Now we have everything set up to be able to trace rays: the acceleration structure, the descriptor sets, the ray tracing pipeline and the shader binding table. Let's try to make images from this.

## main

In the `main` function, we will define a local variable to switch between rasterization and ray tracing. Add the following right after the ray tracing initialization calls:

```` C
bool useRaytracer = true;
````

In the same function, we will add a UI checkbox to make that switch at runtime. Right after the line `ImGui::ColorEdit3(`, we add

```` C
ImGui::Checkbox("Ray Tracer mode", &useRaytracer); // Switch between raster and ray tracing
````

A few lines below, you can find a block containing the `helloVk.rasterize` call. Since our application will now have two render modes, we replace that block by

```` C
  // Rendering Scene
  if(useRaytracer)
  {
    helloVk.raytrace(cmdBuff, clearColor);
  }
  else
  {
    cmdBuff.beginRenderPass(offscreenRenderPassBeginInfo, vk::SubpassContents::eInline);
    helloVk.rasterize(cmdBuff);
    cmdBuff.endRenderPass();
  }
````

Note that the ray tracing behaves more like a compute shader than a graphics task, and is then outside of a render pass.

We should now be able to alternate between rasterization and ray tracing. However, the ray tracing result only renders a flat gray image: the simplistic ray generation shader does not trace any ray yet, and simply returns a fixed color.

Raster                         |     | Ray Trace
:-----------------------------:|:---:|:--------------------------------:
![LOCAL](images/resultRasterCube.png )   | <> |   ![LOCAL](images/resultRaytraceEmptyCube.png)

# Camera Setup

In the context of rasterization, the vertices of the objects are projected from their world-space position into a $[0,1]\times[0,1]\times[0,1]$ cube, before being rasterized on the XY plane. For ray tracing, we need to initialize some rays at the camera position, and intersect the geometry in world space.
 To achieve this, we need to store the inverse view and projection matrices in the `CameraMatrices` at the beginning of the `hello_vulkan.cpp` file:

```` C
struct CameraMatrices
{
  nvmath::mat4f view;
  nvmath::mat4f proj;
  nvmath::mat4f viewInverse;
  // #VKRay
  nvmath::mat4f projInverse;
};
````

## updateUniformBuffer

The computation of the matrix inverses is done in `updateUniformBuffer`, after setting the `ubo.proj` matrix:

```` C
// #VKRay
ubo.projInverse = nvmath::invert(ubo.proj);
````

## Ray generation (`raytrace.rgen`)

It is now time to enrich the ray generation shader to allow it to trace rays. We will first add a new binding to allow the shader to access the camera matrices.

```` C
layout(binding = 0, set = 1) uniform CameraProperties
{
  mat4 view;
  mat4 proj;
  mat4 viewInverse;
  mat4 projInverse;
}
cam;
````

### Note: **Binding**

> The buffer of camera uses `binding = 0` as described in `createDescriptorSetLayout()`. The `set = 1` comes from the fact that it is the second descriptor set in `raytrace()`.

When tracing a ray, the hit or miss shaders need to be able to return some information to the shader program that invoked the ray tracing. This is done through the use of a payload, identified by the `rayPayloadNV` qualifier.

Since the payload struct will be reused in several shaders, we create a new shader file `raycommon.glsl` and add it to the Visual Studio folder.

This file contains only the payload definition:

~~~~ C++
struct hitPayload
{
  vec3 hitValue;
};
~~~~

We now modify `raytrace.rgen` to include this new file. Note that the `#include` directive is an GLSL extension, which we also enable:

~~~~ C++
#extension GL_GOOGLE_include_directive : enable
#include "raycommon.glsl"
~~~~

The payload, identified with `rayPayloadNV` is then our `hitPayload` structure.

```` C
layout(location = 0) rayPayloadNV hitPayload prd;
````

### Note

> In incoming shaders, like miss and closest hit, the payload will be `rayPayloadInNV`.

The `main` function of the shader then starts by computing the floating-point pixel coordinates, normalized between 0 and 1. The `gl_LaunchIDNV` contains the integer coordinates of the pixel being rendered, while `gl_LaunchSizeNV` corresponds to the image size provided when calling `traceRaysNV`.

```` C
void main()
{
    const vec2 pixelCenter = vec2(gl_LaunchIDNV.xy) + vec2(0.5);
    const vec2 inUV = pixelCenter/vec2(gl_LaunchSizeNV.xy);
    vec2 d = inUV * 2.0 - 1.0;
````

From the pixel coordinates, we can apply the inverse transformation of the view and projection matrices of the camera to obtain the origin and direction of the ray.

```` C
  vec4 origin    = cam.viewInverse * vec4(0, 0, 0, 1);
  vec4 target    = cam.projInverse * vec4(d.x, d.y, 1, 1);
  vec4 direction = cam.viewInverse * vec4(normalize(target.xyz), 0);
````

In addition, we provide some flags for the ray: first. a flag indicating that all geometry will be considered opaque, as we also indicated when creating the acceleration structures. We also indicate the minimum and maximum distance of the potential intersections along the ray. Those distances can be useful to reduce the ray tracing costs if intersections before or after a given point do not matter. A typical use case is for computing ambient occlusion.

```` C
  uint  rayFlags = gl_RayFlagsOpaqueNV;
  float tMin     = 0.001;
  float tMax     = 10000.0;
````

We now trace the ray itself, by first providing `traceNV` with the top-level acceleration structure and the ray masks. The `cullMask` value is a mask that will be binary AND-ed with the mask of the geometry instances. Since all instances have a `0xFF` flag as well, they will all be visible. The next 3 parameters indicate which hit group would be called when hitting a surface. For example, a single object may be associated to 2 hit groups representing
the behavior when hit by a direct camera ray, or from a shadow ray. Since each instance has an index indicating the offset of the hit groups for the instance in the shader binding table, the `sbtRecordOffset` will allow to fetch the right kind of shader for that instance. In the case of the primary rays we may want to use the first hit group and use an offset of 0, while for shadow rays the second hit group would be required, hence an offset of 1. The stride indicates the number of hit groups for a single instance. This is particularly useful if the instance offset is not set when creating the instances in the acceleration structure. A stride of 0 indicates that all hit groups are packed together, and the instance offset can be used directly to find them in the SBT. The index of the miss shader comes next, followed by the ray origin, direction and extents. The last parameter identifies the payload that will be carried by the ray, by giving its location index. The last `0` corresponds to the location of our payload, `layout(location = 0) rayPayloadNV hitPayload prd;`.

```` C  
  traceNV(topLevelAS,     // acceleration structure
          rayFlags,       // rayFlags
          0xFF,           // cullMask
          0,              // sbtRecordOffset
          0,              // sbtRecordStride
          0,              // missIndex
          origin.xyz,     // ray origin
          tMin,           // ray min range
          direction.xyz,  // ray direction
          tMax,           // ray max range
          0               // payload (location = 0)
  );
````

Finally, we write the resulting payload into the output buffer.

```` C
    imageStore(image, ivec2(gl_LaunchIDNV.xy), vec4(prd.hitValue, 1.0));
}
````

Raster                         |     | Ray Trace
:-----------------------------:|:---:|:--------------------------------:
![LOCAL](images/resultRasterCube.png)   | <> |   ![LOCAL](images/resultRaytraceFlatCube.png)

## Miss shader (`raytrace.miss`)

To share the clear color of the rasterization with the ray tracer, we will change the return value of the miss shader to return the clear value passed as a push constant. While the `Constants` struct contains more members, here we use the fact that `clearColor` is the first member in the struct, and do not even declare the subsequent members.

```` C
#extension GL_GOOGLE_include_directive : enable
#include "raycommon.glsl"

layout(location = 0) rayPayloadInNV hitPayload prd;

layout(push_constant) uniform Constants
{
  vec4 clearColor;
};

void main()
{
  prd.hitValue = clearColor.xyz * 0.8;
}
````

### Note: Background-Color

>The color of the background is slightly darker to differentiate the two renderers.

# Simple Lighting

The current closest hit shader only returns a flat color. To add some lighting, we will need to introduce the concept of surface normals. However, the ray tracing only provides the barycentric coordinates of the hit point. To obtain the normals and the other vertex attributes, we will need to find them in the vertex buffer and interpolate them using the barycentric coordinates. This is why we extended the usage of the vertex and index buffers when creating the ray tracing descriptor set.

## Closest Hit (`raytrace.rchit`)

When we created the ray tracing descriptor set, we already included the geometry definition. Therefore, we can reference the vertex and index buffers directly in the closest hit shader, via the scene description `binding = 2`

We first include the payload definition and the OBJ-Wavefront structures

```` C
#extension GL_EXT_scalar_block_layout : enable
#extension GL_GOOGLE_include_directive : enable
#include "raycommon.glsl"
#include "wavefront.glsl"
````

Then we describe the resources according to the descriptor set layout

```` C
layout(location = 0) rayPayloadInNV hitPayload prd;

layout(binding = 2, set = 1, scalar) buffer ScnDesc { sceneDesc i[]; } scnDesc;
layout(binding = 5, set = 1, scalar) buffer Vertices { Vertex v[]; } vertices[];
layout(binding = 6, set = 1) buffer Indices { uint i[]; } indices[];
````

In the Hit shader we need all the members of the push constant block:

```` C
layout(push_constant) uniform Constants
{
  vec4  clearColor;
  vec3  lightPosition;
  float lightIntensity;
  int   lightType;
}
pushC;
````

In the `main` function, the `gl_PrimitiveID` allows us to find the vertices of the triangle hit by the ray:

```` C
void main()
{
  // Object of this instance
  uint objId = scnDesc.i[gl_InstanceID].objId;

  // Indices of the triangle
  ivec3 ind = ivec3(indices[objId].i[3 * gl_PrimitiveID + 0],   //
                    indices[objId].i[3 * gl_PrimitiveID + 1],   //
                    indices[objId].i[3 * gl_PrimitiveID + 2]);  //
  // Vertex of the triangle
  Vertex v0 = vertices[objId].v[ind.x];
  Vertex v1 = vertices[objId].v[ind.y];
  Vertex v2 = vertices[objId].v[ind.z];
````

Using the hit point's barycentric coordinates, we can interpolate the normal:

```` C
  const vec3 barycentrics = vec3(1.0 - attribs.x - attribs.y, attribs.x, attribs.y);

  // Computing the normal at hit position
  vec3 normal = v0.nrm * barycentrics.x + v1.nrm * barycentrics.y + v2.nrm * barycentrics.z;
  // Transforming the normal to world space
  normal = normalize(vec3(scnDesc.i[gl_InstanceID].transfoIT * vec4(normal, 0.0)));
````

The world-space position could be calculated in two ways, the first one being to use the information from the hit shader. But this could have precision issues if the hit point is very far.

```` C
  vec3 worldPos = gl_WorldRayOriginNV + gl_WorldRayDirectionNV * gl_HitTNV;
````

Another solution, more precise, consists in computing the position by interpolation, as for the normal

```` C
  // Computing the coordinates of the hit position
  vec3 worldPos = v0.pos * barycentrics.x + v1.pos * barycentrics.y + v2.pos * barycentrics.z;
  // Transforming the position to world space
  worldPos = vec3(scnDesc.i[gl_InstanceID].transfo * vec4(worldPos, 1.0));
````

The light source specified in the constants can then be used to compute the dot product of the normal with the lighting direction, giving a simple diffuse lighting effect:

```` C
  // Vector toward the light
  vec3  L;
  float lightIntensity = pushC.lightIntensity;
  float lightDistance  = 100000.0;
  // Point light
  if(pushC.lightType == 0)
  {
    vec3 lDir      = pushC.lightPosition - worldPos;
    lightDistance  = length(lDir);
    lightIntensity = pushC.lightIntensity / (lightDistance * lightDistance);
    L              = normalize(lDir);
  }
  else // Directional light
  {
    L = normalize(pushC.lightPosition - vec3(0));
  }

  float dotNL = max(dot(normal, L), 0.2);

  prd.hitValue = vec3(dotNL);
}
````

![Light Grey Cube](images/resultRaytraceLightGreyCube.png)

# Simple Materials

The rendering above could be made more interesting by adding support for materials. The imported OBJ objects provide simplified Alias Wavefront material definitions.

## `raytrace.rchit`

These materials define their basic reflectance properties using simple color coefficients, and also support texturing. The buffer containing the materials has already been created for rasterization, and has also been added into the ray tracing descriptor set. Add the binding of the material buffer
and the array of texture samplers:

```` C
layout(binding = 1, set = 1, scalar) buffer MatColorBufferObject { WaveFrontMaterial m[]; } materials[];
layout(binding = 3, set = 1) uniform sampler2D textureSamplers[];
````

The declaration of the material is the same as that used for the rasterizer and is defined in
`wavefront.glsl`.

The `Vertex` structure contains a material index, which we will use to find the corresponding material in the buffer.

We first remove these lines at the end of `main()`

```` C
float dotNL = max(dot(normal, L), 0.2);
prd.hitValue = vec3(dotNL);
````

and fetch the material definition instead:

```` C
  // Material of the object
  int               matIdx = matIndex[objId].i[gl_PrimitiveID];
  WaveFrontMaterial mat    = materials[objId].m[matIdx];
````

**Note**: There is one buffer of materials per object, and each material can be access via the index. And each triangle has an index of material.

From that material definition, we use the diffuse and specular reflectances to compute diffuse lighting. This code also supports textures to modulate the surface albedo.

```` C
  // Diffuse
  vec3 diffuse = computeDiffuse(mat, L, normal);
  if(mat.textureId >= 0)
  {
    uint txtId = mat.textureId + scnDesc.i[gl_InstanceID].txtOffset;
    vec2 texCoord =
        v0.texCoord * barycentrics.x + v1.texCoord * barycentrics.y + v2.texCoord * barycentrics.z;
    diffuse *= texture(textureSamplers[txtId], texCoord).xyz;
  }
  
  // Specular
  vec3 specular = computeSpecular(mat, gl_WorldRayDirectionNV, L, normal);
````

The final lighting is then computed as

```` C
  prd.hitValue = vec3(lightIntensity * (diffuse + specular));
````

![Lighted Cube](images/resultRaytraceLightMatCube.png)

## main

The OBJ model is loaded in `main.cpp` by calling `helloVk.loadModel`. Let's load something more interesting than a cube:

```` C
  // Creation of the example
  helloVk.loadModel(nvh::findFile("media/scenes/Medieval_building.obj", defaultSearchPaths));
  helloVk.loadModel(nvh::findFile("media/scenes/plane.obj", defaultSearchPaths));
````

Since that model is larger, we can change the `CameraManip.setLookat` call to

```` C
CameraManip.setLookat(nvmath::vec3f(4, 4, 4), nvmath::vec3f(0, 1, 0), nvmath::vec3f(0, 1, 0));
````

![resultRaytraceLightMatMedieval](images/resultRaytraceLightMatMedieval.png)

# Shadows

The above allows us to ray trace a scene and apply some lighting, but it is still missing shadows. To this end, we will add a new ray type, and shoot rays from the closest hit shader. This new ray type will require adding a new miss shader.

## `createRaytracingPipeline`

For simple shadow rays we only need to compute whether some geometry was hit along the ray or not. This can be achieved using a Boolean payload initialized as if a hit were found, and ray trace using only an additional miss shader that will set the payload to no hit.

### Note: [Download Shadow Shader](files/shadowShaders.zip)


This archive contains only one file, `raytraceShadow.rmiss`. Add this file to the `src/shaders` directory and rerun CMake. The shader file should compile, and the resulting SPIR-V file should be stored in the `shaders` folder alongside the GLSL file.

In the body of `createRtPipeline`, we need to define the new miss shader right after the previous miss shader:

```` C
  // The second miss shader is invoked when a shadow ray misses the geometry. It
  // simply indicates that no occlusion has been found
  vk::ShaderModule shadowmissSM = nvvkpp::util::createShaderModule(
      m_device, nvh::loadFile("shaders/raytraceShadow.rmiss.spv", true, paths));

````

After pushing the miss shader `missSM`, we also push the miss shader for the shadow rays:

```` C
  // Shadow Miss
  stages.push_back({{}, vk::ShaderStageFlagBits::eMissNV, shadowmissSM, "main"});
  mg.setGeneralShader(static_cast<uint32_t>(stages.size() - 1));
  m_rtShaderGroups.push_back(mg);
````

The pipeline now has to allow shooting rays from the closest hit program, which requires increasing the recursion level to 2:

```` C
  // The ray tracing process can shoot rays from the camera, and a shadow ray can be shot from the
  // hit points of the camera rays, hence a recursion level of 2. This number should be kept as low
  // as possible for performance reasons. Even recursive ray tracing should be flattened into a loop
  // in the ray generation to avoid deep recursion.
  rayPipelineInfo.setMaxRecursionDepth(2);  // Ray depth
````

At the end of the method, we destroy the shader module for the shadow miss shader:

```` C
  m_device.destroy(shadowmissSM);
````

## `traceRaysNV`

The addition of the new miss shader group has modified our shader binding table, which now looks like:

```` bash
+--------------+
| RayGen       |
| Handle       |
+--------------+
| Miss         |
| Handle       |
+--------------+
| ShadowMiss   |
| Handle       |
+--------------+
| HitGroup     |
| Handle       |
+--------------+
````

Therefore, we have to change `HelloVulkan::raytrace` to adjust the the closest hit offset before calling `traceRaysNV`. This also points out that in real-world applications the SBT should be embedded so that it can handle those offsets automatically.

```` C
  vk::DeviceSize hitGroupOffset = 3u * progSize;  // Jump over the raygen and 2 miss shaders
````

## `createRtDescriptorSet`

For each resource entry in the descriptor set, we indicated which shader stage would be able to use it. Since shadow rays will be traced from the closest hit shader, we add `vkSS::eClosestHitNV` to the acceleration structure binding:

```` C
  // Top-level acceleration structure, usable by both the ray generation and the closest hit (to
  // shoot shadow rays)
  m_rtDescSetLayoutBind.emplace_back(
      vkDSLB(0, vkDT::eAccelerationStructureNV, 1, vkSS::eRaygenNV | vkSS::eClosestHitNV));  // TLAS
````

## `raytrace.rchit`

The closest hit shader now needs to be aware of the acceleration structure to be able to shoot rays:

```` C
layout(binding = 0, set = 0) uniform accelerationStructureNV topLevelAS;
````

Those rays will also carry a payload, which will need to be defined at a different location from the payload of the current ray. In this case, the payload will be a simple Boolean value indicating whether an occluder has been found or not:

```` C
layout(location = 1) rayPayloadNV bool isShadowed;
````

**Note**: in the `raytraceShadow.rmiss`, the payload is `rayPayload**In**NV` for accepting the incoming payload. It returns always false, because if it didn't hit something, then it is not in shadow.

In the `main` function, instead of simply setting our payload to `prd.hitValue = c;`, we will initiate a new ray. Note that the index of the miss shader is now 1, since the SBT has 2 miss shaders. The payload location is defined to match the declaration `layout(location = 1)` above. Note, when invoking `traceNV()`  we are setting the flags with:

* `gl_RayFlagsSkipClosestHitShaderNV`: Will not invoke the hit shader, only the miss shader
* `gl_RayFlagsOpaqueNV` : Will not call the any hit shader, so all objects will be opaque
* `gl_RayFlagsTerminateOnFirstHitNV` : The first hit is always good.

Since we skip the shadow hit group, no code will be invoked when hitting a surface. Therefore, we initialize the payload `isShadowed` to `true`, and will rely on the miss shader to set it to false if no surfaces have been encountered. We also set the ray flags to optimize the ray tracing: since these simple shadow rays only need to return whether the ray intersects any surface, we can instruct the ray tracing engine to stop the traversal after finding the first intersection, without trying to execute a closest hit shader.

Shadow rays only need to be cast if the light is in front of the surface, and specular lighting should not be
computed if we are in shadow (since the light source won't be visible from the shading point). The code that previously computed the specular term will then look like this:

```` C
  vec3  specular    = vec3(0);
  float attenuation = 1;

  // Tracing shadow ray only if the light is visible from the surface
  if(dot(normal, L) > 0)
  {
    float tMin   = 0.001;
    float tMax   = lightDistance;
    vec3  origin = gl_WorldRayOriginNV + gl_WorldRayDirectionNV * gl_HitTNV;
    vec3  rayDir = L;
    uint  flags =
        gl_RayFlagsTerminateOnFirstHitNV | gl_RayFlagsOpaqueNV | gl_RayFlagsSkipClosestHitShaderNV;
    isShadowed = true;
    traceNV(topLevelAS,  // acceleration structure
            flags,       // rayFlags
            0xFF,        // cullMask
            0,           // sbtRecordOffset
            0,           // sbtRecordStride
            1,           // missIndex
            origin,      // ray origin
            tMin,        // ray min range
            rayDir,      // ray direction
            tMax,        // ray max range
            1            // payload (location = 1)
    );

    if(isShadowed)
    {
      attenuation = 0.3;
    }
    else
    {
      // Specular
      specular = computeSpecular(mat, gl_WorldRayDirectionNV, L, normal);
    }
  }
````

The final payload value can then be adjusted depending on the result of the shadow ray:

```` C
prd.hitValue = vec3(lightIntensity * attenuation * (diffuse + specular));
````

![LOCAL](images/resultRaytraceShadowMedieval.png)

The final project can be found under this directory.

# Going Further

See the other tutorials in the extra directories. They are all starting from the en of this tutorial and are adding what is necessary for the additional feature.