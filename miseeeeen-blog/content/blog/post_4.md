---
title: "Swiftshader简介"
date: 2020-08-30T13:50:36+08:00
draft: false
---

Swiftshader是Google推出的OpenGL的软件实现.
它的机制和VM差不多, 可以动态地将shader翻译成cpu指令.

下图是它的架构, 主要有三层:
1. Renderer: 负责将Shader中的操作转化为Reactor的调用.
2. Reactor: 一种中间语言，主要就是包装了一下LLVM的调用.
3. LLVM: Swiftshader底层通过LLVM来生成machine code.
![Architecture.png](/images/post_4/16134950-404adada39aaa98a.png)




本文分为三个部分:
- Reactor Layer Introduction
- Renderer Layer Introduction
- Debug Tips

# Reactor Layer Introduction
Reactor是一种中间语言, 可以嵌入在C++中使用.
它的语法参见[Reactor.md]([https://github.com/google/swiftshader/blob/master/docs/Reactor.md](https://github.com/google/swiftshader/blob/master/docs/Reactor.md)
)

Reactor里定义了很多数据类型:
```
// src/Reactor/Reactor.hpp
class Bool;
...
class Short8;
class UShort8;
...
class Int4;
class UInt4;
...
class Float4;
```

可以使用RR_WATCH(variable_name)的形式来print Reactor变量.
前提是在CMakeList.txt中定义ENABLE_RR_PRINT
```
// src/Reactor/Print.hpp

// RR_WATCH() is a helper that prints the name and value of all the supplied
// arguments.
// For example, if you had the Int and bool variables 'foo' and 'bar' that
// you want to print, you can simply write:
//    RR_WATCH(foo, bar)
// When this JIT compiled code is executed, it will print the string
// "foo: 1, bar: true" to stdout.
//
// RR_WATCH() is intended to be used for debugging JIT compiled code, and
// is not intended for production use.
#	define RR_WATCH(...) RR_LOG(RR_WATCH_FMT(__VA_ARGS__), __VA_ARGS__)
```

与LLVM的接口主要在src/Reactor/LLVMReactor.cpp

需要注意的是: Renderer Layer里对它的调用都会生成machine code, 但这些machine code并不会立刻执行, 而是作为data存在的. 只有当swiftshader将程序入口指到这些code的时候，它们才会执行.

# Renderer Layer Introduciton
它处理图像的时候大致会经历三个步骤:
1. VertexRoutine 
2. SetupRoutine 
3. PixelRoutine

```
// src/Render/Render.cpp
	void Renderer::draw(DrawType drawType, unsigned int indexOffset, unsigned int count, bool update)
```
VertexRoutine: 用于处理顶点变换.

SetupRoutine: 根据顶点建立平面方程. 比如说根据三角形的三点来构建一个三角形的平面方程. Pixel Routine会根据平面方程来分别处理不同的pixel.

PixelRoutine: 计算pixel color, 并写入frame buffer. 
PixelRoutine可以细分为三个主要步骤: 
(a) interpolate: 将pixel shader的输入参数根据平面方程做插值
(b) apply shader: 执行pixel shader中的算子
(c) write color: 将数据写入frame buffer

PixelRoutine的处理逻辑可以参见:
```
//  src/Shader/PixelRoutine.cpp
	void PixelRoutine::quad(Pointer<Byte> cBuffer[RENDERTARGETS], Pointer<Byte> &zBuffer, Pointer<Byte> &sBuffer, Int cMask[4], Int &x)
	{

...
        // 这里将1个pixel的x扩展成4个pixel的x
		Float4 xxxx = Float4(Float(x)) + *Pointer<Float4>(primitive + OFFSET(Primitive,xQuad), 16);

...
		If(depthPass || Bool(!earlyDepthTest))
		{
...
            // 这里将1个pixel的y扩展成4个pixel的y
			Float4 yyyy = Float4(Float(y)) + *Pointer<Float4>(primitive + OFFSET(Primitive,yQuad), 16);
...
			for(int interpolant = 0; interpolant < MAX_FRAGMENT_INPUTS; interpolant++)
			{
				for(int component = 0; component < 4; component++)
				{
					if(state.interpolant[interpolant].component & (1 << component))
					{
						if(!state.interpolant[interpolant].centroid)
						{
							v[interpolant][component] = interpolate(xxxx, Dv[interpolant][component], rhw, primitive + OFFSET(Primitive, V[interpolant][component]), (state.interpolant[interpolant].flat & (1 << component)) != 0, state.perspective, false);
						}
						else
						{
							v[interpolant][component] = interpolateCentroid(XXXX, YYYY, rhwCentroid, primitive + OFFSET(Primitive, V[interpolant][component]), (state.interpolant[interpolant].flat & (1 << component)) != 0, state.perspective);
						}
					}
				}
...

			if(colorUsed())
			{
...
				applyShader(cMask);
...
			}

			If(alphaPass)
			{
...
					if(colorUsed())
					{
...
						rasterOperation(f, cBuffer, x, sMask, zMask, cMask);
					}
				}
...
			}
		}
...
	}
```

xQuad和yQuad会在SetupRoutine里进行初始化:
```
// src/Shader/SetupRoutine.cpp
			Float4 xQuad = Float4(0, 1, 0, 1) - Float4(dx);
			Float4 yQuad = Float4(0, 0, 1, 1) - Float4(dy);
```

Swiftshaer使用Vector4f存放4个pixel的rgba. 
Float4里面存放了1个pixel的(r, r, r, r) or (g, g, g, g) or (b, b, b, b) or (a, a, a, a). 明白了这一点, code会容易理解很多.

# Debug Tips:
1. Profile switch of PixelRoutine
```
// src/Main/Config.hpp
#define PERF_PROFILE 0   // Profile various pipeline stages and display the timing in SwiftConfig
```
2. Dump llvm's IR
```
// src/Reactor/LLVMReactor.cpp
		if(false)
		{
			std::error_code error;
			llvm::raw_fd_ostream file(std::string(name) + "-llvm-dump-unopt.txt", error);
			jit->module->print(file, 0);
		}
```

3. Print shader 
```
// src/OpenGL/libGLESv2/Shader.cpp
	if(false)
	{
		static int serial = 1;

		if(false)
		{
			char buffer[256];
			sprintf(buffer, "shader-input-%d-%d.txt", getName(), serial);
			FILE *file = fopen(buffer, "wt");
			fprintf(file, "%s", mSource);
			fclose(file);
		}

		getShader()->print("shader-output-%d-%d.txt", getName(), serial);

		serial++;
	}
```
4. Settings of Swiftshader
Swiftshader提供了很多配置, 包括图像质量/线程数量/优化等级等等, 这些都会极大地影响性能
```
// Swiftshader.ini
...

[Quality]
TextureSampleQuality=2
MipmapQuality=1
PerspectiveCorrection=1
TranscendentalPrecision=2
TransparencyAntialiasing=0

[Processor]
ThreadCount=0
EnableSSE3=1
EnableSSSE3=1
EnableSSE4_1=1

[Optimization]
OptimizationPass1=1
OptimizationPass2=0
OptimizationPass3=0
OptimizationPass4=0
OptimizationPass5=0
OptimizationPass6=0
OptimizationPass7=0
OptimizationPass8=0
OptimizationPass9=0
OptimizationPass10=0
...
```

