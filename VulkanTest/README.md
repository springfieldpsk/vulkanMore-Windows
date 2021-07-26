# VulkanMore

## Vertex buffers 顶点缓冲区

### Vertex input description 顶点输入描述

#### Introduction 引言

在接下来的几章中，我们将用内存中的顶点缓冲区替换顶点着色器中硬编码的顶点数据。我们将从最简单的方法开始，创建一个 CPU 可见缓冲区，并使用 `memcpy` 直接将顶点数据复制到它，然后我们将看到如何使用分级缓冲区将顶点数据复制到高性能内存.

#### Vertex shader 顶点着色器

首先将顶点着色器更改为不再包含着色器代码本身中的顶点数据。顶点着色器使用 in 关键字从顶点缓冲区获取输入。

```cpp
#version 450

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;


void main() {
    gl_Position = vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

`inPosition` 和 `inColor` 变量是顶点属性。它们是在顶点缓冲区中为每个顶点指定的属性，就像我们使用两个数组手动指定每个顶点的位置和颜色一样。确保重新编译顶点着色器！

就像 `fragColor` 一样，`layout(location = x)`注释为输入分配索引，我们以后可以使用这些索引来引用它们。重要的是要知道，有些类型，如 dvec3 64位向量，使用多个插槽。这意味着，其后的指数必须至少高出2:

```cpp
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

你可以在 OpenGL wiki 中找到关于布局限定符的更多信息。

#### Vertex data 顶点数据

我们将顶点数据从着色器代码移动到程序代码中的数组中。首先包括 GLM 库，它为我们提供了与线性代数相关的类型，如向量和矩阵。我们将使用这些类型来指定位置和颜色矢量。

```cpp
#include <glm/glm.hpp>
```

创建一个新的结构称为顶点与两个属性，我们将在顶点着色器内使用它:

```cpp
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
};
```

GLM 为我们提供了与着色器语言中使用的向量类型完全匹配的 C++ 类型。

```cpp
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

现在使用 `Vertex` 结构来指定顶点数据数组。我们使用了与以前完全相同的位置和颜色值，但是现在它们被组合成一个顶点数组。这就是所谓的交错顶点属性。

#### Binding descriptions 绑定描述

下一步是告诉 Vulkan 如何传递这个数据格式到顶点着色器，一旦它被上传到 GPU 内存。需要两种类型的结构来传达这种信息。

第一个结构是 `VkVertexInputBindingDescription`，我们将向 Vertex 结构添加一个成员函数，用正确的数据填充它

```cpp
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};

        return bindingDescription;
    }
};
```

顶点绑定描述了以何种速率从内存中加载整个顶点的数据。它指定数据条目之间的字节数，以及是在每个顶点之后还是在每个实例之后移动到下一个数据条目。

```cpp
VkVertexInputBindingDescription bindingDescription{};
bindingDescription.binding = 0;
bindingDescription.stride = sizeof(Vertex);
bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
```

我们所有的每个顶点数据都打包在一个数组中，所以我们只有一个绑定。`binding` 参数指定绑定数组中绑定的索引。`stride` 参数指定从一个条目到下一个条目的字节数，`inputRate` 参数可以有以下值之一:

- `VK_VERTEX_INPUT_RATE_VERTEX` 每个顶点后的移动到下一个数据条目
- `VK_VERTEX_INPUT_RATE_INSTANCE` 在每个实例之后移动到下一个数据条目

我们不打算使用实例渲染，所以我们将坚持使用每个顶点的数据。

#### Attribute descriptions 属性描述

描述如何处理顶点输入的第二个结构是 `VkVertexInputAttributeDescription`。我们将向 `Vertex` 添加另一个辅助函数来填充这些结构。

```cpp
#include <array>

...

static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions() {
    std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

    return attributeDescriptions;
}
```

正如函数原型所指出的，将会有两个这样的结构。属性描述结构描述如何从来自绑定描述的顶点数据块中提取顶点属性。我们有两个属性，位置和颜色，所以我们需要两个属性描述结构。

```cpp
attributeDescriptions[0].binding = 0;
attributeDescriptions[0].location = 0;
attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
attributeDescriptions[0].offset = offsetof(Vertex, pos);
```

`binding`参数告诉 Vulkan 绑定每个顶点数据的来源。`location`参数引用顶点着色器中输入的`location`指令。位置为0的顶点着色器中的输入是位置，它有两个32位浮动分量。

Format 参数描述属性的数据类型。有点令人困惑的是，这些格式使用与颜色格式相同的枚举来指定。下列着色器类型和格式通常一起使用:

- `float`: `VK_FORMAT_R32_SFLOAT`
- `vec2`: `VK_FORMAT_R32G32_SFLOAT`
- `vec3`: `VK_FORMAT_R32G32B32_SFLOAT`
- `vec4`: `VK_FORMAT_R32G32B32A32_SFLOAT`

如您所见，您应该使用颜色通道数量与着色器数据类型中的组件数量相匹配的格式。它允许使用比着色器中的组件数量更多的通道，但是它们将被无声地丢弃。如果通道数低于组件数，那么 BGA 组件将使用默认值`(0,0,1)`。颜色类型(`SFLOAT`,`UINT`, `SINT`)和位宽度也应该匹配着色器输入的类型。请看下面的例子:

- `ivec2`: `VK_FORMAT_R32G32_SINT`, a 2-component vector of 32-bit signed integers 一个由32位有符号整数组成的2分量向量
- `uvec4`: `VK_FORMAT_R32G32B32A32_UINT`, a 4-component vector of 32-bit unsigned integers 一个由32位无符号整数组成的4分量向量
- `double`: `VK_FORMAT_R64_SFLOAT`, a double-precision (64-bit) float ，双精度(64位)浮点

`Format` 参数隐式定义属性数据的字节大小，而 `offset` 参数指定从每个顶点数据开始读取的字节数。此次定义的绑定每次加载一个顶点，位置属性(pos)从这个结构的开始处偏移0字节。这里使用`offsetof`宏的自动计算偏移量自动计算。

```cpp
attributeDescriptions[1].binding = 0;
attributeDescriptions[1].location = 1;
attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
attributeDescriptions[1].offset = offsetof(Vertex, color);
```

颜色属性的描述方式大致相同

#### Pipeline vertex input 管道顶点输入

我们现在需要设置渲染管线接受这种格式的顶点数据，方法是引用 `createGraphicsPipeline` 中的结构。找到 `vertexInputInfo` 结构并修改它以引用以下两个描述:

```cpp
auto bindingDescription = Vertex::getBindingDescription();
auto attributeDescriptions = Vertex::getAttributeDescriptions();

vertexInputInfo.vertexBindingDescriptionCount = 1;
vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```

管道现在可以接受顶点容器格式的顶点数据，并将其传递给顶点着色器。如果您现在在启用验证层的情况下运行程序，您将看到它提示没有顶点缓冲区绑定到绑定上。下一步是创建一个顶点缓冲区，并将顶点数据移动到它上面，这样 GPU 就可以访问它。

### Vertex buffer creation 创建顶点缓冲区

#### Introduction 创建顶点缓冲区引言

Vulkan 中的缓冲区是用于存储可由显卡读取的任意数据的内存区域。它们可以用来存储顶点数据，这一点我们将在本章中讨论，但它们也可以用于许多其他目的，我们将在以后的章节中探讨。与我们到目前为止处理的 Vulkan 对象不同，缓冲区不会自动为自己分配内存。前面章节的工作已经表明 Vulkan API 使程序员控制几乎所有事情，内存管理就是其中之一。

#### Buffer creation 创建缓冲区

创建一个新函数 `createVertexBuffer`，并在 `createCommandBuffers` 之前从 `initVulkan` 调用它。

```cpp
void initVulkan() {
    createInstance();
    setupDebugMessenger();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandPool();
    createVertexBuffer();
    createCommandBuffers();
    createSyncObjects();
}

...

void createVertexBuffer() {

}
```

创建缓冲区需要填充 `VkBufferCreateInfo` 结构。

```cpp
VkBufferCreateInfo bufferInfo{};
bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
bufferInfo.size = sizeof(vertices[0]) * vertices.size();
```

这个结构的第一个字段是 `size`，它以字节为单位指定缓冲区的大小。使用 `sizeof` 可以直接计算顶点数据的字节大小。

```cpp
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
```

第二个字段是 `usage`，它指示缓冲区中的数据将用于何种目的。可以使用位或。我们的用例将是一个顶点缓冲区，我们将在以后的章节中研究其他类型的用法。

```cpp
bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

就像交换链中的映像一样，缓冲区也可以由特定的队列家族拥有，或者同时在多个队列之间共享。缓冲区将只用于图形队列，因此我们可以坚持独占访问。

`flags` 参数用于配置稀疏缓冲区内存，这一点目前并不重要。我们将它保留在默认值0。

我们现在可以使用 `vkCreateBuffer` 创建缓冲区。定义一个类成员来保存缓冲区句柄并称之为 `vertexBuffer`。

```cpp
VkBuffer vertexBuffer;

...

void createVertexBuffer() {
    VkBufferCreateInfo bufferInfo{};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = sizeof(vertices[0]) * vertices.size();
    bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &vertexBuffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create vertex buffer!");
    }
}
```

缓冲区应该可以在渲染命令中使用，直到程序结束，并且它不依赖于交换链，所以我们将在原始的`cleanup`函数中清理它:

```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);

    ...
}
```

#### Memory requirements 内存需求

缓冲区已经被创建，但是它实际上还没有分配给它任何内存。为缓冲区分配内存的第一步是使用名为 `vkGetBufferMemoryRequirements` 的函数查询其内存需求。

```cpp
VkMemoryRequirements memRequirements;
vkGetBufferMemoryRequirements(device, vertexBuffer, &memRequirements);
```

`VkMemoryRequirements` 结构有三个字段:

- `size`: 所需内存量的大小(以字节为单位)可能与`bufferInfo.size`不同.
- `alignment`: 缓冲区在分配的内存区域开始时的字节偏移量，取决于`bufferInfo.usage` 及 `bufferInfo.flags`.
- `memoryTypeBits`: 适合于缓冲区的内存类型的位字段

显卡可以提供不同类型的内存来分配。每种类型的内存根据允许的操作和性能特征而变化。我们需要结合缓冲区的需求和我们自己的应用程序需求来找到正确的内存类型来使用。让我们为此创建一个新函数 `findMemoryType`。

```cpp
uint32_t findMemoryType(uint32_t typeFilter, VkMemoryPropertyFlags properties) {

}
```

首先，我们需要使用 `vkGetPhysicalDeviceMemoryProperties` 查询关于可用内存类型的信息

```cpp
VkPhysicalDeviceMemoryProperties memProperties;
vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProperties);
```

`Vkphysicaldevicemoryproperties` 结构有两个数组 `memoryTypes` 和 `memoryHeaps`。内存堆是不同的内存资源，比如专用的 VRAM 和当 VRAM 用完时在 RAM 中的交换空间。这些堆中存在不同类型的内存。现在我们只关心内存的类型，而不是它来自哪个堆，但是您可以想象这会影响性能。

让我们首先找到一个适合缓冲区本身的内存类型:

```cpp
for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
    if (typeFilter & (1 << i)) {
        return i;
    }
}

throw std::runtime_error("failed to find suitable memory type!");
```

`typeFilter` 参数将用于指定适当的内存类型的位字段。这意味着我们可以通过简单的迭代找到一个合适的内存类型的索引，并检查相应的位是否设置为1。

然而，我们不仅仅对适合于顶点缓冲区的内存类型感兴趣。我们还需要能够写入我们的顶点数据的内存。`memoryTypes` 数组由 `VkMemoryType` 结构组成，这些结构指定了堆和每种类型内存的属性。这些属性定义了内存的特殊功能，比如能够映射内存，这样我们就可以从 CPU 写入内存。这个属性是用 `VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT` 表示的，但是我们也需要使用 `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT` 属性。当我们映射内存的时候，我们就知道为什么了。

我们现在可以修改循环来检查是否支持这个属性:

```cpp
for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++) {
    if ((typeFilter & (1 << i)) && (memProperties.memoryTypes[i].propertyFlags & properties) == properties) {
        return i;
    }
}
```

我们可能有不止一个理想的属性，所以我们应该检查位 AND 的结果是否不仅仅是非零，而是等于所需的属性位字段。如果存在一个适合缓冲区的内存类型，并且该内存类型也具有我们需要的所有属性，那么我们将返回它的索引，否则我们将抛出一个异常。

#### Memory allocation 内存分配

现在我们有了一种确定正确内存类型的方法，因此我们实际上可以通过填充 `VkMemoryAllocateInfo` 结构来分配内存。

```cpp
VkMemoryAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
allocInfo.allocationSize = memRequirements.size;
allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
```

内存分配现在就像指定大小和类型一样简单，这两者都是从顶点缓冲区的内存需求和期望的属性中派生出来的。创建一个类成员将句柄存储到内存中，并用 `vkAllocateMemory` 分配它。

```cpp
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;

...

if (vkAllocateMemory(device, &allocInfo, nullptr, &vertexBufferMemory) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate vertex buffer memory!");
}
```

如果内存分配成功，那么我们现在可以使用 `vkBindBufferMemory` 将这个内存与缓冲区关联起来:

```cpp
vkBindBufferMemory(device, vertexBuffer, vertexBufferMemory, 0);
```

前三个参数是不言自明的，第四个参数是内存区域内的偏移量。因为这个内存是专门为这个顶点缓冲区分配的，所以偏移量只是0。如果偏移量为非零，则需要被 `memRequirements.alignment` 整除。

当然，就像 C++ 中的动态内存分配一样，内存应该在某个时刻被释放。一旦缓冲区不再使用，绑定到缓冲区对象的内存可能会被释放，所以让我们在缓冲区被销毁后释放它:

```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, vertexBuffer, nullptr);
    vkFreeMemory(device, vertexBufferMemory, nullptr);
```

#### Filling the vertex buffer 填充顶点缓冲区

现在是将顶点数据复制到缓冲区的时候了。这是通过使用 `vkMapMemory` 将缓冲区内存映射到 CPU 可访问内存来实现的。

```cpp
void* data;
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
```

这个函数允许我们访问由偏移量和大小定义的指定内存资源的区域。这里的偏移量和大小分别为`0`和 `bufferInfo.size`。还可以指定特殊值 `VK_WHOLE_SIZE`  来映射所有内存。倒数第二个参数可以用来指定标志，但是当前 API 中还没有任何可用的参数。它必须设置为值0。最后一个参数指定指向映射内存的指针的输出。

```cpp
void* data;
vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
memcpy(data, vertices.data(), (size_t) bufferInfo.size);
vkUnmapMemory(device, vertexBufferMemory);
```

您现在可以简单地使用 `memcpy` 将顶点数据映射到映射的内存中，然后再次使用 `vkUnmapMemory` 将其取消映射。不幸的是，驱动程序可能不会立即将数据复制到缓冲区内存中，例如，因为缓存。还有一种可能是，对缓冲区的写操作在映射的内存中还不可见。解决这个问题有两种方法:

- 使用主机一致的内存堆，用 `VK_MEMORY_PROPERTY_HOST_COHERENT_BIT`
- 在写入映射的内存后，调用 `vkInvalidateMappedMemoryRanges`,然后再从映射的内存中读取

我们采用了第一种方法，这种方法确保映射的内存始终与分配的内存的内容匹配。请记住，这可能会导致比显式刷新性能稍微差一点，但我们将在下一章中看到为什么这并不重要。

刷新内存范围或使用一致的内存堆意味着驱动程序将知道我们对缓冲区的写操作，但这并不意味着它们实际上在 GPU 上是可见的。向 GPU 传输数据是在后台进行的操作，规范只是告诉我们，在下一次调用 `vkQueueSubmit` 时，数据传输保证是完整的。

#### Binding the vertex buffer 绑定顶点缓冲区

现在剩下的就是在渲染操作期间绑定顶点缓冲区了。我们将扩展 `createCommandBuffers` 函数来做到这一点。

```cpp
vkCmdBindPipeline(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);

VkBuffer vertexBuffers[] = {vertexBuffer};
VkDeviceSize offsets[] = {0};
vkCmdBindVertexBuffers(commandBuffers[i], 0, 1, vertexBuffers, offsets);

vkCmdDraw(commandBuffers[i], static_cast<uint32_t>(vertices.size()), 1, 0, 0);
```

`vkCmdBindVertexBuffers` 函数用于将顶点缓冲区与绑定链接，就像我们在前一章中设置的那样。除了命令缓冲区之外，前两个参数指定我们要为其指定顶点缓冲区的偏移量和绑定数量。最后两个参数指定要绑定的顶点缓冲区数组和开始读取顶点数据的字节偏移量。您还应该更改对 `vkCmdDraw` 的调用，以传递缓冲区中的顶点数，而不是硬编码的数字3。

现在运行这个程序，你会再次看到熟悉的三角形

尝试通过修改顶点数组将顶点的颜色改为白色:

```cpp
const std::vector<Vertex> vertices = {
    {{0.0f, -0.5f}, {1.0f, 1.0f, 1.0f}},
    {{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
    {{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
};
```

再次运行该程序，您应该会看到以下内容:

![img1](img/顶点缓冲区演示.png)

在下一章中，我们将看到一种不同的方式来复制顶点数据到一个顶点缓冲区，这将带来更好的性能，但需要更多的工作。

### Staging buffer 分级缓冲器

#### Introduction 分级缓冲器 引言

我们现在使用的顶点缓冲区可以正常工作，但是允许我们从 CPU 访问它的内存类型可能不是显卡本身可以读取的最佳内存类型。最理想的内存具有 `VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT` 标志，并且通常不能被专用显卡上的CPU访问。在本章中，我们将创建两个顶点缓冲区。CPU 可访问内存中的一个分级缓冲区，用于将顶点数组中的数据上传到设备本地内存中的最终顶点缓冲区。然后，我们将使用一个缓冲区复制命令将数据从分级缓冲器移动到实际的顶点缓冲区。

#### Transfer queue 转移队列

缓冲区复制命令需要一个支持传输操作的队列族，这是使用 `VK_QUEUE_TRANSFER_BIT`  指示的。好消息是，任何具有 `VK_QUEUE_GRAPHICS_BIT`  或 `VK_QUEUE_COMPUTE_BIT`  功能的队列族已经隐式地支持 `VK_QUEUE_TRANSFER_BIT`  操作。在这些情况下，不需要实现在 `queueFlags` 中显式列出它。

如果您喜欢挑战，那么您仍然可以尝试使用不同的队列族专门用于转移操作。它将要求您对您的程序进行以下修改:

- 修改 `QueueFamilyIndices` 及 `findQueueFamilies` 方法来显式地查找队列族`VK_QUEUE_TRANSFER_BIT` ，但不是`VK_QUEUE_GRAPHICS_BIT`
- 修改 `createLogicalDevice` 请求传输队列的句柄
- 传输队列家族中提交的命令缓冲区创建第二个命令池
- 更改 `sharingMode` 资源的数量 `VK_SHARING_MODE_CONCURRENT` 并指定图形和转移队列族
- 提交任何传输命令，如 `vkCmdCopyBuffer` (我们将在本章中使用)到传输队列，而不是图形队列

这需要做一些工作，但是它将教会您很多关于队列族之间如何共享资源的知识。

#### Abstracting buffer creation 抽象缓冲区的创建

因为我们将在本章中创建多个缓冲区，所以将缓冲区创建转移到辅助函数是一个好主意。创建一个新函数 `createBuffer`，并将 `createVertexBuffer` 中的代码(映射除外)移动到该函数。

```cpp
void createBuffer(VkDeviceSize size, VkBufferUsageFlags usage, VkMemoryPropertyFlags properties, VkBuffer& buffer, VkDeviceMemory& bufferMemory) {
    VkBufferCreateInfo bufferInfo{};
    bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
    bufferInfo.size = size;
    bufferInfo.usage = usage;
    bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateBuffer(device, &bufferInfo, nullptr, &buffer) != VK_SUCCESS) {
        throw std::runtime_error("failed to create buffer!");
    }

    VkMemoryRequirements memRequirements;
    vkGetBufferMemoryRequirements(device, buffer, &memRequirements);

    VkMemoryAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &bufferMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate buffer memory!");
    }

    vkBindBufferMemory(device, buffer, bufferMemory, 0);
}
```

确保为缓冲区大小、内存属性和使用情况添加参数，这样我们就可以使用这个函数创建许多不同类型的缓冲区。最后两个参数是要将句柄写入的输出变量。

你现在可以从 `createVertexBuffer` 中删除缓冲区创建和内存分配代码，只需调用 `createBuffer` 即可:

```cpp
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();
    createBuffer(bufferSize, VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, vertexBuffer, vertexBufferMemory);

    void* data;
    vkMapMemory(device, vertexBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, vertexBufferMemory);
}
```

运行您的程序，以确保顶点缓冲区仍然正常工作。

#### Using a staging buffer 使用临时缓冲区

我们现在要修改 `createVertexBuffer` ，只使用主机可见缓冲区作为临时缓冲区，并使用设备本地缓冲区作为实际的顶点缓冲区。

```cpp
void createVertexBuffer() {
    VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
        memcpy(data, vertices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);
}
```

我们现在使用一个新的 `stagingBuffer` 和 `stagingBufferMemory` 来映射和复制顶点数据。在本章中，我们将使用两个新的缓冲区用法标志:

- `VK_BUFFER_USAGE_TRANSFER_SRC_BIT` 缓冲区可用作内存传输操作的源
- `VK_BUFFER_USAGE_TRANSFER_DST_BIT` 缓冲区可用作内存传输操作中的目标

`vertexBuffer` 现在是从设备本地的内存类型分配的，这通常意味着我们不能使用 `vkMapMemory` 。但是，我们可以将数据从 `stagingBuffer` 复制到 `vertexBuffer` 。我们必须指出我们打算通过为 `stagingBuffer` 指定传输源标志和为 `vertexBuffer` 指定传输目的地标志以及顶点缓冲区使用标志来实现这一点。

现在我们要编写一个函数，将内容从一个缓冲区复制到另一个缓冲区，称为 `copyBuffer` 。

```cpp
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {

}
```

内存传输操作使用命令缓冲区执行，就像绘图命令一样。因此，我们必须首先分配一个临时命令缓冲区。您可能希望为这些类型的短期缓冲区创建一个单独的命令池，因为实现可以应用内存分配优化。在这种情况下，您应该在命令池生成期间使用 `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT` 标志。

```cpp
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBufferAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);
}
```

然后立即开始记录命令缓冲:

```cpp
VkCommandBufferBeginInfo beginInfo{};
beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

vkBeginCommandBuffer(commandBuffer, &beginInfo);
```

我们只使用命令缓冲区一次，并且等待从函数返回，直到复制操作完成执行。因此使用 `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT` 告诉驱动程序我们的意图。

```cpp
VkBufferCopy copyRegion{};
copyRegion.srcOffset = 0; // Optional
copyRegion.dstOffset = 0; // Optional
copyRegion.size = size;
vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);
```

缓冲区的内容使用 `vkCmdCopyBuffer` 命令进行传输。它以源缓冲区和目标缓冲区作为参数，以及要复制的区域数组。这些区域在 `VkBufferCopy` 结构中定义，由源缓冲区偏移量、目标缓冲区偏移量和大小组成。不像 `vkMapMemory` 命令，这里不可能指定 `VK_WHOLE_SIZE` 。

```cpp
vkEndCommandBuffer(commandBuffer);
```

这个命令缓冲区只包含复制命令，因此我们可以在复制命令之后立即停止录制。现在执行命令缓冲区来完成传输:

```cpp
VkSubmitInfo submitInfo{};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffer;

vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
vkQueueWaitIdle(graphicsQueue);
```

与 `draw` 命令不同，这次我们不需要等待任何事件。我们只是想立即在缓冲区上执行传输。同样有两种可能的方式等待这个转移完成。我们可以使用 fence 并使用 `vkWaitForFences` 等待，或者简单地使用 `vkQueueWaitIdle` 等待传输队列空闲。Fence 允许您同时安排多个传输并等待所有传输完成，而不是一次执行一个。这可能给驱动更多的机会来优化。

```cpp
vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
```

不要忘记清理用于传输操作的命令缓冲区。

我们现在可以从 `createVertexBuffer` 函数调用 `copyBuffer` 来将顶点数据移动到设备本地缓冲区:

```cpp
createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);

copyBuffer(stagingBuffer, vertexBuffer, bufferSize);
```

在将数据从暂存缓冲区复制到设备缓冲区后，我们应该清理它:

```cpp
...

    copyBuffer(stagingBuffer, vertexBuffer, bufferSize);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

运行你的程序来验证你是否又看到了熟悉的三角形。这个改进现在可能看不到，但是它的顶点数据现在正在从高性能内存中加载。当我们开始渲染更多的复杂几何图形时，这个问题就变得很重要了。

#### Conclusion 分级缓冲器 总结

需要注意的是，在现实应用程序中，不应该为每个缓冲区实际调用 `vkallocatemory` 。同时内存分配的最大数量受限于 `maxmemorylocationcount` 物理设备限制，即使在高端硬件如 NVIDIA GTX 1080上也可能低至 `4096` 。同时为大量对象分配内存的正确方法是创建一个自定义分配器，该分配器通过使用我们在许多函数中看到的偏移量参数在许多不同对象之间分配单个内存。

您可以自己实现这样一个分配器，或者使用由 GPUOpen 提供的 [VulkanMemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator) 库。然而，对于本教程来说，对每个资源使用单独的分配是可以的，因为我们现在还没有达到这些限制中的任何一个。

### Index buffer 索引缓冲区

#### Introduction 索引缓冲区 引言

在真实世界的应用程序中渲染的3d网格通常会在多个三角形之间共享顶点。甚至像画一个矩形这样简单的事情也会这么做:

![img2](img/vertex_vs_index.svg)

绘制一个矩形需要两个三角形，这意味着我们需要一个顶点缓冲区和6个顶点。问题是，两个顶点的数据需要重复，从而产生50% 的冗余。对于更复杂的网格，这种情况只会变得更糟，顶点被平均3个三角形重复使用。这个问题的解决方案是使用索引缓冲区。

索引缓冲区实质上是指向顶点缓冲区的指针数组。它允许您对顶点数据进行重新排序，并为多个顶点重用现有数据。上面的插图演示了如果我们有一个顶点缓冲区包含四个唯一顶点中的每一个，矩形的索引缓冲区会是什么样子。前三个索引定义直角上三角形，后三个索引定义左下三角形的顶点。

#### Index buffer creation 创建索引缓冲区

在本章中，我们将修改顶点数据并添加索引数据来绘制一个像图中那样的矩形。修改顶点数据来表示四个角:

```cpp
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}},
    {{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}},
    {{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}},
    {{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}}
};
```

左上角是红色，右上角是绿色，右下角是蓝色，左下角是白色。我们将添加一个新的数组索引来表示索引缓冲区的内容。它应该匹配图中的索引来绘制右上角三角形和左下角三角形。

```cpp
const std::vector<uint16_t> indices = {
    0, 1, 2, 2, 3, 0
};
```

根据顶点中条目的数量，可以为索引缓冲区使用 `uint16_t` 或 `uint32_t` 。我们现在可以坚持使用 `uint16_t`，因为我们使用的唯一顶点数少于65535。

就像顶点数据，索引需要上传到一个 `VkBuffer` 才能使得 GPU 能够访问他们。定义两个新的类成员来保存索引缓冲区的资源:

```cpp
VkBuffer vertexBuffer;
VkDeviceMemory vertexBufferMemory;
VkBuffer indexBuffer;
VkDeviceMemory indexBufferMemory;
```

我们现在要添加的 `createIndexBuffer` 函数与 `createVertexBuffer` 几乎完全相同:

```cpp
void initVulkan() {
    ...
    createVertexBuffer();
    createIndexBuffer();
    ...
}

void createIndexBuffer() {
    VkDeviceSize bufferSize = sizeof(indices[0]) * indices.size();

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
    memcpy(data, indices.data(), (size_t) bufferSize);
    vkUnmapMemory(device, stagingBufferMemory);

    createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_INDEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, indexBuffer, indexBufferMemory);

    copyBuffer(stagingBuffer, indexBuffer, bufferSize);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

只有两个显著的区别。 `bufferSize` 现在等于索引数量乘以索引类型的大小，可以是 `uint16_t` 或 `uint32_t` 。 `indexBuffer` `的使用应该是VK_BUFFER_USAGE_INDEX_BUFFER_BIT` ，而不是 `VK_BUFFER_USAGE_VERTEX_BUFFER_BIT` ，这是有意义的。除此之外，过程是完全一样的。我们创建一个临时缓冲区，将索引的内容复制到，然后将其复制到最终的设备本地索引缓冲区。

索引缓冲区应该在程序结束时清理，就像顶点缓冲区一样:

```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyBuffer(device, indexBuffer, nullptr);
    vkFreeMemory(device, indexBufferMemory, nullptr);

    vkDestroyBuffer(device, vertexBuffer, nullptr);
    vkFreeMemory(device, vertexBufferMemory, nullptr);

    ...
}
```

#### Using an index buffer 使用索引缓冲区

使用索引缓冲区绘图涉及对 `createCommandBuffers` 的两个更改。我们首先需要绑定索引缓冲区，就像我们做的顶点缓冲区。区别在于您只能有一个索引缓冲区。不幸的是，不可能为每个顶点属性使用不同的索引，所以我们仍然必须完全复制顶点数据，即使只有一个属性发生变化。

```cpp
vkCmdBindVertexBuffers(commandBuffers[i], 0, 1, vertexBuffers, offsets);

vkCmdBindIndexBuffer(commandBuffers[i], indexBuffer, 0, VK_INDEX_TYPE_UINT16);
```

索引缓冲区与 `vkCmdBindIndexBuffer` 绑定，后者具有索引缓冲区、字节偏移量和索引数据类型作为参数。如前所述，可能的类型是 `VK_INDEX_TYPE_UINT16` 和 `VK_INDEX_TYPE_UINT32` 。

仅仅绑定一个索引缓冲区并不会改变任何东西，我们还需要更改 drawing 命令来告诉 Vulkan 使用索引缓冲区。用 `vkCmdDrawIndexed` 替换 `vkCmdDraw` :

```cpp
vkCmdDrawIndexed(commandBuffers[i], static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

对这个函数的调用非常类似于 `vkCmdDraw` 。除了指定命令缓冲区外，前两个参数指定索引的数量和实例的数量。我们没有使用实例，所以只需指定`1`个实例。索引数表示将传递给顶点缓冲区的顶点数。下一个参数指定索引缓冲区中的偏移量，使用值1将导致显卡从第二个索引处开始读取。倒数第二个参数指定要添加到索引缓冲区中的索引的偏移量。最后一个参数为实例指定了一个偏移量，我们没有使用它。

现在运行你的程序，你应该会看到以下内容:

![img3](img/index%20buffer.png)

现在您知道了如何通过使用索引缓冲区重用顶点来节省内存。这在我们将要加载复杂的3d模型的未来一章中将变得尤为重要。

上一章已经提到，您应该分配多个资源，比如从单个内存分配中分配缓冲区，但实际上您应该更进一步。驱动程序开发人员建议您将多个缓冲区(如顶点缓冲区和索引缓冲区)存储到单个 `VkBuffer` 中，并在命令中使用偏移量(如 `vkCmdBindVertexBuffers`)。在这种情况下，优点是您的数据缓存更友好，因为它们之间的距离更近。如果在相同的呈现操作期间没有使用相同的内存块，甚至可以为多个资源重新使用相同的内存块，当然前提是要刷新它们的数据。这就是所谓的别名，一些 Vulkan 函数有明确的标志来指定要执行此操作。

## Uniform buffers 统一缓冲区

### Descriptor layout and buffer 描述符布局和缓冲区

#### Introduction 统一缓冲区引言

我们现在可以将任意属性的顶点传递给顶点着色器，但是全局变量呢？我们将从这一章开始讨论3D 图形，这需要一个模型-视图-投影矩阵。我们可以将它作为顶点数据包含进去，但是这样会浪费内存，而且每当转换发生变化时，我们都需要更新顶点缓冲区。变换可以很容易地改变每一帧。

在 Vulkan 解决这个问题的正确方法是使用资源描述符。描述符是着色器自由访问资源(如缓冲区和映像)的一种方式。我们将设置一个包含转换矩阵的缓冲区，并让顶点着色器通过描述符访问它们。描述符的使用包括三个部分:

- 在管道创建期间指定描述符布局
- 从描述符池中分配描述符集
- 在渲染期间绑定描述符集

描述符布局指定管道将要访问的资源类型，就像渲染通道指定将要访问的附件类型一样。描述符集指定将绑定到描述符的实际缓冲区或图像资源，就像framebuffer指定要绑定到渲染通道附件的实际图像视图一样。描述符集合随后被绑定为绘图命令，就像顶点缓冲区和framebuffer一样。

有许多类型的描述符，但是在本章中我们将使用统一的缓冲区对象(UBO)。我们将在以后的章节中研究其他类型的描述符，但基本过程是相同的。假设我们在 c 结构中有一个顶点着色器需要的数据，如下所示:

```cpp
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

然后我们可以将数据复制到 `VkBuffer`，并通过统一的缓冲区对象描述符从顶点着色器访问它，如下所示:

```cpp
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

我们将更新每一帧的模型、视图和投影矩阵，使前一章中的矩形在3D 中旋转。

#### Vertex shader  顶点着色器

修改顶点着色器以包含上面指定的统一缓冲区对象。我假设您熟悉 MVP 转换。如果你没有，看看第一章中提到的[资源](https://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/)。

```cpp
#version 450

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

请注意， `uniform` , `in` 和 `out` 的顺序并不重要。绑定指令类似于属性的 `location` 指令。我们将在描述符布局中引用这个绑定。带有 `gl_Position` 的行被更改为使用转换来计算剪辑坐标中的最终位置。与2D三角形不同的是，剪辑坐标的最后一个分量可能不是1，当转换为屏幕上最终的标准化设备坐标时，这个分量会是除法的结果。这在透视投影中用作透视分割，对于使近距离的物体看起来比远距离的物体更大是必不可少的。

#### Descriptor set layout 描述符集布局

下一步是在 C++ 端定义 UBO，并在 Vulkan 中告诉顶点着色器中关于这个描述符的定义。

```cpp
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

我们可以使用 GLM 中的数据类型精确地匹配着色器中的定义。矩阵中的数据以着色器期望的方式进行二进制兼容，因此稍后我们可以通过 `memcpy` 将一个 `UniformBufferObject` 映射到 `VkBuffer` 。

我们需要提供关于用于管线创建的着色器中所使用的每个描述符绑定的详细信息，就像我们必须为每个顶点属性及其 `location` 索引所做的那样。我们将设置一个新函数来定义所有这些信息，该函数称为 `createDescriptorSetLayout` 。它应该在管线创建之前被调用，因为我们需要管线中使用它。

```cpp
void initVulkan() {
    ...
    createDescriptorSetLayout();
    createGraphicsPipeline();
    ...
}

...

void createDescriptorSetLayout() {

}
```

每个绑定都需要通过 `VkDescriptorSetLayoutBinding` 结构进行描述。

```cpp
void createDescriptorSetLayout() {
    VkDescriptorSetLayoutBinding uboLayoutBinding{};
    uboLayoutBinding.binding = 0;
    uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
    uboLayoutBinding.descriptorCount = 1;
}
```

前两个字段指定着色器中使用的绑定和描述符的类型，描述符是统一的缓冲区对象。着色器变量可以表示统一缓冲区对象的数组， `descriptorCount` 指定数组中的值数。例如，这可以用来为一个骨架中的每个骨骼指定一个转换骨骼动画。我们的 `MVP` 转换位于单个统一的缓冲区对象中，因此我们使用的 `descriptorCount` 为 `1` 。

```cpp
uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
```

我们还需要指定描述符将被引用的着色器阶段。 `stageFlags` 字段可以是 `VkShaderStageFlagBits` 值或值 `VK_SHADER_STAGE_ALL_GRAPHICS` 的组合。在我们的例子中，我们只引用顶点着色器中的描述符。

```cpp
uboLayoutBinding.pImmutableSamplers = nullptr; // Optional
```

`pImmutableSamplers` 字段仅与图像采样相关的描述符有关，我们将在后面讨论这一点。您可以将其保留为默认值。

所有的描述符绑定都被组合成一个 `VkDescriptorSetLayout` 对象,在 `pipelineLayout` 上定义一个新的类成员

```cpp
VkDescriptorSetLayout descriptorSetLayout;
VkPipelineLayout pipelineLayout;
```

然后我们可以使用 `vkCreateDescriptorSetLayout` 创建它，这个函数接受一个简单的 `VkDescriptorSetLayoutCreateInfo` 和绑定数组:

```cpp
VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = 1;
layoutInfo.pBindings = &uboLayoutBinding;

if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create descriptor set layout!");
}
```

我们需要在管道创建期间指定描述符集布局，以告诉 Vulkan 着色器将使用哪些描述符。描述符集布局在管道布局对象中指定。修改 `VkPipelineLayoutCreateInfo` 以引用布局对象:

```cpp
VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 1;
pipelineLayoutInfo.pSetLayouts = &descriptorSetLayout;
```

您可能想知道为什么可以在这里指定多个描述符集布局，因为一个描述符集已经包含了所有绑定。我们将在下一章回到这个问题，在那里我们将研究描述符池和描述符集。

在我们创建新的图形管道时，描述符布局应该保持不变，直到程序结束:

```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);

    ...
}
```

#### Uniform buffer

在下文中，我们将为着色器指定包含 UBO 数据的缓冲区，但是我们首先需要创建这个缓冲区。我们将每帧都将新数据复制到统一缓冲区，因此使用临时缓冲区实际上没有任何意义。在这种情况下，它只会增加额外的开销，可能会降低性能，而不是提高性能。

我们应该有多个缓冲区，因为多个帧可能在同一时间飞行，我们不想在准备下一帧时更新缓冲区，而前一帧仍在读取它！我们可以为每个帧或每个交换链映像提供一个Uniform buffer。但是，由于我们需要引用每个交换链映像中的命令缓冲区的Uniform buffer，因此对每个交换链映像使用一个Uniform buffer最为合理。

为此，添加新的类成员 `uniformBuffers` 和 `uniformBuffersMemory`

```cpp
VkBuffer indexBuffer;
VkDeviceMemory indexBufferMemory;

std::vector<VkBuffer> uniformBuffers;
std::vector<VkDeviceMemory> uniformBuffersMemory;
```

类似地，创建一个新的函数 `createUniformBuffers` ，在 `createIndexBuffer` 之后调用，并分配缓冲区:

```cpp
void initVulkan() {
    ...
    createVertexBuffer();
    createIndexBuffer();
    createUniformBuffers();
    ...
}

...

void createUniformBuffers() {
    VkDeviceSize bufferSize = sizeof(UniformBufferObject);

    uniformBuffers.resize(swapChainImages.size());
    uniformBuffersMemory.resize(swapChainImages.size());

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        createBuffer(bufferSize, VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, uniformBuffers[i], uniformBuffersMemory[i]);
    }
}
```

我们将编写一个单独的函数，每帧用一个新的转换更新统一缓冲区，所以这里不会有 `vkMapMemory` 。统一数据将用于所有的绘制调用，因此只有当我们停止渲染时，包含它的缓冲区才应该被销毁。因为它还取决于交换链映像的数量，这些映像在重新创建后可能会发生变化，所以我们将在 `cleanupSwapChain` 中清理它:

```cpp
void cleanupSwapChain() {
    ...

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        vkDestroyBuffer(device, uniformBuffers[i], nullptr);
        vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
    }
}
```

这意味着我们还需要在 `recreateSwapChain` 中重新创建它:

```cpp
void recreateSwapChain() {
    ...

    createFramebuffers();
    createUniformBuffers();
    createCommandBuffers();
}
```

#### Updating uniform data

创建一个新的函数 `updateUniformBuffer` ，在我们知道要获取哪个交换链映像之后，从 `drawFrame` 函数中添加一个对它的调用:

```cpp
void drawFrame() {
    ...

    uint32_t imageIndex;
    VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    ...

    updateUniformBuffer(imageIndex);

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

    ...
}

...

void updateUniformBuffer(uint32_t currentImage) {

}
```

这个函数将在每一帧中生成一个新的变换，使几何体旋转。我们需要包含两个新的头文件来实现这个功能:

```cpp
#define GLM_FORCE_RADIANS
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

#include <chrono>
```

`glm/gtc/matrix_transform.hpp`头文件公开的函数可用于生成模型转换，如  `glm::rotate`，视图转换，如 `glm::lookAt`，以及投影转换，如`glm::perspective`。`GLM_FORCE_RADIANS` 定义对于确保像 `glm::rotate` 这样的函数使用弧度作为参数是必要的，以避免任何可能的混淆。

`Chrono` 标准库头公开了执行精确计时的函数。我们将用这个来确保几何图形每秒旋转90度，与帧速率无关。

```cpp
void updateUniformBuffer(uint32_t currentImage) {
    static auto startTime = std::chrono::high_resolution_clock::now();

    auto currentTime = std::chrono::high_resolution_clock::now();
    float time = std::chrono::duration<float, std::chrono::seconds::period>(currentTime - startTime).count();
}
```

`updateUniformBuffer` 函数将从一些逻辑开始，计算以秒为单位的时间，因为渲染已经以单精度浮点开始。

现在我们将在统一的缓冲区对象中定义模型、视图和投影变换。模型旋转将是一个简单的使用时间变量围绕 z 轴旋转:

```cpp
UniformBufferObject ubo{};
ubo.model = glm::rotate(glm::mat4(1.0f), time * glm::radians(90.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

`glm::rotate` 函数以现有的转换、旋转角度和旋转轴为参数。 `glm::mat4(1.0f)` 构造函数返回一个单位矩阵。使用的旋转角度为 `time * glm::radians(90.0f)` 以实现每秒旋转90°

```cpp
ubo.view = glm::lookAt(glm::vec3(2.0f, 2.0f, 2.0f), glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

对于视图转换，我决定从上以45度的角度来看几何图形。`glm::lookAt`函数以眼睛位置、中心位置和上轴为参数。

```cpp
ubo.proj = glm::perspective(glm::radians(45.0f), swapChainExtent.width / (float) swapChainExtent.height, 0.1f, 10.0f);
```

我选择了一个45度垂直视场的透视投影。其他参数是宽高比，近视图面和远视图面。重要的是使用当前交换链范围来计算长宽比，以考虑调整窗口大小后的新宽度和高度。

```cpp
ubo.proj[1][1] *= -1;
```

GLM 最初是为 OpenGL 设计的，其中剪辑坐标的 y 坐标是反转的。最简单的补偿方法是在投影矩阵中翻转 y 轴的缩放因子上的符号。如果您不这样做，那么图像将呈现颠倒。

现在已经定义了所有的转换，因此我们可以将统一缓冲区对象中的数据复制到当前的统一缓冲区。这和我们对顶点缓冲区的处理方式完全一样，除了没有分级缓冲区:

```cpp
void* data;
vkMapMemory(device, uniformBuffersMemory[currentImage], 0, sizeof(ubo), 0, &data);
    memcpy(data, &ubo, sizeof(ubo));
vkUnmapMemory(device, uniformBuffersMemory[currentImage]);
```

以这种方式使用 UBO 并不是将频繁更改的值传递给着色器的最有效方法。将一小块数据缓冲区传递给着色器的一种更有效的方法是推送常量。我们可以在以后的章节中看到这些。

在下一章中，我们将研究描述符集，它实际上将 VkBuffers 绑定到统一缓冲区描述符，以便着色器可以访问这个转换数据。

### Descriptor pool and sets 描述符池和集

#### Introduction 描述符池和集引言

上一章描述符的布局描述了可以绑定的描述符的类型。在本章中，我们将为每个 `VkBuffer` 资源创建描述符集，以将其绑定到统一缓冲区描述符。

#### Descriptor pool 描述符池

描述符集不能直接创建，它们必须像命令缓冲区一样从池中分配。描述符集的等价物毫不奇怪地称为描述符池。我们将编写一个新函数 `createDescriptorPool` 来设置它。

```cpp
void initVulkan() {
    ...
    createUniformBuffers();
    createDescriptorPool();
    ...
}

...

void createDescriptorPool() {

}
```

我们首先需要使用 `VkDescriptorPoolSize` 结构描述描述符集将包含哪些描述符类型以及它们的数量。

```cpp
VkDescriptorPoolSize poolSize{};
poolSize.type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
poolSize.descriptorCount = static_cast<uint32_t>(swapChainImages.size());
```

我们将为每个帧分配一个这样的描述符，这个池大小的结构将参考主 `VkDescriptorPoolCreateInfo` :

```cpp
VkDescriptorPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
poolInfo.poolSizeCount = 1;
poolInfo.pPoolSizes = &poolSize;
```

除了可用的个别描述符的最大数量，我们还需要指定可以分配的描述符集的最大数量:

```cpp
poolInfo.maxSets = static_cast<uint32_t>(swapChainImages.size());
```

该结构有一个类似于命令池的可选标志，用于确定是否可以释放单个描述符集: `VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT` 。我们不会在创建描述符集之后触及它，所以我们不需要这个标志。您可以将标志保留为其默认值 `0` 。

```cpp
VkDescriptorPool descriptorPool;

...

if (vkCreateDescriptorPool(device, &poolInfo, nullptr, &descriptorPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create descriptor pool!");
}
```

添加一个新的类成员来存储描述符池的句柄，并调用 `vkCreateDescriptorPool` 来创建它。在重新创建交换链时，描述符池应该被销毁，因为它取决于映像的数量:

```cpp
void cleanupSwapChain() {
    ...

    for (size_t i = 0; i < swapChainImages.size(); i++) {
        vkDestroyBuffer(device, uniformBuffers[i], nullptr);
        vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
    }

    vkDestroyDescriptorPool(device, descriptorPool, nullptr);
}
```

并在 `recreateSwapChain` 中重新创建:

```cpp
void recreateSwapChain() {
    ...

    createUniformBuffers();
    createDescriptorPool();
    createCommandBuffers();
}
```

#### Descriptor set 描述符集

我们现在可以分配描述符集本身，为此添加一个 `createDescriptorSets` 函数:

```cpp
void initVulkan() {
    ...
    createDescriptorPool();
    createDescriptorSets();
    ...
}

void recreateSwapChain() {
    ...
    createDescriptorPool();
    createDescriptorSets();
    ...
}

...

void createDescriptorSets() {

}
```

描述符集分配用 `VkDescriptorSetAllocateInfo` 结构描述。您需要指定要分配的描述符池，要分配的描述符集的数量，以及基于它们的描述符布局:

```cpp
std::vector<VkDescriptorSetLayout> layouts(swapChainImages.size(), descriptorSetLayout);
VkDescriptorSetAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
allocInfo.descriptorPool = descriptorPool;
allocInfo.descriptorSetCount = static_cast<uint32_t>(swapChainImages.size());
allocInfo.pSetLayouts = layouts.data();
```

在本例中，我们将为每个交换链映像创建一个描述符集，所有描述符都具有相同的布局。不幸的是，我们确实需要布局的所有副本，因为下一个函数需要一个与集数相匹配的数组。

添加一个类成员来保存描述符集句柄，并用 `vkAllocateDescriptorSets` 分配它们:

```cpp
VkDescriptorPool descriptorPool;
std::vector<VkDescriptorSet> descriptorSets;

...

descriptorSets.resize(swapChainImages.size());
if (vkAllocateDescriptorSets(device, &allocInfo, descriptorSets.data()) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate descriptor sets!");
}
```

您不需要显式地清除描述符集，因为当描述符池被销毁时，它们将被自动释放。对 `vkAllocateDescriptorSets` 的调用将分配描述符集，每个描述符集有一个uniform buffer描述符。

现在已经分配了描述符集，但仍然需要配置描述符集中的描述符。现在我们将添加一个循环来填充每个描述符:

```cpp
for (size_t i = 0; i < swapChainImages.size(); i++) {

}
```

对缓冲区的描述符，如我们的uniform buffer描述符，使用 `VkDescriptorBufferInfo` 结构进行配置。此结构指定缓冲区和其中包含描述符数据的区域。

```cpp
for (size_t i = 0; i < swapChainImages.size(); i++) {
    VkDescriptorBufferInfo bufferInfo{};
    bufferInfo.buffer = uniformBuffers[i];
    bufferInfo.offset = 0;
    bufferInfo.range = sizeof(UniformBufferObject);
}
```

如果您正在覆盖整个缓冲区，就像我们在本例中所做的那样，那么也可以对范围使用 `VK_WHOLE_SIZE`  值。描述符的配置使用 `vkUpdateDescriptorSets` 函数更新，该函数以 `VkWriteDescriptorSet` 结构的数组作为参数。

```cpp
VkWriteDescriptorSet descriptorWrite{};
descriptorWrite.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrite.dstSet = descriptorSets[i];
descriptorWrite.dstBinding = 0;
descriptorWrite.dstArrayElement = 0;
```

前两个字段指定要更新的描述符集和绑定。我们给出了统一的缓冲区绑定索引0。记住，描述符可以是数组，因此我们还需要指定要更新的数组中的第一个索引。我们没有使用数组，因此索引只是0。

```cpp
descriptorWrite.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorWrite.descriptorCount = 1;
```

我们需要再次指定描述符的类型。可以在一个数组中同时更新多个描述符，从索引 `dstArrayElement` 开始。 `descriptorCount` 字段指定要更新的数组元素的数量。

```cpp
descriptorWrite.pBufferInfo = &bufferInfo;
descriptorWrite.pImageInfo = nullptr; // Optional
descriptorWrite.pTexelBufferView = nullptr; // Op
```

最后一个字段引用了一个具有 `descriptorCount` 结构的数组，该结构实际上配置了描述符。这取决于描述符的类型，您实际上需要使用三个描述符之一。 `pBufferInfo` 字段用于引用缓冲区数据的描述符，pImageInfo 用于引用图像数据的描述符， `pTexelBufferView` 用于引用缓冲区视图的描述符。我们的描述符是基于缓冲区的，所以我们使用了 `pBufferInfo` 。

```cpp
vkUpdateDescriptorSets(device, 1, &descriptorWrite, 0, nullptr);
```

使用 `vkUpdateDescriptorSets` 应用更新。它接受两种数组作为参数: `VkWriteDescriptorSet`  数组和 `VkCopyDescriptorSet` 数组。顾名思义，后者可以用来相互复制描述符。

#### Using descriptor sets 使用描述符集

现在，我们需要更新 `createCommandBuffers` 函数，以便使用 `vkCmdBindDescriptorSets` 将每个交换链映像的正确描述符集绑定到着色器中的描述符。这需要在 `vkCmdDrawIndexed` 调用之前完成:

```cpp
vkCmdBindDescriptorSets(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, 1, &descriptorSets[i], 0, nullptr);
vkCmdDrawIndexed(commandBuffers[i], static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

与顶点和索引缓冲区不同，描述符集对于图形管道不是唯一的。因此，我们需要指定是否要将描述符集绑定到图形或计算管道。下一个参数是描述符所基于的布局。接下来的三个参数指定第一个描述符集的索引、要绑定的集的数量和要绑定的集的数组。我们一会儿再回到这个话题。最后两个参数指定用于动态描述符的偏移量数组。我们将在以后的章节中讨论这些问题。

如果您现在运行您的程序，那么您将注意到，不幸的是，什么都不可见。问题是，由于我们在投影矩阵中所做的 y 型翻转，顶点现在是按逆时针顺序绘制的，而不是顺时针顺序。这将导致反面剔除的生效，并防止任何几何体被绘制。转到 `createGraphicsPipeline` `函数，修改VkPipelineRasterizationStateCreateInfo` 中的 `frontFace` ，以更正以下内容:

```cpp
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE;
```

再次运行你的程序，你现在应该看到以下内容:

![img4](img/UniformBuffer.png)

由于投影矩阵对长宽比进行了校正，矩形变成了正方形。 `updateUniformBuffer` 负责调整屏幕大小，因此我们不需要在 `recreateSwapChain` 中重新创建描述符集。

#### Alignment requirements 对齐要求

到目前为止，我们一直忽略的一件事是 C++ 结构中的数据应该如何与着色器中的统一定义匹配。在两者中使用相同的类型似乎已经足够明显:

```cpp
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

例如，尝试修改结构体和着色器，使其看起来像这样:

```cpp
struct UniformBufferObject {
    glm::vec2 foo;
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    vec2 foo;
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

重新编译你的着色器和你的程序，并运行它，你会发现你工作到目前为止的彩色矩形已经消失！这是因为我们没有考虑到对齐的要求。

Vulkan 希望结构中的数据在内存中以特定的方式对齐，例如:

- 标量必须按 N (= 4个字节，32位浮点数) 对齐
- `vec2` 必须按 2N (= 8字节)对齐
- `vec3` 或 `vec4` 必须按 4N (= 16字节)对齐
- 嵌套结构必须按照其成员的基本对齐方式 (四舍五入为16的倍数) 进行对齐
- `mat4` 矩阵必须与 `vec4` 有相同的对其方式

您可以在[规范](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/chap14.html#interfaces-resources-layout)中找到对齐要求的完整列表。

我们的原始着色器只有三个 `mat4` 字段已经满足了对齐的要求。因为每个 `mat4` 的大小是4 x 4 x 4 = 64个字节，`model` 的偏移量为`0`，`view` 的偏移量为`64`，`proj` 的偏移量为`128`。所有这些都是16的倍数，这就是为什么它运行良好的原因。

新的结构以 `vec2` 开始，它的大小只有 `8` 个字节，因此丢弃了所有的偏移量。现在`model`的偏移量是8，`view`的偏移量是72，`proj` 的偏移量是136，没有一个是16的倍数。为了解决这个问题，我们可以使用 C++11 中引入的 `alignas` 说明符:

```cpp
struct UniformBufferObject {
    glm::vec2 foo;
    alignas(16) glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

如果你现在编译并再次运行你的程序，你会看到着色器再次正确地接收到它的矩阵值。

幸运的是，有一种方法可以让你在大多数时间里不必考虑这些校准要求。我们可以在include GLM 之前定义 `GLM_FORCE_DEFAULT_ALIGNED_GENTYPES` :

```cpp
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEFAULT_ALIGNED_GENTYPES
#include <glm/glm.hpp>
```

这将迫使GLM使用一个已经为我们指定了对齐要求的`vec2`和`mat4`版本。如果您添加了这个定义，那么您可以删除alignas说明符，您的程序仍然可以工作。

不幸的是，如果你开始使用嵌套结构，这个方法可能会崩溃:

```cpp
struct Foo {
    glm::vec2 v;
};

struct UniformBufferObject {
    Foo f1;
    Foo f2;
};
```

下面是着色器的定义:

```cpp
struct Foo {
    vec2 v;
};

layout(binding = 0) uniform UniformBufferObject {
    Foo f1;
    Foo f2;
} ubo;
```

在这种情况下，f2的偏移量为`8`，而它的偏移量应该为`16`，因为它是一个嵌套结构。在这种情况下，你必须自己指定对齐方式:

```cpp
struct UniformBufferObject {
    Foo f1;
    alignas(16) Foo f2;
};
```

这些陷阱是明确对齐的一个很好的理由。这样，您就不会被对齐错误的奇怪症状搞得措手不及。

```cpp
struct UniformBufferObject {
    alignas(16) glm::mat4 model;
    alignas(16) glm::mat4 view;
    alignas(16) glm::mat4 proj;
};
```

不要忘记在删除 `foo` 字段后重新编译你的 shader。

#### Multiple descriptor sets 多个描述符集

正如一些结构和函数调用暗示的那样，实际上可以同时绑定多个描述符集。在创建管道布局时，您需要为每个描述符集指定一个描述符布局。然后着色器可以像这样引用特定的描述符集:

```cpp
layout(set = 0, binding = 0) uniform UniformBufferObject { ... }
```

您可以使用这个特性来放置每个对象不同的描述符，以及共享到单独的描述符集中的描述符。在这种情况下，可以避免跨 draw 调用重新绑定大多数描述符，这可能会更有效。

## Texture mapping 纹理映射

### Images 图片

#### Introduction 引言 图片

到目前为止，几何体已经使用每个顶点的颜色着色，这是一个相当有限的方法。在本教程的这一部分，我们将实现纹理映射，使几何体看起来更有趣。这也将允许我们在以后的章节中加载和绘制基本的3d模型。

为我们的应用程序添加纹理包括以下步骤:

- 创建一个由设备内存支持的映像对象
- 用图像文件中的像素填充它
- 创建一个图像采样器
- 添加一个组合的图像采样描述符以从纹理中采样颜色

我们以前已经使用过映像对象，但是这些对象是通过交换链扩展自动创建的。这次我们必须自己创造一个。创建一个图像并用数据填充它类似于创建顶点缓冲区。我们将首先创建一个 staging 资源并用像素数据填充它，然后将其复制到最终用于渲染的图像对象。尽管可以为此目的创建一个临时映像，Vulkan 还允许您将像素从 `VkBuffer` 复制到一个映像，而且在某些硬件上这样做的 API 实际上更快。我们将首先创建这个缓冲区，并用像素值填充它，然后创建一个图像来复制像素。创建映像与创建缓冲区没有很大的区别。它包括查询内存需求，分配设备内存并绑定它，就像我们之前看到的那样。

然而，在处理图像时，我们还需要注意一些额外的事情。图像可以有不同的布局，影响像素在内存中的组织方式。例如，由于图形硬件的工作方式，简单地逐行存储像素可能无法获得最佳性能。在对图像执行任何操作时，必须确保它们具有在该操作中使用的最佳布局。实际上，当我们指定渲染通道时，我们已经看到了其中的一些布局:

- `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR` 最适合展示
- `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL` 作为从片段着色器中绘制颜色的最佳附件
- `VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL` 作为转移操作的最佳来源，比如 `vkCmdCopyImageToBuffer`
- `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL` 作为转移操作的最佳目的地，比如 `vkCmdCopyBufferToImage`
- `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL` 从着色器中取样的最佳选择

转换图像布局的最常见方式之一是管道屏障。管道屏障主要用于同步对资源的访问，比如确保在读取图像之前对其已经完全写入，但它们也可用于转换布局。在本章中，我们将看到管道屏障是如何实现此目的的。当使用 `VK_SHARING_MODE_EXCLUSIVE` 时，还可以使用障碍来转移队列族所有权。

#### Image library 图像库

有许多库可用于加载图像，您甚至可以编写自己的代码来加载简单的格式，如 BMP 和 PPM。在本教程中，我们将使用 stb 集合中的 `stb_image` 库。它的优点是所有代码都在一个文件中，因此不需要任何复杂的构建配置。下载 `stb_image.h` 并将其存储在一个方便的位置，比如保存 GLFW 和 GLM 的目录。将位置添加到包含路径中。

[stb](https://github.com/nothings/stb)

#### Loading an image 加载图像

像这样包含图片库:

```cpp
#define STB_IMAGE_IMPLEMENTATION
#include <stb_image.h>
```

头文件默认情况下只定义函数的原型。一个代码文件需要包含带有 `STB_IMAGE_IMPLEMENTATION` 定义的头，以便包含函数主体，否则我们会得到链接错误。

```cpp
void initVulkan() {
    ...
    createCommandPool();
    createTextureImage();
    createVertexBuffer();
    ...
}

...

void createTextureImage() {

}
```

创建一个新的函数 `createTextureImage` ，我们将加载一个图像并将其上传到 Vulkan 图像对象中。我们将使用命令缓冲区，因此应该在 `createCommandPool` 之后调用它。

在`shaders`目录旁边创建一个新的`textures`目录来存储纹理图像。我们将从该目录加载一个名为 `texture.jpg` 的映像。我选择使用下面的 CC0许可图像缩放到512 x 512像素，但请随意选择任何你想要的图像。该库支持最常见的图像文件格式，如 JPEG、 PNG、 BMP 和 GIF。

使用这个库加载图片非常简单:

```cpp
void createTextureImage() {
    int texWidth, texHeight, texChannels;
    stbi_uc* pixels = stbi_load("textures/texture.jpg", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
    VkDeviceSize imageSize = texWidth * texHeight * 4;

    if (!pixels) {
        throw std::runtime_error("failed to load texture image!");
    }
}
```

`Stbi_load` 函数将文件路径和要加载的通道数作为参数。`STBI_rgb_alpha`值强制图像加载 alpha 通道，即使它没有alpha通道，这对将来与其他纹理的一致性非常好。中间的三个参数是图像中的宽度、高度和实际通道数的输出。返回的指针是像素值数组中的第一个元素。对于 `texWidth * texHeight * 4` 的总值，像素以每像素4字节的方式逐行排列，使用`STBI_rgb_alpha`。

#### Staging buffer 分级缓冲区

我们现在要在主机可见内存中创建一个缓冲区，这样我们就可以使用 `vkMapMemory` 并将像素复制到它。将这个临时缓冲区的变量添加到 `createTextureImage` 函数:

```cpp
VkBuffer stagingBuffer;
VkDeviceMemory stagingBufferMemory;
```

缓冲区应该在主机可见内存中，这样我们可以映射它，它应该可以作为一个传输源，这样我们可以在之后将它复制到一个映像上:

```cpp
createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);
```

然后我们可以直接将从图像加载库获得的像素值复制到缓冲区:

```cpp
void* data;
vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
    memcpy(data, pixels, static_cast<size_t>(imageSize));
vkUnmapMemory(device, stagingBufferMemory);
```

现在不要忘记清理原始的像素数组:

```cpp
stbi_image_free(pixels);
```

#### Texture Image 纹理图像

虽然我们可以设置着色器来访问缓冲区中的像素值，但是最好使用 Vulkan 中的图像对象来实现这个目的。图像对象将使检索颜色更容易和更快，因为我们可以使用二维坐标。图像对象中的像素被称为 texels，我们将从这一点开始使用这个名称。添加以下新类成员:

```cpp
VkImage textureImage;
VkDeviceMemory textureImageMemory;
```

图像的参数是在 `VkImageCreateInfo` 结构中指定的:

```cpp
VkImageCreateInfo imageInfo{};
imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
imageInfo.imageType = VK_IMAGE_TYPE_2D;
imageInfo.extent.width = static_cast<uint32_t>(texWidth);
imageInfo.extent.height = static_cast<uint32_t>(texHeight);
imageInfo.extent.depth = 1;
imageInfo.mipLevels = 1;
imageInfo.arrayLayers = 1;
```

在 `imageType` 字段中指定的图像类型，告诉 Vulkan 图像中的文本将被寻址到何种坐标系。创建一维、二维和三维图像是可能的。一维图像可用于存储数据数组或梯度，二维图像主要用于纹理，三维图像可用于存储体素体积，例如。`extent` 字段指定图像的尺寸，基本上就是每个轴上有多少个 texels 。这就是为什么 `depth` 必须是`1`而不是`0`。我们的纹理将不是一个数组，同时我们现在不会使用 mipmapping。

```cpp
imageInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
```

Vulkan 支持许多可能的图像格式，但是我们应该对 texels 使用与缓冲区中的像素相同的格式，否则复制操作将失败。

```cpp
imageInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
```

`tiling` 字段可以有以下两个值之一:

- `VK_IMAGE_TILING_LINEAR` 像我们的像素数组一样，texel是按行顺序排列的
- `VK_IMAGE_TILING_OPTIMAL` texel按照实现定义的最佳访问顺序排列

与图像的布局不同，平铺模式不能在以后更改。如果你想直接访问图像内存中的文本，那么你必须使用 `VK_IMAGE_TILING_LINEAR`。我们将使用临时缓冲区而不是临时映像，因此不需要这样做。我们将使用 `VK_IMAGE_TILING_OPTIMAL` 从着色器进行高效访问。

```cpp
imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
```

一个图片的 `initialLayout` 只有两个可能的值:

- `VK_IMAGE_LAYOUT_UNDEFINED` 无法被GPU使用且第一个变换将丢弃texel
- `VK_IMAGE_LAYOUT_PREINITIALIZED` 无法被GPU使用但是第一个变换将保留texel

在很少的情况下，需要在第一次转换时保留texel。然而，一个例子是，如果您想结合`VK_IMAGE_TILING_LINEAR`布局使用一个图像作为分段图像。在这种情况下，您需要上传texel数据到它，然后将图像转换为传输源而不丢失数据。然而，在我们的例子中，我们首先要将图像转换为传输目的地，然后将texel数据从缓冲区对象复制到它，所以我们不需要这个属性，可以安全地使用`VK_IMAGE_LAYOUT_UNDEFINED`。

```cpp
imageInfo.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
```

`usage` 字段具有与创建缓冲区时相同的语义。映像将用作缓冲区副本的目的地，因此应该将其设置为传输目的地。我们还希望能够从着色器访问图像来为我们的网格着色，所以使用应该包括 `VK_IMAGE_USAGE_SAMPLED_BIT`。

```cpp
imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

映像将只被一个队列族使用:支持图形传输操作(因此也支持)的队列族。

`samples` 标志与重采样有关。这只适用于将作为附件使用的图像，所以坚持使用一次采样。对于与稀疏图像相关的图像，有一些可选的标志。稀疏图像是指只有特定区域使用内存存储的图像。例如，如果对体素地形使用3D 纹理，那么可以使用它来避免分配内存来存储大量的“空气”值。我们不会在本教程中使用它，所以让它保持默认值0。

```cpp
if (vkCreateImage(device, &imageInfo, nullptr, &textureImage) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image!");
}
```

图片是使用 `vkCreateImage` 创建的，没有任何特别值得注意的参数。图形硬件有可能不支持 `VK_FORMAT_R8G8B8A8_SRGB`  格式。你您应该有一个可接受的替代方案列表，并选择受支持的最佳方案。但是，对这种特殊格式的支持非常普遍，我们将跳过这一步。使用不同的格式也需要烦人的转换。我们将在深度缓冲区一章回到这个问题，在那里我们将实现这样一个系统。

```cpp
VkMemoryRequirements memRequirements;
vkGetImageMemoryRequirements(device, textureImage, &memRequirements);

VkMemoryAllocateInfo allocInfo{};
allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
allocInfo.allocationSize = memRequirements.size;
allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);

if (vkAllocateMemory(device, &allocInfo, nullptr, &textureImageMemory) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate image memory!");
}

vkBindImageMemory(device, textureImage, textureImageMemory, 0);
```

为映像分配内存的工作方式与为缓冲区分配内存完全相同。除了使用 `vkGetImageMemoryRequirements` 而不是 `vkGetBufferMemoryRequirements` ，并使用 `vkBindImageMemory` 而不是 `vkBindBufferMemory` 。

这个函数已经变得相当大了，在后面的章节中需要创建更多的图像，所以我们应该将图像创建抽象到 createImage 函数中，就像我们对缓冲区所做的那样。创建函数并将图像对象的创建和内存分配移动到它:

```cpp
void createImage(uint32_t width, uint32_t height, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory) {
    VkImageCreateInfo imageInfo{};
    imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
    imageInfo.imageType = VK_IMAGE_TYPE_2D;
    imageInfo.extent.width = width;
    imageInfo.extent.height = height;
    imageInfo.extent.depth = 1;
    imageInfo.mipLevels = 1;
    imageInfo.arrayLayers = 1;
    imageInfo.format = format;
    imageInfo.tiling = tiling;
    imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
    imageInfo.usage = usage;
    imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
    imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

    if (vkCreateImage(device, &imageInfo, nullptr, &image) != VK_SUCCESS) {
        throw std::runtime_error("failed to create image!");
    }

    VkMemoryRequirements memRequirements;
    vkGetImageMemoryRequirements(device, image, &memRequirements);

    VkMemoryAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
    allocInfo.allocationSize = memRequirements.size;
    allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

    if (vkAllocateMemory(device, &allocInfo, nullptr, &imageMemory) != VK_SUCCESS) {
        throw std::runtime_error("failed to allocate image memory!");
    }

    vkBindImageMemory(device, image, imageMemory, 0);
}
```

我已经设置了宽度、高度、格式、平铺模式、使用和内存属性参数，因为这些参数在整个教程中我们将创建的图像之间都会有所不同。

现在 `createTextureImage` 函数可以简化为:

```cpp
void createTextureImage() {
    int texWidth, texHeight, texChannels;
    stbi_uc* pixels = stbi_load("textures/texture.jpg", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
    VkDeviceSize imageSize = texWidth * texHeight * 4;

    if (!pixels) {
        throw std::runtime_error("failed to load texture image!");
    }

    VkBuffer stagingBuffer;
    VkDeviceMemory stagingBufferMemory;
    createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

    void* data;
    vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
        memcpy(data, pixels, static_cast<size_t>(imageSize));
    vkUnmapMemory(device, stagingBufferMemory);

    stbi_image_free(pixels);

    createImage(texWidth, texHeight, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, textureImage, textureImageMemory);
}
```

#### Layout transitions 布局转换

我们现在要写的函数涉及到记录和执行一个命令缓冲区，所以现在是时候把这个逻辑转移到一个或两个辅助函数中:

```cpp
VkCommandBuffer beginSingleTimeCommands() {
    VkCommandBufferAllocateInfo allocInfo{};
    allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
    allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
    allocInfo.commandPool = commandPool;
    allocInfo.commandBufferCount = 1;

    VkCommandBuffer commandBuffer;
    vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);

    VkCommandBufferBeginInfo beginInfo{};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

    vkBeginCommandBuffer(commandBuffer, &beginInfo);

    return commandBuffer;
}

void endSingleTimeCommands(VkCommandBuffer commandBuffer) {
    vkEndCommandBuffer(commandBuffer);

    VkSubmitInfo submitInfo{};
    submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
    submitInfo.commandBufferCount = 1;
    submitInfo.pCommandBuffers = &commandBuffer;

    vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
    vkQueueWaitIdle(graphicsQueue);

    vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
}
```

这些函数的代码基于 `copyBuffer` 中的现有代码，现在可以将该函数简化为:

```cpp
void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    VkBufferCopy copyRegion{};
    copyRegion.size = size;
    vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);

    endSingleTimeCommands(commandBuffer);
}
```

如果我们仍然使用缓冲区，那么我们现在可以编写一个函数来记录并执行 `vkCmdCopyBufferToImage` 来完成工作，但是这个命令要求图像首先处于正确的布局中。创建一个新函数来处理布局转换:

```cpp
void transitionImageLayout(VkImage image, VkFormat format, VkImageLayout oldLayout, VkImageLayout newLayout) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    endSingleTimeCommands(commandBuffer);
}
```

执行布局转换最常见的方法之一是使用图像存储屏障。这样的管道屏障通常用于同步对资源的访问，比如确保对缓冲区的写入在从缓冲区读取之前完成，但当使用`VK_SHARING_MODE_EXCLUSIVE`时，它也可以用于转换图像布局和传输队列族所有权。有一个等效的缓冲区内存屏障来为缓冲区实现这一点。

```cpp
VkImageMemoryBarrier barrier{};
barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
barrier.oldLayout = oldLayout;
barrier.newLayout = newLayout;
```

前两个字段指定布局转换。如果不关心图像的现有内容，可以使用`VK_IMAGE_LAYOUT_UNDEFINED`作为`oldLayout`。

```cpp
barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
```

如果您正在使用屏障转移队列族所有权，那么这两个字段应该是队列族的索引。如果您不想这样做，它们必须被设置为`VK_QUEUE_FAMILY_IGNORED`(不是默认值!)。

```cpp
barrier.image = image;
barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
barrier.subresourceRange.baseMipLevel = 0;
barrier.subresourceRange.levelCount = 1;
barrier.subresourceRange.baseArrayLayer = 0;
barrier.subresourceRange.layerCount = 1;
```

`image`和`subresourceRange`指定受影响的映像和映像的特定部分。我们的图像不是数组，也没有mipmapping级别，所以只指定了一个级别和层。

```cpp
barrier.srcAccessMask = 0; // TODO
barrier.dstAccessMask = 0; // TODO
```

`barrier`主要用于同步目的，因此您必须指定涉及资源的哪些操作必须在`barrier`之前发生，涉及资源的哪些操作必须在`barrier`上等待。尽管已经使用`vkQueueWaitIdle`来手动同步，但我们仍然需要这样做。正确的值取决于新旧布局，所以一旦我们确定了要使用哪个过渡，我们就会回到这个问题上。

```cpp
vkCmdPipelineBarrier(
    commandBuffer,
    0 /* TODO */, 0 /* TODO */,
    0,
    0, nullptr,
    0, nullptr,
    1, &barrier
);
```

所有类型的管道屏障都使用相同的函数提交。命令缓冲区之后的第一个参数指定了应该在屏障之前发生的操作在哪个管道阶段发生。第二个参数指定管道阶段，操作将在该阶段中等待屏障。允许您在屏障之前和之后指定的管道阶段取决于您如何在屏障之前和之后使用资源。允许的值列在此规格表中。例如，如果你打算在`barrier`之后从`uniform`读取，你应该指定使用`VK_ACCESS_UNIFORM_READ_BIT`和将从`uniform`读取的最早的着色器作为管道阶段，例如`VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT`。为这种类型的使用指定一个非着色器管道阶段是没有意义的，当你指定一个管道阶段不匹配的使用类型时，验证层会警告你。

第三个参数为`0`或`VK_DEPENDENCY_BY_REGION_BIT`。后者将障碍变成了每个区域的条件。例如，这意味着该实现允许已经开始读取到目前为止所写入的资源部分。

最后三对参数代表了三种可用类型的管道屏障数组: 内存屏障、缓冲内存屏障和我们正在使用的图像内存屏障。请注意，我们还没有使用 `VkFormat` 参数，但是我们将在深度缓冲区一章中使用该参数进行特殊转换。

#### Copying buffer to image 将缓冲区复制到图像

在回到`createTextureImage`之前，我们要再写一个辅助函数:`copyBufferToImage`:

```cpp
void copyBufferToImage(VkBuffer buffer, VkImage image, uint32_t width, uint32_t height) {
    VkCommandBuffer commandBuffer = beginSingleTimeCommands();

    endSingleTimeCommands(commandBuffer);
}
```

就像缓冲区复制一样，您需要指定缓冲区的哪一部分将被复制到图像的哪一部分。这是通过`VkBufferImageCopy`结构发生的:

```cpp
VkBufferImageCopy region{};
region.bufferOffset = 0;
region.bufferRowLength = 0;
region.bufferImageHeight = 0;

region.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
region.imageSubresource.mipLevel = 0;
region.imageSubresource.baseArrayLayer = 0;
region.imageSubresource.layerCount = 1;

region.imageOffset = {0, 0, 0};
region.imageExtent = {
    width,
    height,
    1
};
```

这些字段中的大多数都是不言自明的。`bufferOffset`指定像素值在缓冲区中开始的字节偏移量。`bufferRowLength`和`bufferImageHeight`字段指定了像素在内存中的布局方式。例如，在图像的行之间可以有一些填充字节。为两者指定0表示像素只是像在我们的例子中那样紧密地压缩。`imageSubresource`、`imageOffset`和`imageExtent`字段指出了我们想要复制的像素的图像的哪一部分。

使用vkCmdCopyBufferToImage函数执行将从缓冲区复制到图像操作:

```cpp
vkCmdCopyBufferToImage(
    commandBuffer,
    buffer,
    image,
    VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
    1,
    &region
);
```

第四个参数表示图像当前使用的布局。我在这里假设图像已经过渡到最适合复制像素的布局。现在我们只将一块像素复制到整个图像中，但是可以指定`VkBufferImageCopy`数组来在一个操作中从这个缓冲区到图像执行许多不同的复制。

#### Preparing the texture image 准备纹理图像

现在我们已经有了设置纹理图像所需的所有工具，因此我们将回到 `createTextureImage` 函数。我们在这里做的最后一件事是创建纹理图像。下一步是将 staging buffer 复制到纹理图像。这包括两个步骤:

- 将纹理图像转换为 `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`
- 执行缓冲区到图像复制操作

可以很轻松的使用我们之前创建的函数来实现

```cpp
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL);
copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
```

这个图像是用`VK_IMAGE_LAYOUT_UNDEFINED`布局创建的，所以在转换为`textureImage`时应该指定旧的布局。请记住，我们可以这样做，因为在执行复制操作之前，我们并不关心它的内容。

为了能够从着色器中的纹理图像开始采样，我们需要最后一个转换来为着色器访问做准备:

```cpp
transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);
```

#### Transition barrier masks 转换屏障遮罩

如果你现在运行应用程序时启用了验证层，那么你会看到它报告`transitionImageLayout`中的访问遮罩和管道阶段无效。我们仍然需要根据过渡中的布局来设置这些。

我们需要处理两个过渡:

- 未定义→传输目的地:传输不需要等待任何东西的写入
- 传输目的地→着色器读取:着色器读取应该等待传输写入，特别是着色器读取片段着色器，因为那是我们将要使用纹理的地方

这些规则是使用以下访问遮罩和管道阶段指定的:

```cpp
// transitionImageLayout
VkPipelineStageFlags sourceStage;
VkPipelineStageFlags destinationStage;

if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL) {
    barrier.srcAccessMask = 0;
    barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;

    sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
    destinationStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
} else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL && newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL) {
    barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
    barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

    sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
    destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
} else {
    throw std::invalid_argument("unsupported layout transition!");
}

vkCmdPipelineBarrier(
    commandBuffer,
    sourceStage, destinationStage,
    0,
    0, nullptr,
    0, nullptr,
    1, &barrier
);
```

正如您在前面的代码中所看到的，传输写必须发生在管道传输阶段。因为写操作不需要等待任何东西，所以可以为barrier之前的操作指定一个空的访问掩码和可能最早的管道阶段`VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`。需要注意的是，`VK_PIPELINE_STAGE_TRANSFER_BIT`并不是图形和计算管道中的一个真正的阶段。这更像是一个转移发生的伪阶段。有关伪阶段的更多信息和其他示例，请参阅[文档](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/man/html/VkPipelineStageFlagBits.html)。

图像将在相同的管道阶段写入，然后由片段着色器读取，这就是为什么我们在片段着色器管道阶段指定着色器读取访问。

如果我们将来需要做更多的转换，那么我们将扩展这个函数。应用程序现在应该可以成功运行了，尽管当然还没有视觉上的改变。

需要注意的一点是，命令缓冲区提交在开始时会导致隐式的`VK_ACCESS_HOST_WRITE_BIT`同步。因为`transitionImageLayout`函数只使用一个命令执行一个命令缓冲区，所以如果您在布局转换中需要`VK_ACCESS_HOST_WRITE_BIT`依赖，您可以使用这个隐式同步并将`srcAccessMask`设置为0。这取决于你是否想要明确它，但我个人并不喜欢依赖这些类似opengl的“隐藏”操作。

实际上，有一种特殊类型的图像布局支持所有操作，即`VK_IMAGE_LAYOUT_GENERAL`。当然，它的问题是，它不一定为任何操作提供最佳性能。在一些特殊情况下需要它，比如使用图像作为输入和输出，或者在图像离开预初始化布局后读取图像。

到目前为止，所有提交命令的助手函数都设置为通过等待队列空闲来同步执行。对于实际应用程序，建议将这些操作组合在一个命令缓冲区中，并异步执行它们，以获得更高的吞吐量，特别是`createTextureImage`函数中的转换和复制。通过创建帮助函数记录命令的`setupCommandBuffer`，并添加一个`flushSetupCommands`来执行迄今为止已经记录的命令，尝试进行这方面的实验。最好是在纹理映射工作之后，检查纹理资源是否仍然设置正确。

#### Cleanup  清理

完成 createTextureImage 函数，在结束时清理 staging 缓冲区及其内存:

```cpp
    transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);

    vkDestroyBuffer(device, stagingBuffer, nullptr);
    vkFreeMemory(device, stagingBufferMemory, nullptr);
}
```

使用主纹理图像直到程序结束:

```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);

    ...
}
```

图像现在包含了纹理，但我们仍然需要一种方式来从图形管道访问它。我们将在下一章讨论。

### Image view and sampler 图像视图和取样器

在本章中，我们将创建两个更多的资源，用于图形管道对图像进行采样。第一个资源是我们之前在处理交换链图像时已经见过的，但第二个是新的-它与着色器如何从图像中读取texel有关。

#### Texture image view 纹理图像视图

我们已经看到，使用交换链图像和framebuffer，图像是通过图像视图而不是直接访问的。我们还需要为纹理图像创建这样的图像视图。

添加一个类成员来保存纹理图像的 `VkImageView` ，并创建一个新的函数 `createTextureImageView` ，我们将在其中创建它:

```cpp
VkImageView textureImageView;

...

void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createVertexBuffer();
    ...
}

...

void createTextureImageView() {

}
```

这个函数的代码可以直接基于`createImageViews`。你只需要改变`format`和`image`:

```cpp
VkImageViewCreateInfo viewInfo{};
viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
viewInfo.image = textureImage;
viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
viewInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
viewInfo.subresourceRange.baseMipLevel = 0;
viewInfo.subresourceRange.levelCount = 1;
viewInfo.subresourceRange.baseArrayLayer = 0;
viewInfo.subresourceRange.layerCount = 1;
```

我省略了显式的`viewInfo.components`初始化，因为`VK_COMPONENT_SWIZZLE_IDENTITY`被定义为0。通过调用`vkCreateImageView`完成图像视图的创建:

```cpp
if (vkCreateImageView(device, &viewInfo, nullptr, &textureImageView) != VK_SUCCESS) {
    throw std::runtime_error("failed to create texture image view!");
}
```

因为很多逻辑都是从 `createImageViews` 中复制出来的，所以你可能希望把它抽象成一个新的 `createImageView` 函数:

```cpp
VkImageView createImageView(VkImage image, VkFormat format) {
    VkImageViewCreateInfo viewInfo{};
    viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
    viewInfo.image = image;
    viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
    viewInfo.format = format;
    viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
    viewInfo.subresourceRange.baseMipLevel = 0;
    viewInfo.subresourceRange.levelCount = 1;
    viewInfo.subresourceRange.baseArrayLayer = 0;
    viewInfo.subresourceRange.layerCount = 1;

    VkImageView imageView;
    if (vkCreateImageView(device, &viewInfo, nullptr, &imageView) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture image view!");
    }

    return imageView;
}
```

现在 `createTextureImageView` 函数可以简化为:

```cpp
void createTextureImageView() {
    textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB);
}
```

并且 `createImageViews` 可以简化为:

```cpp
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

    for (uint32_t i = 0; i < swapChainImages.size(); i++) {
        swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat);
    }
}
```

确保在程序结束的时候销毁图像视图，就在销毁图像本身之前:

```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroyImageView(device, textureImageView, nullptr);

    vkDestroyImage(device, textureImage, nullptr);
    vkFreeMemory(device, textureImageMemory, nullptr);
```

#### Samplers 采样器

着色器可以直接从图像中读取texel，但当它们被用作纹理时，这并不常见。纹理通常通过采样器访问，采样器将应用过滤和转换来计算最终检索到的颜色。

这些滤波器有助于处理过采样等问题。考虑一个映射到具有更多片段而不是texel的几何图形的纹理。如果你简单地在每个片段中取纹理坐标中最近的texel，那么你会得到像第一张图片一样的结果:

![texture_filtering](img/texture_filtering.png)

如果你通过线性插值结合4个最接近的texel，那么你会得到一个像右边一样平滑的结果。当然，你的应用程序可能有更适合左风格的美术风格要求(如《我的世界》)，但在传统图形应用程序中，右风格是首选。当你从纹理中读取颜色时，采样器对象会自动为你应用这个过滤。

采样不足是相反的问题，你有比片段更多的texel。这将导致在以尖角采样像棋盘纹理这样的高频模式时产生伪影:

![anisotropic_filtering](img/anisotropic_filtering.png)

如图所示，纹理在远处变得模糊混乱。解决这个问题的方法是[各向异性过滤](https://en.wikipedia.org/wiki/Anisotropic_filtering)，它也可以由采样器自动应用。

除了这些过滤器之外，采样器还可以处理转换。它决定了当你试图通过它的寻址模式读取图像外的texel时会发生什么。下图展示了一些可能性:

![texture_addressing](img/texture_addressing.png)

我们现在将创建一个函数 `createTextureSampler` 来设置这样一个采样对象。稍后我们将使用这个取样器从着色器的纹理中读取颜色。

```cpp
void initVulkan() {
    ...
    createTextureImage();
    createTextureImageView();
    createTextureSampler();
    ...
}

...

void createTextureSampler() {

}
```

采样器是通过 `VkSamplerCreateInfo` 结构配置的，该结构指定应该应用的所有过滤器和转换。

```cpp
VkSamplerCreateInfo samplerInfo{};
samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
samplerInfo.magFilter = VK_FILTER_LINEAR;
samplerInfo.minFilter = VK_FILTER_LINEAR;
```

`magFilter`和`minFilter`字段指定了如何插值放大或缩小的texel。放大涉及上面描述的过采样问题，缩小涉及向下采样问题。可选`VK_FILTER_NEAREST`和`VK_FILTER_LINEAR`，即最邻近与线性插值。

```cpp
samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
samplerInfo.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;
```

可以使用`addressMode`字段为每个轴指定寻址模式。下面列出了可用的值。上面的图片展示了其中的大部分。注意轴被称为U, V和W，而不是X, Y和z。这是纹理空间坐标的约定。

- `VK_SAMPLER_ADDRESS_MODE_REPEAT` 当超出图像尺寸时，重复纹理
- `VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEA` 与repeat相似，但是当超出坐标范围的时候，会反转坐标来镜像图像
- `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE` 图像坐标范围以外取最接近坐标的边缘的颜色。
- `VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE` 与上相似，但使用相反的边缘代替最接近的边缘
- `VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER` 当采样超过图像的尺寸时返回纯色

我们在这里使用哪种寻址模式并不重要，因为在本教程中我们不会对图像以外的部分进行取样。然而，重复模式可能是最常见的模式，因为它可以用于瓷砖纹理，如地板和墙壁。

```cpp
samplerInfo.anisotropyEnable = VK_TRUE;
samplerInfo.maxAnisotropy = ???;
```

这两个字段指定是否应该使用各向异性过滤。除非考虑性能问题，否则没有理由不使用它。`maxAnisotropy`字段限制了可用于计算最终颜色的纹理样本的数量。值越低，性能越好，但质量越低。为了找出我们可以使用哪个值，我们需要像这样检索物理设备的属性:

```cpp
VkPhysicalDeviceProperties properties{};
vkGetPhysicalDeviceProperties(physicalDevice, &properties);
```

如果您查看`VkPhysicalDeviceProperties`结构的文档，您将看到它包含一个名为`limits`的`VkPhysicalDeviceLimits`成员。这个结构体又有一个名为`maxSamplerAnisotropy`的成员，这是我们可以为`maxAnisotropy`指定的最大值。如果我们想要获得最大质量，我们可以直接使用这个值:

```cpp
samplerInfo.maxAnisotropy = properties.limits.maxSamplerAnisotropy;
```

您可以在程序的开头查询这些属性，并将它们传递给需要它们的函数，或者在 `createTextureSampler` 函数本身查询这些属性。

```cpp
samplerInfo.borderColor = VK_BORDER_COLOR_INT_OPAQUE_BLACK;
```

`borderColor`字段指定当使用clamp to border寻址模式对图像进行采样时返回的颜色。可以以float或int格式中返回黑色、白色或透明。不能指定任意颜色

```cpp
samplerInfo.unnormalizedCoordinates = VK_FALSE;
```

`unnormalizedCoordinates`字段指定了你想用哪个坐标系来处理图像中的texel。如果这个字段是`VK_TRUE`，那么你可以简单地在`[0,texWidth)`和`[0,texHeight)`范围内使用坐标。如果它是`VK_FALSE`，那么所有轴上的texel都使用`[0,1)`范围进行处理。现实世界的应用程序几乎总是使用标准化的坐标，因为这样就可以使用具有完全相同坐标的不同分辨率的纹理。

```cpp
samplerInfo.compareEnable = VK_FALSE;
samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;
```

如果启用了比较函数，那么texels将首先与一个值进行比较，比较的结果将用于筛选操作。这主要用于阴影贴图的[percentage-closer filtering](https://developer.nvidia.com/gpugems/GPUGems/gpugems_ch11.html)。我们将在以后的章节中讨论这个问题。

```cpp
samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
samplerInfo.mipLodBias = 0.0f;
samplerInfo.minLod = 0.0f;
samplerInfo.maxLod = 0.0f;
```

以上所有这些字段都将用于 mipmapping。我们将在后面的章节中讨论 mipmapping，但基本上它是另一种可以应用的过滤器。

采样器的功能现在已经完全确定。添加一个类成员来持有采样器对象的句柄，并使用`vkCreateSampler`创建采样器:

```cpp
VkImageView textureImageView;
VkSampler textureSampler;

...

void createTextureSampler() {
    ...

    if (vkCreateSampler(device, &samplerInfo, nullptr, &textureSampler) != VK_SUCCESS) {
        throw std::runtime_error("failed to create texture sampler!");
    }
}
```

注意，采样器没有在任何地方引用`VkImage`。采样器是一个独特的对象，它提供了从纹理中提取颜色的接口。它可以应用于任何你想要的图像，无论是1D、2D还是3D。这与许多较老的api不同，后者将纹理图像和过滤组合成单个状态。

当我们不再访问图片时，在程序结束时销毁取样器:

```cpp
void cleanup() {
    cleanupSwapChain();

    vkDestroySampler(device, textureSampler, nullptr);
    vkDestroyImageView(device, textureImageView, nullptr);

    ...
}
```

#### Anisotropy device feature 各向异性器件特性

如果你现在运行你的程序，你会看到这样的验证层消息:

![Anisotropy](img/Anisotropy.png)

这是因为各向异性过滤实际上是一个可选的设备功能。我们需要更新`createLogicalDevice`函数来请求它:

```cpp
VkPhysicalDeviceFeatures deviceFeatures{};
deviceFeatures.samplerAnisotropy = VK_TRUE;
```

尽管现代显卡不太可能不支持它，但我们应该更新 `isDeviceSuitable` 来检查它是否可用:

```cpp
bool isDeviceSuitable(VkPhysicalDevice device) {
    ...

    VkPhysicalDeviceFeatures supportedFeatures;
    vkGetPhysicalDeviceFeatures(device, &supportedFeatures);

    return indices.isComplete() && extensionsSupported && swapChainAdequate && supportedFeatures.samplerAnisotropy;
}
```

`vkGetPhysicalDeviceFeatures` 重用 `VkPhysicalDeviceFeatures` 结构来指示哪些功能是支持的，而不是通过设置布尔值来请求的。

除了加强各向异性过滤的可用性，也可以通过设置条件而不使用它:

```cpp
samplerInfo.anisotropyEnable = VK_FALSE;
samplerInfo.maxAnisotropy = 1.0f;
```

在下一章中，我们将把图像和采样器对象暴露给着色器，以便在正方形上绘制纹理。

### Combined image sampler 组合图像采样器

#### Introduction  引言

我们第一次在教程的uniform buffers部分研究了描述符。在本章中，我们将看到一种新的描述符: 组合图像采样器。这个描述符使得着色器可以通过一个采样器对象访问图像资源，就像我们在前一章中创建的那样。

我们将首先修改描述符布局、描述符池和描述符集，以包含这样的组合图像采样描述符。在此之后，我们将添加纹理坐标至`Vertex`和修改片段着色器从纹理读取颜色，而不是仅仅通过顶点插值获取颜色。

#### Updating the descriptors 更新描述符

浏览`createDescriptorSetLayout`函数，并为组合图像采样描述符添加一个`VkDescriptorSetLayoutBinding`。我们将简单地把它放在uniform buffers之后的绑定中:

```cpp
VkDescriptorSetLayoutBinding samplerLayoutBinding{};
samplerLayoutBinding.binding = 1;
samplerLayoutBinding.descriptorCount = 1;
samplerLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
samplerLayoutBinding.pImmutableSamplers = nullptr;
samplerLayoutBinding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;

std::array<VkDescriptorSetLayoutBinding, 2> bindings = {uboLayoutBinding, samplerLayoutBinding};
VkDescriptorSetLayoutCreateInfo layoutInfo{};
layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
layoutInfo.bindingCount = static_cast<uint32_t>(bindings.size());
layoutInfo.pBindings = bindings.data();
```

确保设置 `stageFlags` 以表明我们打算在片段着色器中使用组合图像采样描述符。这就是确定片段颜色的地方。我们可以在顶点着色器中使用纹理采样，例如，通过[高度图](https://en.wikipedia.org/wiki/Heightmap)动态变形顶点网格。

我们还必须创建一个更大的描述符池，通过将另一个类型为`VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER`的`VkPoolSize`添加到`VkDescriptorPoolCreateInfo`中来为组合图像采样器的分配腾出空间。转到`createDescriptorPool`函数并修改它，使其包含这个描述符的`VkDescriptorPoolSize`:

```cpp
std::array<VkDescriptorPoolSize, 2> poolSizes{};
poolSizes[0].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
poolSizes[0].descriptorCount = static_cast<uint32_t>(swapChainImages.size());
poolSizes[1].type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
poolSizes[1].descriptorCount = static_cast<uint32_t>(swapChainImages.size());

VkDescriptorPoolCreateInfo poolInfo{};
poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
poolInfo.poolSizeCount = static_cast<uint32_t>(poolSizes.size());
poolInfo.pPoolSizes = poolSizes.data();
poolInfo.maxSets = static_cast<uint32_t>(swapChainImages.size());
```

某些问题验证层不会捕获，不足的描述符池就是一个很好的例子: 在 Vulkan 1.1中，如果描述符池空间不足，`vkAllocateDescriptorSets` 可能会失败，错误代码为 `VK_ERROR_POOL_OUT_OF_MEMORY`，但是驱动程序也可能尝试在内部解决这个问题。这意味着有时候(取决于硬件、池大小和分配大小)驱动程序会让我们得到超出描述符池限制的分配。而另外一些时候，`vkAllocateDescriptorSets` 将失败并返回 `VK_ERROR_POOL_OUT_OF_MEMORY`。若分配在某些机器上成功，但在其他机器上失败，会是一件让特别令人沮丧的事。

因为 Vulkan 将分配的责任转移到驱动程序，所以不再严格要求只分配由相应的`descriptorCount`指定的相同数量的特定类型的描述符(`VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER`，等等)。然而，我们还是应该按照指定的要求来做，若以后启用[Best Practice Validation](https://vulkan.lunarg.com/doc/view/1.1.126.0/windows/best_practices.html)，`VK_LAYER_KHRONOS_validation` 将会警告这种类型的问题。

最后一步是将实际图像和采样器资源绑定到描述符集中的描述符。转到 `createDescriptorSets` 函数。

```cpp
for (size_t i = 0; i < swapChainImages.size(); i++) {
    VkDescriptorBufferInfo bufferInfo{};
    bufferInfo.buffer = uniformBuffers[i];
    bufferInfo.offset = 0;
    bufferInfo.range = sizeof(UniformBufferObject);

    VkDescriptorImageInfo imageInfo{};
    imageInfo.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
    imageInfo.imageView = textureImageView;
    imageInfo.sampler = textureSampler;

    ...
}
```

组合图像采样器结构的资源必须在 `VkDescriptorImageInfo` 结构中指定，就像在 `VkDescriptorBufferInfo` 结构中指定uniform buffer描述符的缓冲区资源一样。这就是上一章中的对象聚集到一起的地方。

```cpp
std::array<VkWriteDescriptorSet, 2> descriptorWrites{};

descriptorWrites[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[0].dstSet = descriptorSets[i];
descriptorWrites[0].dstBinding = 0;
descriptorWrites[0].dstArrayElement = 0;
descriptorWrites[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
descriptorWrites[0].descriptorCount = 1;
descriptorWrites[0].pBufferInfo = &bufferInfo;

descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
descriptorWrites[1].dstSet = descriptorSets[i];
descriptorWrites[1].dstBinding = 1;
descriptorWrites[1].dstArrayElement = 0;
descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
descriptorWrites[1].descriptorCount = 1;
descriptorWrites[1].pImageInfo = &imageInfo;

vkUpdateDescriptorSets(device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
```

就像缓冲区一样，描述符必须用这个图像信息更新。这次我们使用的是 `pImageInfo` 数组而不是 `pBufferInfo` 。描述符现在已经准备好被着色器使用了

#### Texture coordinates 纹理坐标

纹理映射还缺少一个重要的组成部分，那就是每个顶点的实际坐标。这些坐标决定了图像实际上是如何映射到几何体的。

```cpp
struct Vertex {
    glm::vec2 pos;
    glm::vec3 color;
    glm::vec2 texCoord;

    static VkVertexInputBindingDescription getBindingDescription() {
        VkVertexInputBindingDescription bindingDescription{};
        bindingDescription.binding = 0;
        bindingDescription.stride = sizeof(Vertex);
        bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

        return bindingDescription;
    }

    static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        attributeDescriptions[1].binding = 0;
        attributeDescriptions[1].location = 1;
        attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[1].offset = offsetof(Vertex, color);

        attributeDescriptions[2].binding = 0;
        attributeDescriptions[2].location = 2;
        attributeDescriptions[2].format = VK_FORMAT_R32G32_SFLOAT;
        attributeDescriptions[2].offset = offsetof(Vertex, texCoord);

        return attributeDescriptions;
    }
};
```

修改 `Vertex` 结构以包含一个 `vec2` 表示纹理坐标。一定要添加一个 `VkVertexInputAttributeDescription` ，这样我们就可以使用纹理坐标作为顶点着色器的输入。这是必要的，以便能够通过它们的片段着色器插值整个正方形的表面。

```cpp
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f},{1.0f,0.0f}},
    {{ 0.5f, -0.5f}, {0.0f, 1.0f, 0.0f},{0.0f,0.0f}},
    {{ 0.5f,  0.5f}, {0.0f, 0.0f, 1.0f},{0.0f,1.0f}},
    {{-0.5f,  0.5f}, {1.0f, 1.0f, 1.0f},{1.0f,1.0f}}
};
```

在本教程中，我将使用从左上角的`0,0`到右下角的`1,1`的坐标简单地用纹理填充正方形。你可以随意尝试不同的坐标。尝试使用0以下或以上1的坐标，看看寻址模式的行动！

#### Shaders 着色器

最后一步是修改着色器从纹理中取样颜色。我们首先需要修改顶点着色器，以便通过纹理坐标传递到片段着色器:

```GLSL
layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;
layout(location = 2) in vec2 inTexCoord;

layout(location = 0) out vec3 fragColor;
layout(location = 1) out vec2 fragTexCoord;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

就像每个顶点的颜色一样， `fragTexCoord` 值将通过光栅化器平滑插值到正方形区域中。我们可以通过使用片段着色器将纹理坐标输出为颜色来实现可视化:

```GLSL
#version 450

layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragTexCoord, 0.0, 1.0);
}
```

您应该看到类似下面的图像。不要忘记重新编译着色器！

![texcoord_visualization](img/texcoord_visualization.png)

绿色通道表示水平坐标，红色通道表示垂直坐标。黑色和黄色的边角确认纹理坐标在正方形的`0,0`到`1,1`之间被正确插值。由于没有更好的选择，所以使用颜色可视化数据在着色器编程替代 printf 调试。

在 GLSL 中，组合图像采样描述符用采样均匀表示。在片段着色器中添加对它的引用:

```GLSL
layout(binding = 1) uniform sampler2D texSampler;
```

对于其他类型的图像，有等效的 `sample1d` 和 `sampler3D` 类型，确保使用正确的采样器类型。

```GLSL
void main() {
    outColor = texture(texSampler, fragTexCoord);
}
```

纹理采样使用内置的 `texture` 函数。它采用一个 `sampler` 和坐标作为参数。采样器自动处理背景中的滤波和变换。现在，当你运行应用程序时，你应该看到方块上的纹理:

![texture_on_square](img/texture_on_square.png)

尝试通过缩放纹理坐标值大于1来实验寻址模式。例如，当使用 `VK_SAMPLER_ADDRESS_MODE_REPEAT` 时，下面的片段着色器在下面的图像中产生结果:

```GLSL
void main() {
    outColor = texture(texSampler, fragTexCoord * 2.0);
}
```

![texture_on_square_repeated](img/texture_on_square_repeated.png)

你也可以使用顶点颜色来操作纹理颜色:

```GLSL
void main() {
    outColor = vec4(fragColor * texture(texSampler, fragTexCoord).rgb, 1.0);
}
```

这里分离了 RGB 和 alpha 通道，以防止 alpha 通道受到缩放干扰

![texture_on_square_colorized](img/texture_on_square_colorized.png)

您现在知道如何访问着色器的图像！这是一种非常强大的技术，当与也写入 framebuffer 的图像结合时。您可以使用这些图像作为输入，以实现炫酷的效果，如在3D 世界中的后处理和相机显示。

## Depth buffering 深度缓冲

### Introduction 深度缓冲引言

到目前为止，我们使用的几何图形被投影成3 d 图形，但它仍然是完全平坦的。在这一章中，我们要在这个位置加上一个 z 坐标来准备三维网格。我们将使用这个第三个坐标在当前的正方形上放置一个正方形，以查看当几何图形没有按深度排序时出现的问题。

### 3D geometry 3d几何

更改 `Vertex` 结构以使用 3d 向量表示位置，并更新相应的 `VkVertexInputAttributeDescription` 格式:

```cpp
struct Vertex {
    glm::vec3 pos;
    glm::vec3 color;
    glm::vec2 texCoord;

    ...

    static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions() {
        std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions{};

        attributeDescriptions[0].binding = 0;
        attributeDescriptions[0].location = 0;
        attributeDescriptions[0].format = VK_FORMAT_R32G32B32_SFLOAT;
        attributeDescriptions[0].offset = offsetof(Vertex, pos);

        ...
    }
};
```

接下来，更新顶点着色器以接受和转换3D 坐标作为输入。不要忘记重新编译它！

```GLSL
layout(location = 0) in vec3 inPosition;

...

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 1.0);
    fragColor = inColor;
    fragTexCoord = inTexCoord;
}
```

最后，将`vertices`容器更新为包含 z 坐标:

```cpp
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}}
};
```

如果您现在运行应用程序，那么您应该会看到与以前完全相同的结果。现在是时候添加一些额外的几何图形，使场景更加有趣，并演示我们将在本章中处理的问题。复制顶点来定义一个正方形在当前顶点下面的位置，如下所示:

![extra_square](img/extra_square.svg)

使用 `-0.5f` 的 z 坐标，并为额外的方块添加适当的索引:

```cpp
const std::vector<Vertex> vertices = {
    {{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}},

    {{-0.5f, -0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {0.0f, 0.0f}},
    {{0.5f, -0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {1.0f, 0.0f}},
    {{0.5f, 0.5f, -0.5f}, {0.0f, 0.0f, 1.0f}, {1.0f, 1.0f}},
    {{-0.5f, 0.5f, -0.5f}, {1.0f, 1.0f, 1.0f}, {0.0f, 1.0f}}
};

const std::vector<uint16_t> indices = {
    0, 1, 2, 2, 3, 0,
    4, 5, 6, 6, 7, 4
};
```

现在运行你的程序，你会看到一个类似埃舍尔的例子:

![depth_issues](img/depth_issues.png)

问题是，下方块的片段被绘制在上方块的片段之上，仅仅因为它在索引数组的后面。有两种方法可以解决这个问题:

- 按深度从后到前排序调用所有的绘图操作
- 使用深度缓冲器进行深度测试

第一种方法通常用于绘制透明对象，因为与顺序无关的透明性是一个难以解决的挑战。然而，根据深度排序片段的问题通常使用深度缓冲器来解决。深度缓冲器是一个附加的附件，它存储每个位置的深度，就像颜色附件存储每个位置的颜色一样。每当光栅化程序产生一个片段时，深度测试将检查新片段是否比前一个片段更接近。如果不是，那么新的片段就会被丢弃。通过深度测试的片段将自己的深度写入深度缓冲区。可以从片段着色器中操纵这个值，就像操纵颜色输出一样。

```cpp
#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
```

由GLM生成的透视投影矩阵默认使用OpenGL深度范围-1.0到1.0。我们需要使用`GLM_FORCE_DEPTH_ZERO_TO_ONE`定义将其配置为使用0.0到1.0的Vulkan范围。