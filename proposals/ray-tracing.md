# Hardware Ray Tracing

**Roadmap:** This proposal is **under active development, but has not been standardized for inclusion in the WebGPU specification. The proposal is likely to change before it is standardized.** WebGPU implementations **must not** expose this functionality; doing so is a spec violation. Note however, an implementation might provide an option (e.g. command line flag) to enable a draft implementation, for developers who want to test this proposal.

Last modified: 2025-03-30

Issue: #NN

# Requirements

**Vulkan**

For acceleration structures: `VK_KHR_acceleration_structure` and `VK_KHR_deferred_host_operations` and either
  - Vulkan 1.1 and `VK_EXT_descriptor_indexing` and `VK_KHR_buffer_device_address`
  - Vulkan 1.2

For ray queries: all features required for acceleration structures and `VK_KHR_ray_query` and `VK_KHR_spirv_1_4` or Vulkan 1.2

**DirectX12**

Ray Tracing Tier 1.1 or higher for acceleration structures and ray queries.

**Metal**

Metal 2.4, as ray queries need to be able to be in fragment and vertex shaders

# WGSL

## Enable extensions

| enable extension      | Description                                                                         |
|-----------------------|-------------------------------------------------------------------------------------|
| **ray-query**         | Enables the new types and all functions except `rayQueryGet*HitVertexPositions`     |
| **ray-vertex-return** | Enables the tag `vertex_return` and the `rayQueryGet*HitVertexPositions` functions  |

## Types

`acceleration_structure` corresponds to a TLAS in the webgpu API. 

`ray_query`

These types also support having a list of tags added to them which are enclosed in `<>`. These enclosing brackets are not require, and it will be assumed that the type is untagged if these are left out. E.g. `ray_query` is equivalent to `ray_query<>`. All tags add functionality, so a `ray_query<...>` can be automatically converted to a `ray_query`.

## Acceleration structure tags

| Tag             | Description                                     | Requirements              |
|-----------------|-------------------------------------------------|---------------------------|
| `vertex_return` | Allows getting the vertices of the hit triangle | Feature ray-vertex-return |

## New WGSL structures

````wgsl
struct RayDesc {
    // Contains flags to use for this ray (e.g. consider all `Blas`es opaque)
    flags: u32,
    // If the bitwise and of this and any `TlasInstance`'s `mask` is not zero then the object inside
    // the `Blas` contained within that `TlasInstance` may be hit.
    cull_mask: u32,
    // Only points on the ray whose t is greater than this may be hit.
    t_min: f32,
    // Only points on the ray whose t is less than this may be hit.
    t_max: f32,
    // The origin of the ray.
    origin: vec3<f32>,
    // The direction of the ray, t is calculated as the length down the ray divided by the length of `dir`.
    dir: vec3<f32>,
}

struct RayIntersection {
    // The kind of the hit, all other members of this struct are zeroed if this is equal
    // to constant `RAY_QUERY_INTERSECTION_NONE`.
    kind: u32,
    // Distance from starting point, measured in units of `RayDesc::dir`.
    t: f32,
    // Corresponds to `instance.custom_data` where `instance` is the `TlasInstance`
    // that the intersected object was contained in.
    instance_custom_data: u32,
    // The index into the `TlasPackage` to get the `TlasInstance` that the hit object is in
    instance_index: u32,
    // The offset into the shader binding table. Currently, this value is always 0.
    sbt_record_offset: u32,
    // The index into the `Blas`'s build descriptor (e.g. if `BlasBuildEntry::geometry` is
    // `BlasGeometries::TriangleGeometries` then it is the index into that contained vector).
    geometry_index: u32,
    // The object hit's index into the provided buffer (e.g. if the object is a triangle
    // then this is the triangle index)
    primitive_index: u32,
    // Two of the barycentric coordinates, the third can be calculated (only useful if this is a triangle).
    barycentrics: vec2<f32>,
    // Whether the hit face is the front (only useful if this is a triangle).
    front_face: bool,
    // Matrix for converting from object-space to world-space.
    //
    // This matrix needs to be on the left side of the multiplication. Using it the other way round will not work.
    // Use it this way: `let transformed_vector = intersecion.object_to_world * vec4<f32>(x, y, z, transform_multiplier);
    object_to_world: mat4x3<f32>,
    // Matrix for converting from world-space to object-space
    //
    // This matrix needs to be on the left side of the multiplication. Using it the other way round will not work.
    // Use it this way: `let transformed_vector = intersecion.world_to_object * vec4<f32>(x, y, z, transform_multiplier);
    world_to_object: mat4x3<f32>,
}
````

## New WGSL functions

````wgsl
rayQueryInitialize(rq: ptr<function, ray_query>, acceleration_structure: acceleration_structure, ray_desc: RayDesc);
````
Initializes the ray query with an acceleration structure and a ray descriptor.

````wgsl
rayQueryProceed(rq: ptr<function, ray_query>) -> bool
````
Traces the ray in the initialized ray_query (partially) through the scene. Returns `true` if a triangle that was hit by the ray was in a `GPUBLAS` that is not marked as opaque. Returns `false` if all triangles that were hit by the ray were in `GPUBLAS`es that were marked as opaque. The hit is considered `Candidate` if this function returns `true`, and the hit is considered `Committed` if this function returns false. A `Candidate` intersection interrupts the ray traversal. A `Candidate` intersection may happen anywhere along the ray, it should not be relied on to give the closest hit. A `Candidate` intersection is to allow the user themselves to decide if that intersection is valid*. If one wants to get the closest hit a `Committed` intersection should be used. Calling this function multiple times will cause the ray traversal to continue if it was interrupted by a `Candidate` intersection. If the `ray_query` is not initialized this call id a no-op.

````wgsl
rayQueryGenerateIntersection(hit_t: f32)
````
Generates a hit from procedural geometry at a particular distance. If the last `rayQueryProceed` did not hit an aabb or returned `true`, this call is a no-op.

````wgsl
rayQueryConfirmIntersection()
````
Commits a hit from triangular non-opaque geometry. If the last `rayQueryProceed` did not hit a triangle or returned `true`, this call is a no-op.

````wgsl
rayQueryTerminate()
````
Aborts the query.

````wgsl
rayQueryGetCommittedIntersection(rq: ptr<function, ray_query>) -> RayIntersection
````
Returns intersection details about a hit considered `Committed`. This function returns a zeroed structure if the hit was not considered `Committed`. Any members in the struct that depend on a particular hit type are zeroed if it is not that hit type.

````wgsl
rayQueryGetCandidateIntersection(rq: ptr<function, ray_query>) -> RayIntersection
````
Returns intersection details about a hit considered `Candidate`. This function returns a zeroed structure if the hit was not considered `Candidate`. Any members in the struct that depend on a particular hit type are zeroed if it is not that hit type.

````wgsl
getCommittedHitVertexPositions(rq: ptr<function, ray_query<vertex_return>>) -> array<vec3<f32>, 3>
````
Returns the vertices of the hit triangle considered `Committed`. Returns a zeroed array if the last hit was not considered `Committed`.

````wgsl
getCandidateHitVertexPositions(rq: ptr<function, ray_query<vertex_return>>) -> array<vec3<f32>, 3>
````
Returns the vertices of the hit triangle considered `Candidate`. Returns a zeroed array if the last hit was not considered `Candidate`.

## New WGSL constants

### For `RayDesc.flags`

````wgsl 
const FORCE_OPAQUE = 0x1;
````
All `GPUBLAS`es are considered opaque.

````wgsl
const FORCE_NO_OPAQUE = 0x2;
````
All `GPUBLAS`es are considered non-opaque.

````wgsl
const TERMINATE_ON_FIRST_HIT = 0x4;
````
Instead of searching for the closest hit return the first hit.

````wgsl
const CULL_BACK_FACING = 0x10;
````
If `RayIntersection.front_face` is false do not return a hit.

````wgsl
const CULL_FRONT_FACING = 0x20;
````
If `RayIntersection.front_face` is true do not return a hit.

````wgsl
const CULL_OPAQUE = 0x40;
````
Skip all `GPUBLAS`es that are marked as opaque.

````wgsl
const CULL_NO_OPAQUE = 0x80;
````
Skip all `GPUBLAS`es that are not marked as opaque.

````wgsl
const SKIP_TRIANGLES = 0x100;
````
If the `GPUBLAS` a intersection is checking contains triangles do not return a hit.

````wgsl
const SKIP_AABBS = 0x200;
````
If the `GPUBLAS` a intersection is checking contains AABBs do not return a hit.

### For `RayIntersection.kind`

````wgsl
const RAY_QUERY_INTERSECTION_NONE = 0;
````
The ray hit nothing.

```wgsl
const RAY_QUERY_INTERSECTION_TRIANGLE = 1;
```
The ray hit a triangle.

````wgsl
const RAY_QUERY_INTERSECTION_GENERATED = 2;
````
The ray hit a custom object, this will only happen in a committed intersection if a ray which intersected a bounding box for a custom object which was then committed.

````wgsl
const RAY_QUERY_INTERSECTION_AABB = 3;
````
The ray hit a AABB, this will only happen in a candidate intersection if the ray intersects the bounding box for a custom object.

# API

## GPU Feature

New GPU features:

| Feature                    | Description                                                                                                                  |
|----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| **ray-query**              | Allows the corresponding enable extension, also allows creation of bind groups and bind group layouts containing `GPUTLAS`es |
| **ray-vertex-return**      | Allows the corresponding enable extension                                                                                    |
| **acceleration-structure** | Allows the creation and building of `GPUBLAS`es and `GPUTLAS`es, allows compaction of `GPUBLAS`es                            |

## Limits

Four new limits:

| Limit name                              | Description                                                                              | Type                                                            | Limit class                                                  | Default  |
|-----------------------------------------|------------------------------------------------------------------------------------------|-----------------------------------------------------------------|--------------------------------------------------------------|----------|
| maxBlasGeometryCount                    | The maximum number of geometry descriptors a `GPUBLAS` is allowed to have.               | [GPUSize32](https://www.w3.org/TR/webgpu/#typedefdef-gpusize32) | [maximum](https://www.w3.org/TR/webgpu/#limit-class-maximum) | 2^24 - 1 |
| maxBlasPrimitiveCount                   | The maximum number of triangle or aabb descriptors a `GPUBLAS` is allowed to have.       | [GPUSize32](https://www.w3.org/TR/webgpu/#typedefdef-gpusize32) | [maximum](https://www.w3.org/TR/webgpu/#limit-class-maximum) | 2^28     |
| maxTlasInstanceCount                    | The maximum number of instances a `GPUTLAS` is allowed to have.                          | [GPUSize32](https://www.w3.org/TR/webgpu/#typedefdef-gpusize32) | [maximum](https://www.w3.org/TR/webgpu/#limit-class-maximum) | 2^24 - 1 |
| maxAccelerationStructuresPerShaderStage | The maximum number of `GPUTLAS`es that are allowed to be bound to a single shader stage. | [GPUSize32](https://www.w3.org/TR/webgpu/#typedefdef-gpusize32) | [maximum](https://www.w3.org/TR/webgpu/#limit-class-maximum) | 16       |

On Vulkan these limits would be

| Limit                                   | Range            |
|-----------------------------------------|------------------|
| maxBlasGeometryCount                    | 2^24 - 1 or more |
| maxBlasPrimitiveCount                   | 2^29 - 1 or more |
| maxTlasInstanceCount                    | 2^24 - 1 or more |
| maxAccelerationStructuresPerShaderStage | 16 or more       |

On DirectX these limits would be

| Limit                                   | Range                                       |
|-----------------------------------------|---------------------------------------------|
| maxBlasGeometryCount                    | 2^24                                        |
| maxBlasPrimitiveCount                   | 2^29                                        |
| maxTlasInstanceCount                    | 2^24                                        |
| maxAccelerationStructuresPerShaderStage | Implementation dependent - counts as an SRV |

On Metal these limits would be

| Limit                                   | Range                                         |
|-----------------------------------------|-----------------------------------------------|
| maxBlasGeometryCount                    | 2^24                                          |
| maxBlasPrimitiveCount                   | 2^28                                          |
| maxTlasInstanceCount                    | 2^24                                          |
| maxAccelerationStructuresPerShaderStage | Implementation dependent - counts as a buffer |

Useful info: [Vulkan limits](https://docs.vulkan.org/spec/latest/chapters/limits.html#VkPhysicalDeviceAccelerationStructurePropertiesKHR), [DXR limits](https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html#geometry-limits), [Metal limit](https://developer.apple.com/documentation/metal/mtlaccelerationstructureusage/extendedlimits?language=objc).

## Allowed `GPUVertexFormat` for a triangle BLAS build given features

| Format    | Allowed by                 |
|-----------|----------------------------|
| float32x3 | **acceleration-structure** |

## Flags

````javascript
namespace GPUAccelerationStructureGeometryFlags {
    const GPUFlagsConstant OPAQUE = 0x0001;
    const GPUFlagsConstant NO_DUPLICATE_ANY_HIT_INVOCATION = 0x0002;
}

namespace GPUAccelerationStructureFlags {
    const GPUFlagsConstant ALLOW_UPDATE = 0x0001;
    const GPUFlagsConstant ALLOW_COMPACTION = 0x0002;
    const GPUFlagsConstant PREFER_FAST_TRACE = 0x0004;
    const GPUFlagsConstant PREFER_FAST_BUILD = 0x0008;
    const GPUFlagsConstant LOW_MEMORY = 0x0010;
    const GPUFlagsConstant USE_TRANSFORM = 0x0020;
    const GPUFlagsConstant ALLOW_RAY_HIT_VERTEX_RETURN = 0x0040;
}
````

## New types

````javascript
enum GPUAccelerationStructureUpdateMode {
    "prefer-update",
    "build",
}

dictionary GPUBLASTriangleGeometrySizeDescriptor {
    required GPUVertexFormat vertexFormat;
    required unsigned long vertexCount;
    GPUIndexFormat indexFormat;
    unsigned long indexCount;
    GPUFlagsConstant flags;
}

typedef (sequence<GPUBLASTriangleGeometrySizeDescriptor>) GPUBLASGeometrySizeDescriptor;

dictionary GPUBLASDescriptor
    : GPUObjectDescriptorBase {
    GPUFlagsConstant flags;
    GPUAccelerationStructureUpdateMode updateMode;
}

dictionary GPUBLASTriangleGeometry {
    required GPUBLASTriangleGeometrySizeDescriptor size;
    required GPUBuffer vertexBuffer;
    unsigned long firstVertex = 0;
    required GPUSize64 vertexStride;
    GPUBuffer indexBuffer;
    unsigned long firstIndex;
    GPUBuffer transformBuffer;
    GPUSize64 transformBufferOffset;
}

typedef (sequence<GPUBLASTriangleGeometry>) GPUBLASGeometries;

dictionary GPUBLASBuildEntry {
    GPUBLAS blas;
    GPUBLASGeometries geometry;
}

dictionary GPUTLASDescriptor
    : GPUObjectDescriptorBase {
    unsigned long maxInstances;
    GPUFlagsConstant flags;
    GPUAccelerationStructureUpdateMode updateMode;
}
````

Note: The `*Count` fields of `GPUBlasTriangleGeometry.size` may be less than or equal to those of the corresponding `GPUBLASTriangleGeometrySizeDescriptor` when creating the `GPUBLAS`.

## New resources

````javascript
interface GPUBLAS {
    readonly attribute GPUFlagsConstant usage;

    Promise<undefined> prepareToCompactAsync();

    undefined destroy();
};
````

### `GPUBLAS`'s content timeline properties
 - `preparingToCompact`
   - A `Promise<void>` that is initially null
 - `readyToCompact`
   - A `boolean` indicating whether this `BLAS` is ready to compact.
     - If `this.compactState` is `ready`

### `GPUBLAS`'s device timeline properties
 - `compactState`
   - `idle`
     - Uncompacted, read back has not yet started
   - `pending`
     - `prepareToCompactAsync` has been called, waiting for the latest build to complete
   - `ready`
     - Size has been read back.
   - `compacted`
     - Blas was created by a compaction
 - `compactionSize`
   - Undefined if `compactState` is not `ready`, otherwise the size of BLAS which needs to be created to compact
 - `sizes`
   - The `GPUBLASGeometrySizeDescriptor` passed in to `createBlas`
 - `flags`
   - The `GPUFlagsConstant` passed in to `createBlas`
 - `buildIndex`
   - A `unsigned long long?` that is never zero, starts out null.
 - `mapping`
   - Type `active buffer mapping` or nul, starts as null.

### `GPUBLAS.prepareToCompactAsync()`
Prepares the `GPUBLAS` to be compacted

#### `GPUBLAS.prepareToCompactAsync()` device timeline validation steps:
 - Check `this` is valid.
 - Check `buildIndex` is not undefined.
 - Check `this.compactState` is idle.
 - Check `this.flags` contains `ALLOW_COMPACTION`

If any of these validation steps are unsatisfied, generate a validation error and return, then either:

 - When the device timeline becomes informed of the completion of an unspecified queue timeline point:
   - After the completion of currently-enqueued operations that use *this*
   - No later than the completion of all currently-enqueued operations.
 - When `this.compactState` is set to `idle`.
Proceed to read back steps.
#### `GPUBLAS.prepareToCompactAsync()` device timeline read back steps:
 - If `this.compactState` is `idle`, return
 - Read into `compactionSize` the data from `mapping`

Note: this is very similar to map async, internally this code maps and reads back a buffer that had a query written to it when building

````javascript
interface GPUTLASInstance {
    required GPUBLAS blas;
    // Not sure what this should be. In rust `[f32; 12]`.
    TODO matrix;
    unsigned long customData;
    octet mask;
}
````

````javascript
interface GPUTLAS {
    attribute FrozenArray<GPUTLASInstance?> instances;

    undefined destroy();
};
````

### `GPUTLAS`'s device timeline properties
- `flags`
  - The `GPUFlagsConstant` passed in to `createTlas`
- `buildIndex`
  - An `unsigned long long?` that is never zero, starts out null.
- `dependents`
  - An `Array<GPUBLAS>` containing the most recent build's `this.instances`'s `GPUBLAS`es

## GPUBindGroupLayout and GPUBindGroup

````javascript
dictionary GPUAccelerationStructureBindingLayout {
    boolean vertexReturn = false;
}
````

````javascript
dictionary GPUAccelerationStructureBinding {
    required GPUTLAS tlas;
}
````

One new member in `GPUBindGroupLayoutEntry`.

````javascript
dictionary GPUBindGroupLayoutEntry {
    required GPUIndex32 binding;
    required GPUShaderStageFlags visibility;

    GPUBufferBindingLayout buffer;
    GPUSamplerBindingLayout sampler;
    GPUTextureBindingLayout texture;
    GPUStorageTextureBindingLayout storageTexture;
    GPUExternalTextureBindingLayout externalTexture;
    GPUAccelerationStructureBindingLayout accelerationStructure;
};
````
`GPUAccelerationStructureBindingLayout`: When exists, indicates the binding resource type for this `GPUBindGroupLayoutEntry`
is `GPUAccelerationStructureBinding`.

## GPUDevice

````javascript
interface GPUDevice {
    GPUBLAS createBlas(
        GPUBLASDescriptor descriptor,
        GPUBLASGeometrySizeDescriptor sizeDescriptor,
    );
    GPUTLAS createTlas(GPUTLASDescriptor descriptor);
}
````

### Additional `GPUDevice` device timeline properties
- `nextAccelerationStructureBuildCommandIndex`
  - An `unsigned long long` indicating what the next acceleration structure will be.

### `GPUDevice`'s device timeline build index fetch steps
 - Atomically load, increment and store to this command encoder's device's `nextAccelerationStructureBuildCommandIndex`.
 - Return the loaded value.

### `createBlas` device timeline steps
- Check *this* is not invalid or lost
- If `sizeDescriptor` is `sequence<GPUBLASTriangleGeometry>`
  - Check `sizeDescriptor.length` is less than `limits.maxBlasGeometryCount` 
  - For each `triangleSizeDescriptor` in `sizeDescriptor`
    - Check `triangleSizeDescriptor.vertexFormat` is allowed given the features.
    - Check `triangleSizeDescriptor.vertexCount` is less than `limits.maxBlasPrimitiveCount`.
    - If `triangleSizeDescriptor.indexFormat` is set 
      - Check `triangleSizeDescriptor.indexCount` is set.
      - Check `triangleSizeDescriptor.indexCount` is less than `limits.maxBlasPrimitiveCount`.
- If any of these validation steps are unsatisfied, generate a validation error and return.
- Create a blas given `descriptor` and `sizeDescriptor`

### `createTlas` device timeline steps
- Check *this* is not invalid or lost
- Check `descriptor.maxInstances` is less than `limits.maxTlasInstanceCount`.

## GPUCommandEncoder

One new method:
````javascript
interface GPUCommandEncoder {
    undefined buildAccelerationStructures(
        sequence<GPUBLASBuildEntry> blasEntries,
        sequence<GPUTLAS> tlases,
    );
}
````

### `buildAccelerationStructures`

#### `buildAccelerationStructures`'s content timeline steps:
- For each `tlas` in `tlases`
  - Make a copy of `tlas.instances` for the device timeline
- Go to `device timeline validation steps`

#### `buildAccelerationStructures`'s device timeline validation steps:
- For each `blasEntry` in `blasEntries`
  - Check `blasEntry.blas` is valid
  - Check `blasEntry.blas.compactState` is not `compacted`
  - If `blasEntry.geometry` is `sequence<GPUBLASTriangleGeometry>`
    - Check `blasEntry.blas.sizes` is `sequence<GPUBLASTriangleGeometrySizeDescriptor>`
    - Check `blasEntry.geometry.length` is `blasEntry.blas.sizes`
    - For each `triangleDesc` in `blasEntry.geometry` and the corresponding `triangleSizeDesc` in `blasEntry.blas.sizes`
      - Check `triangleDesc.vertexBuffer` is valid
      - Check `triangleDesc.size.vertexCount` is less than `triangleSizeDesc.vertexCount`
      - Check `triangleDesc.size.vertexFormat` is equal to `triangleSizeDesc.vertexFormat`
      - Check (`triangleDesc.firstVertex` + `triangleDesc.size.vertexCount`) * `triangleDesc.vertexStride` is less than or equal to the length of `triangleDesc.vertexBuffer`
      - If `triangleDesc.indexBuffer` is set
        - Check it is valid
        - Check `triangleDesc.firstIndex` is set
        - Check `triangleDesc.size.indexFormat` is set and matches `triangleSizeDesc.indexFormat`
        - Check `triangleDesc.size.indexCount` is set and is less than or equal to `triangleSizeDesc.indexCount`
        - Check (`triangleDesc.firstIndex` + `triangleDesc.size.indexCount`) * `triangleDesc.size.indexFormat`'s Byte Size less than or equal to the length of `triangleDesc.indexBuffer`
        - Check `triangleDesc.size.indexCount` is a multiple of 3.
      - Otherwise
        - Check `triangleDesc.firstIndex` is not set
        - Check `triangleDesc.size.indexFormat` is not set
        - Check `triangleSizeDesc.indexFormat` is not set
        - Check `triangleDesc.size.indexCount` is not set
        - Check `triangleSizeDesc.indexCount` is not set
      - If `triangleDesc.transformBuffer` is set
        - Check it is valid
        - Check `triangleDesc.transformBufferOffset` is set
        - Check `blasEntry.blas.flags` contains `USE_TRANSFORM`
        - Check `triangleDesc.transformBufferOffset` + 48 is less than or equal to `triangleDesc.transformBuffer`
      - Otherwise
        - Check `triangleDesc.transformBufferOffset` is not set
        - Check `blasEntry.blas.flags` does not contain `USE_TRANSFORM`
- For each `tlas` in `tlases`
  - Check `tlas` is valid
  - Create `newDependencies` an `Array<GPUBLAS>`
  - For each `instance` in the copy of `tlas.instances`
    - If instance is not null
      - Check `instance.blas` is valid
      - Push `instance.blas` to `newDependencies`
- If any check is not valid, invalidate *this* and return
- For each `blasEntry` in `blasEntries`
  - Set `blasEntry.blas.compactState` to `idle`
- Enqueue a command on *this* that issues the queue timeline steps
#### `buildAccelerationStructures`'s queue timeline steps:
- For each `blasEntry` in `blasEntries`
  - Build `blasEntry.blas` using the data from the `blasEntry.geometry` entries.
  - If `blasEntry.blas.flags` has `ALLOW_COMPACTION`, begin reading the compacted size back.
- Build every `GPUTLAS` in `tlases` with the `GPUTLASInstance`es contained in it.


## GPUQueue

````javascript
interface GPUQueue {
    GPUBLAS compactBLAS(GPUBLAS blas);
}
````

### Additional validation steps on `submit`
- For every `buildAccelerationStructures` called on submitted encoders
    - Run `build index fetch steps` on this command encoder's device. the value returned will be `currentBuildIndex`
    - Set the field `buildIndex` on every `GPUBLAS` built by this command to `currentBuildIndex`
    - For every `GPUTLAS` built by this command
      - For every `GPUBLAS` in `newDependencies`
        - Check `buildIndex` in this blas is not null

### `compactBLAS`'s device timeline steps.
- Check `blas.compactState` is `ready`
- If this check fails, invalidate *this* and return.
- Create a new `GPUBLAS` `compactedBlas` of size `blas.compactionSize`
- Run `build index fetch steps` on this command encoder's device. the value returned will be `currentBuildIndex`
  - Set `compactedBlas.buildIndex` to `currentBuildIndex`
  - Set `compactedBlas.compactState` to `compacted`
- Enqueue a command on *this* that issues the queue timeline steps
### `compactBLAS`'s queue timeline steps
- Perform a compacting copy from `blas` to `compactedBlas`

# Open Questions:


