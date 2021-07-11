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
