---
layout: post
title: Shader数据类型的对齐与尺寸
category: Graphics-API
---

**原创内容，转载请注明**

---

## OpenGL std140 / Vulkan

| Type             | Alignment | Size | ArrayStride |
|:----------------:|:---------:|:----:|:-----------:|
| bool/int/float   | 4         | 4    | 16          |
| bvec2/ivec2/vec2 | 8         | 8    | 16          |
| bvec3/ivec3/vec3 | 16        | 12   | 16          |
| bvec4/ivec4/vec4 | 16        | 16   | 16          |
| mat2             | 16        | 32   | 32          |
| mat3             | 16        | 48   | 48          |
| mat4             | 16        | 64   | 64          |

*   矩阵的对齐与尺寸需要参考的值为MatrixStride，Spirv disamble出来的数据，所有矩阵MatrixStride都为16。

1.  行主序(Row-Major)矩阵的大小为 行数\*MatrixStride
2.  列主序(Column-Major，OpenGL默认列主序)矩阵的大小为 列数\*MatrixStride.

*   数组的对齐跟尺寸需要参考的值为ArrayStride. (不支持二维数组)
    数组的大小为 元素个数\*ArrayStride

```
UniformBuffer {
    float a0;       // offset 0, stride 4, total size 4
    float a1[2];    // offset 16, arrayStride 16, total size 16 + 16 * 3 = 48
    mat2 a4;        // offset 48, matrixStride 16, total size 48 + 16 * 2 = 80
    vec2 a2;        // offset 80, stride 8, total size 80 + 8 = 88
    vec2 a3[2];     // offset 96, arrayStride 16, total size 96 + 16 * 2 = 128
    mat3 a5;        // offset 128, matrixStride 16, total size 128 + 48 = 176
    mat3 a6[2];     // offset 176, arrayStride 48, total size 176 + 48 * 2 = 272
    mat4 a7;        // offset 272, matrixStride 16, toatal size 272 + 64 = 336
    int a8;         // offset 336, stride 4, total size: 340
}
// 总分配大小16字节对齐，共352个字节
```

## Metal

| Type                            | Alignment | Size |
|:------------------------------- |:---------:|:----:|
| bool/char                       | 1         | 1    |
| \[u]int/float                   | 4         | 4    |
| \[u]int2/float2                 | 8         | 8    |
| \[u]int3/float3                 | 16        | 16   |
| packed\_\[u]int3/packed\_float3 | 4         | 12   |
| \[u]int4/float4                 | 16        | 16   |
| float2x2                        | 8         | 16   |
| float3x3                        | 16        | 48   |
| float4x4                        | 16        | 64   |

## spirv-cross类型对应

| OpenGL/Vulkan    | Metal                                              |
|:----------------:|:--------------------------------------------------:|
| bool             | uint                                               |
| int              | int                                                |
| float            | float                                              |
| bvec2/ivec2/vec2 | uint2/int2/float2                                  |
| bvec3/ivec3/vec3 | \[packed\_]uint3/\[packed\_]int3/\[packed\_]float3 |
| bvec4/ivec4/vec4 | uint4/int4/float4                                  |
| mat2         | float2x2                                       |
| mat3             | float3x3                                           |
| mat4             | float4x4                                           |

CPU端结构体内存布局规则与OpenGL/Vulka规则一致
其中有一特例，仅mat2-float2x2的内存布局不一致，需特殊处理：

*   OpenGL/Vulkan的mat2内存布局为：
```
            +--- alignas(16)
         0	| float 0
         4  | float 1
         8	| <padding member> (size=8)
        16	| float 2
        20  | float 3
        24  | <padding member> (size=8)
        32	+--- sizeof(32)
```

*   Metal的float2x2内存布局为：
```
            +--- alignas(8)
         0	| float 0
         4  | float 1
         8	| float 2
        12	| float 3
        16	+--- sizeof(16)
```

```
// 输入的OpenGL
uniform float f1;
uniform mat2 m2;
uniform float f2;

// spirv-cross输出的Metal
struct VertexUniformBlock
{
    float f1;
    char _m1_pad[12];   // 填充12字节，来保证m2为16字节对齐
    float2x2 m2;
    char _m2_pad[16];   // 填充16字节，来保证mat2-float2x2尺寸一致，均为32字节
    float f2;
}
```
