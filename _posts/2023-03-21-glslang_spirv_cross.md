---
layout: post
title: glslangValidator和spirv-cross使用说明
category: Graphics-API
---

**原创内容，转载请注明**

---

## glslangValidator

### 读取glsl450生成spv

{% highlight sh %}
glslangValidator -V <input filename> -o <output filename>
 
# 例如：
glslangValidator -V vertexShaders.vert -o  vertexShaders.spv

{% endhighlight %}

### 修改生成spv的入口函数名

{% highlight sh %}
glslangValidator -V <input filename> --source-entrypoint main -e <name> -o <output filename>

# 例如：
glslangValidator -V vertexShaders.vert --source-entrypoint main -e xyzmain -o vertexShaders.spv
{% endhighlight %}

## spirv-cross

### 转换各平台shader（各版本均支持）

{% highlight sh %}
- GLSL: ./spirv-cross --version 330 --no-es test.spv
- ESSL: ./spirv-cross --version 100 --es test.spv
- MSL:  ./spirv-cross --msl test.spv
- HLSL: ./spirv-cross --hlsl test.spv
{% endhighlight %}

### 查看反射信息

{% highlight sh %}
spirv-cross <input filename> --reflect

# 例如：
spirv-cross vertexShaders.spv --reflect
{% endhighlight %}