发布版本 1/13/2025

**WIP**

- uniform
- material shader
- permutation
- vertex factory

# 引言

欢迎参加腾讯游戏引擎图形学课题培养计划！我们将从熟悉Unreal Engine开始，分别实现Compute Shader、Vertex Shader和Pixel Shader，以及最后结合Unreal渲染管线实现可微渲染。

希望通过这次课题，可以让你更加了解虚幻引擎和DX图形API的渲染机制。如果你对于此教程/课题有意见建议，也欢迎随时联系我们。

# 这不是个挑战，而是作业

在这篇教程的源代码部分（仅限于列出来的文件），需要修改xx处，才会解决xx处编译和链接错误（包括Visual Studio报错和shader编译错误），xx处运行报错和错误结果（从Unreal Editor启动开始）。

需要你把这些错误找到并改正，并且以注释的方式备注。不能添加注释的文件（如json格式）不用添加。举例：

```c++
class ExampleClass{
	//int* pField; 这里的指针没有初始化，导致后续判空条件失效
	int* pField = nullptr;
}
...

if(ExampleClassInstance->pField==nullptr){
	...
}
```

将你的独立repo上传至Github公有仓库，将链接发至导师。

# Chapter 1. 引擎编译和工程创建

# 1.1 从源代码编译UE引擎

此部分参考Epic官方文档。

https://dev.epicgames.com/documentation/en-us/unreal-engine/downloading-unreal-engine-source-code

https://dev.epicgames.com/documentation/en-us/unreal-engine/building-unreal-engine-from-source

需要注意的是，编译UE引擎可能提示msvc版本不匹配。所以推荐在Visual Studio Installer中额外安装MSVC v14.36版本。

![alt text](image-5.png)

# 1.2 创建项目和Module

编译结束后，启动UE引擎，创建C++空项目。不要勾选Starter Content和Raytracing。

![alt text](image.png)

项目根目录按图创建目录和文件

![alt text](image-11.png)

编辑器打开MyProject.uproject，在"Modules"里面加入

```json
{
	"Name": "ExampleComputeShader",
	"Type": "Runtime",
	"LoadingPhase": "Default"
}
```

编写代码：

```C#
// ExampleComputeShader.Build.cs
using UnrealBuildTool;

public class ExampleComputeShader : ModuleRules
{
	public ExampleComputeShader(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
	
		PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore" });

		// PrivateDependencyModuleNames.AddRange(new string[] { "" });

		// Uncomment if you are using Slate UI
		// PrivateDependencyModuleNames.AddRange(new string[] { "Slate", "SlateCore" });
		
		// Uncomment if you are using online features
		// PrivateDependencyModuleNames.Add("OnlineSubsystem");

		// To include OnlineSubsystemSteam, add it to the plugins section in your uproject file with the Enabled attribute set to true
	}
}
```

```C++
// ExampleComputeShaderModule.h
#include "Modules/ModuleManager.h"

class FExampleComputeShaderModule : public IModuleInterface {
    virtual void StartupModule() override;
    // virtual void ShutdownModule() override;
};
```

```C++
// ExampleComputeShaderModule.cpp
#include "ExampleComputeShaderModule.h"

IMPLEMENT_MODULE(FExampleComputeShaderModule, ExampleComputeShader)


void FExampleComputeShaderModule::StartupModule()
{
	// 在模块初始化后将项目目录的Shaders文件夹Mount到/MyShaders
	AddShaderSourceDirectoryMapping(TEXT("/MyShader"), FPaths::ProjectDir() / TEXT("Shaders"));
}
```

> ## **重要概念：Module**
> 
> 与传统的C++项目组织方式不同，Unreal在类/全局函数上层用Module封装。每一个Module必须有[ModuleName].Build.cs文件，定义了该Module的Public和Private引用以及编译的配置。每一个Module通常有Public和Private的文件夹，引用它的Module只能访问到Public文件夹中的头文件。
>
> 每一个Module在项目编译时都会编译出dll和lib文件，供动态/静态链接。我们可以显式地使用MODULENAME_API宏来暴露符号，使链接器可以链接到你引用该Module中的符号。
> 
> 更多的内容参考 https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules

# 1.3 编译&启动项目

Explorer打开项目根目录，右键MyProject.uproject文件，选择Generate Visual Studio Project files，打开MyProject.sln，Solution Explorer找到Games/MyProject，右键Build。

![alt text](image-3.png)

Build结束后，点击Local Windows Debugger，启动项目。

![alt text](image-6.png)

>
> ## 重要概念：UnrealBuildTool
> 
> UnrealBuildTool是作用于Unreal引擎自身编译和项目编译的工具链。该工具由C#代码编写，并且可以在Visual Studio中进行调试。该工具读取Build.cs和Target.cs，并生成所有的编译参数。每次对项目目录的文件进行创建和删除，或者更换工具链，需要我们右键uproject文件，Generate Visual Studio Project files更新sln文件。我们可以查看Saved/Logs查看UBT的日志。在Intermediate/Build/Win64/x64/UnrealEditor目录下找到rsp文件查看编译参数。
>
> 参考：https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-build-tool-in-unreal-engine
> 
> 参考：https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-response-files?view=vs-2022

# Chapter 2. 使用UE实现Global Compute Shader

UE提供了面向HLSL的各种接口。其中涉及到底层调用（RHI、RenderCore）以及渲染引擎（Basepass，Lumen）等。我们在这部分先主要了解RHI、RenderCore，并且使用这两个模块实现一个Compute Shader。这个Compute Shader将接受1个float输入，2个float参数，输出1个结果。输入输出我们将使用GPU Buffer。主要的步骤如下：

- 编写usf文件（GPU/HLSL端）
- 使用UE自带的函数和宏创建该shader（CPU/C++端）
- 定义shader的输入输出参数（CPU/C++端）
- 创建shader所需的buffer资源
- 调用shader
- 创建blueprint函数接口，方便调用和调试

# 2.1 编写usf文件

usf全称为Unreal Shader File, 使用hlsl格式编写。为了便于调试usf，我们在VS的Solution Explorer中找到Engine/UE5项目，找到Config/ConsoleCariables.ini，在[Startup]下面添加

```ini
[Startup]
r.ShaderDevelopmentMode=1
r.DumpShaderDebugInfo=1
```

这样我们可以在shader编译失败马上重试，并且在项目Saved/ShaderDebugInfo文件夹内找到Shader的Debug信息。

打开项目根目录，新建Shaders/ExampleComputeShader.usf文件，输入HLSL代码：

```c
// ExampleComputeShader.usf
#include "/Engine/Public/Platform.ush"

float Scale;
float Translate;

RWStructuredBuffer<float> InputBuffer;
RWStructuredBuffer<float> OutputBuffer;

[numthreads(64, 64, 64)]
void FunctionMultiply(
	uint3 DispatchThreadId : SV_DispatchThreadID,
	uint GroupIndex : SV_GroupIndex)
{
    OutputBuffer[DispatchThreadId.x] = InputBuffer[DispatchThreadId.x] * Scale + Translate;
}
```

# 2.2 使用UE自带的函数和宏创建global shader

修改ExampleComputeShaderModule.cpp：

```c++
// ExampleComputeShaderModule.cpp

/**
 * FGlobalShader是定义在全局上的，Unreal也有FMaterialShader，目前我们不做讨论。
 * FGlobalShader的初始化将由Unreal定义的宏进行管理，我们可以使用宏方便地初始化Global Shader
 * 需要注意的是，FGlobalShader不能出现成员变量。我们会用其他的方法设置Shader参数
 */
class FExampleComputeShaderCS : public FGlobalShader
{
public:
	DECLARE_SHADER_TYPE(FExampleComputeShaderCS, Global) // 定义一堆函数，我们不用管是什么
};

IMPLEMENT_SHADER_TYPE(, FExampleComputeShaderCS, TEXT("/MyShaders/ExampleComputeShader.usf"), TEXT("FunctionMultiply"), SF_Compute)
// 使用宏自动实现FExampleComputeShaderCS的函数。第一个参数可为空，第二个参数是类名，第三个是usf的path，第四个是usf的入口函数名，第五个是shader类型(SF_Vertex, SF_Pixel, SF_Compute等)。

```

修改ExampleComputeShader.Build.cs，添加RHI和RenderCore的依赖:
```c#
// ExampleComputeShader.Build.cs
using UnrealBuildTool;

public class ExampleComputeShader : ModuleRules
{
	public ExampleComputeShader(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
	
		PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore" });

		PrivateDependencyModuleNames.AddRange(new string[] { "RHI", "RenderCore" }); // Add this

		// Uncomment if you are using Slate UI
		// PrivateDependencyModuleNames.AddRange(new string[] { "Slate", "SlateCore" });
		
		// Uncomment if you are using online features
		// PrivateDependencyModuleNames.Add("OnlineSubsystem");

		// To include OnlineSubsystemSteam, add it to the plugins section in your uproject file with the Enabled attribute set to true
	}
}
```

> ## 链接器错误：LNK2019
>
> 此处如忘记加RHI或RenderCore依赖，项目构建报链接器错误LNK2019
> ```
> [3/4] Link [x64] UnrealEditor-ExampleComputeShader.dll
>   Creating library G:\UnrealProjects\MyProject\Intermediate\Build\Win64\x64\UnrealEditor\Development\ExampleComputeShader\UnrealEditor-ExampleComputeShader.sup.lib and object G:\UnrealProjects\MyProject\Intermediate\Build\Win64\x64\UnrealEditor\Development\ExampleComputeShader\UnrealEditor-ExampleComputeShader.sup.exp
>ExampleComputeShaderModule.cpp.obj : error LNK2019: unresolved external symbol "__declspec(dllimport) class FName __cdecl LegacyShaderPlatformToShaderFormat(enum EShaderPlatform)" referenced in function "void __cdecl DispatchExampleComputeShader_RenderThread(class FRHICommandList &,class FExampleComputeShaderResource *,unsigned int,unsigned int,unsigned int)" 
>ExampleComputeShaderModule.cpp.obj : error LNK2019: unresolved external symbol "__declspec(dllimport) private: void __cdecl FRHIResource::Destroy(void)const " referenced in function "public: __cdecl TRefCountPtr<class FRHIBuffer>::~TRefCountPtr<class FRHIBuffer>(void)"
>ExampleComputeShaderModule.cpp.obj : error LNK2019: unresolved external symbol "__declspec(dllimport) public: __cdecl FRenderResource::FRenderResource(void)" referenced in function "public: __cdecl FExampleComputeShaderResource::FExampleComputeShaderResource(void)"
>ExampleComputeShaderModule.cpp.obj : error LNK2019: unresolved external symbol "__declspec(dllimport) public: virtual __cdecl FRenderResource::~FRenderResource(void)" referenced in function "int `public: __cdecl FExampleComputeShaderResource::FExampleComputeShaderResource(class dtor$0 &&)'::`1'::dtor$0"
> ```
> 此问题的解决方法是全局搜索未找到的符号（比如LegacyShaderPlatformToShaderFormat），在引擎源文件中找到对应的函数定义
> ```c++
> // Engine/Source/Runtime/RHI/Public/RHIString.h
> ...
> RHI_API FName LegacyShaderPlatformToShaderFormat(EShaderPlatform Platform);
> ...
> ```
> 发现使用RHI_API，表示我们需要在自己的项目中添加RHI模块的依赖。

# 2.3 定义Shader的输入输出参数

定义Global Shader的参数有几种写法，我们详细介绍使用SHADER_PARAMETER_STRUCT的写法。对于其他定义方式（如使用FShaderParameter/FShaderResourceParameter）将不作介绍，读者可以自行研究[Reference](https://www.cnblogs.com/wakuwaku/articles/16706479.html)。

修改FExampleComputeShaderCS的定义：

```c++
// 添加头文件
#include "ShaderParameterStruct.h"

// 使用ShaderParameterStruct.h内的宏定义Shader参数结构体。
// **注意** 这里每个变量名字必须与usf文件中Shader参数名一致。
BEGIN_SHADER_PARAMETER_STRUCT(FExampleComputeShaderParameters,)
SHADER_PARAMETER(float, Scale)     // 基本类型的shader参数
SHADER_PARAMETER(float, Translate) // 基本类型的shader参数
SHADER_PARAMETER_UAV(RWStructuredBuffer<float>, InputBuffer)  // UAV Buffer类型的shader参数，需要我们后续手动管理
SHADER_PARAMETER_UAV(RWStructuredBuffer<float>, OutputBuffer) // UAV Buffer类型的shader参数，需要我们后续手动管理
END_SHADER_PARAMETER_STRUCT()


/**
 * FGlobalShader是定义在全局上的，Unreal也有FMaterialShader，目前我们不做讨论。
 * FGlobalShader的初始化将由Unreal定义的宏进行管理，我们可以使用宏方便地初始化Global Shader
 * 需要注意的是，FGlobalShader不能出现成员变量。我们会用其他的方法设置Shader参数
 */
class FExampleComputeShaderCS : public FGlobalShader
{
public:
	DECLARE_SHADER_TYPE(FExampleComputeShaderCS, Global) // 定义一堆函数，我们不用管是什么
	SHADER_USE_PARAMETER_STRUCT(FExampleComputeShaderCS, FGlobalShader)  // 定义该Shader使用SHADER_PARAMETER_STRUCT来定义Shader参数

	using FParameters = FExampleComputeShaderParameters; // SHADER_USE_PARAMETER_STRUCT需要我们定义FParameters，我们在类外定义的，所以只需要using就可以。
	//当然也可以直接在FGlobalShader内部定义BEGIN_SHADER_PARAMETER_STRUCT(FParameters,) 两种方法使用一个即可
};

IMPLEMENT_SHADER_TYPE(, FExampleComputeShaderCS, TEXT("/MyShaders/ExampleComputeShader.usf"), TEXT("FunctionMultiply"), SF_Compute)
// 使用宏自动实现FExampleComputeShaderCS的函数。第一个参数可为空，第二个参数是类名，第三个是usf的path，第四个是usf的入口函数名，第五个是shader类型(SF_Vertex, SF_Pixel, SF_Compute等)。
```

>
> ## 重要概念：**SRV和UAV**
> 
> SRV和UAV都是在DX11中的buffer类型。区别是，UAV可以在HLSL中写入，而SRV不行。SRV可以在任意的shader绑定，但UAV只能绑定pixel shader和compute shader。后续章节将讲解如何在图形渲染管线中使用UAV。
>
> 参考：https://learn.microsoft.com/en-us/windows/uwp/graphics-concepts/shader-resource-view--srv-

# 2.4 创建Shader Buffer所需资源

我们将创建一个FRenderResource的子类。

```c++
// ExampleComputeShaderModule.h

class EXAMPLECOMPUTESHADER_API FExampleComputeShaderResource : public FRenderResource // 使用EXAMPLECOMPUTESHADER_API因为我们要在别的模块访问到这个类
{
	static FExampleComputeShaderResource* GInstance; // 单例模式
public:
	FRWBufferStructured InputBuffer;  // RWStructuredBuffer<float> InputBuffer;
	FRWBufferStructured OutputBuffer; // RWStructuredBuffer<float> OutputBuffer;

	// 初始化所有buffer，override基类
	// 此函数由FRenderResource::InitResource(FRHICommandList&)调用
	virtual void InitRHI(FRHICommandListBase& RHICmdList) override;

	
	// 此函数由FRenderResource::ReleaseResource(FRHICommandList&)调用
	virtual void ReleaseRHI() override; // 释放所有buffer

	static FExampleComputeShaderResource* Get(); // 单例模式
private:
	FExampleComputeShaderResource() {} // 构造函数私有
};
```

```c++
// ExampleComputeShaderModule.cpp

// 初始化单例指针
FExampleComputeShaderResource* FExampleComputeShaderResource::GInstance = nullptr;

FExampleComputeShaderResource* FExampleComputeShaderResource::Get()
{
	if(GInstance==nullptr)
	{
		GInstance = new FExampleComputeShaderResource();
		// 创建执行在RenderThread的任务。第一个括号写全局唯一的标识符，一般为这个Task起个名字。第二个写Lambda表达式，用FRHICommandList& RHICmdList作为函数参数。
		ENQUEUE_RENDER_COMMAND(FInitExampleComputeShaderResource)([](FRHICommandList& RHICmdList)
		{
			GInstance->InitResource(RHICmdList);
		});
	}
	return GInstance;
}

// Buffer初始化函数
void FExampleComputeShaderResource::InitRHI(FRHICommandListBase& RHICmdList)
{
	InputBuffer.Initialize(RHICmdList, TEXT("InputBuffer"), sizeof(float), 1);
	OutputBuffer.Initialize(RHICmdList, TEXT("OutputBuffer"), sizeof(float), 1);
}

// Buffer释放函数
void FExampleComputeShaderResource::ReleaseRHI()
{
	InputBuffer.Release();
	OutputBuffer.Release();
}

```

> ## 重要概念：GameThread和RenderThread
> 
> 为了提高效率，Unreal Engine将游戏逻辑和渲染流程分开执行，使用专门的RenderThread负责处理所有渲染相关的工作。所有的游戏逻辑（加载模块、蓝图函数）等都在GameThread执行，而渲染相关的流程在RenderThread执行。判断一个函数是否在RenderThread执行，我们看它：a.是否接受一个FRHICommandList&类型的参数；b.是否有函数IsInRenderingThread()的检查。在GameThread执行Rendering指令会缺失相关渲染环境，导致一系列错误，反之亦然。
>
> GameThread可以向RenderThread提交渲染任务，使用ENQUEUE_RENDER_COMMAND。具体调用方式见上方代码。使用FlushRenderingCommands()函数flush所有RenderThread任务。
> 
> 参考：https://www.cnblogs.com/kekec/p/15464958.html

# 2.5 调用Shader

终于我们到了最后一步。现在我们需要写一个函数整合刚才所有的资源，调用我们的Compute Shader。这里的写法相对比较固定。
```c++
// ExampleComputeShaderModule.cpp
void DispatchExampleComputeShader_RenderThread(FRHICommandList& RHICmdList, FExampleComputeShaderResource* Resource, float Scale, float Translate, uint32 ThreadGroupX, uint32 ThreadGroupY, uint32 ThreadGroupZ)
{
	TShaderMapRef<FExampleComputeShaderCS> Shader(GetGlobalShaderMap(GMaxRHIFeatureLevel)); // 声明ShaderMap
	SetComputePipelineState(RHICmdList, Shader.GetComputeShader()); // 设置该RHICmdList的Pipeline为ComputeShader
	{
		typename FExampleComputeShaderCS::FParameters Parameters{}; // 创建Shader参数

		// 设置基本类型
		Parameters.Scale = Scale;
		Parameters.Translate = Translate;

		// 设置buffer类型
		Parameters.InputBuffer = Resource->InputBuffer.UAV;
		Parameters.OutputBuffer = Resource->OutputBuffer.UAV;

		// 传入参数
		SetShaderParameters(RHICmdList, Shader, Shader.GetComputeShader(), Parameters);
	}

	// 调用Compute shader
	DispatchComputeShader(RHICmdList, Shader.GetShader(), ThreadGroupX, ThreadGroupY, ThreadGroupZ);

	// 取消绑定buffer参数
	//UnsetShaderSRVs(RHICmdList, Shader, Shader.GetComputeShader());
	UnsetShaderUAVs(RHICmdList, Shader, Shader.GetComputeShader());
}

void DispatchExampleComputeShader_GameThread(float InputVal, float Scale, float Translate, FExampleComputeShaderResource* Resource)
{
	// 加入RenderThread任务
	ENQUEUE_RENDER_COMMAND(FDispatchExampleComputeShader)([Resource, InputVal, Scale, Translate](FRHICommandListImmediate& RHICmdList)
	{
		// LockBuffer并写入数据
		float* InputGPUBuffer = static_cast<float*>(RHICmdList.LockBuffer(Resource->InputBuffer.Buffer, 0, sizeof(float), RLM_WriteOnly));
		*InputGPUBuffer = InputVal; // 可以使用FMemory::Memcpy
		// UnlockBuffer
		RHICmdList.UnlockBuffer(Resource->InputBuffer.Buffer);
		// 调用RenderThread版本的函数
		DispatchExampleComputeShader_RenderThread(RHICmdList, Resource, Scale, Translate, 1, 1, 1);
	});
}

float GetGPUReadback(FExampleComputeShaderResource* Resource, float& OutputVal)
{
	float* pOutputVal = &OutputVal;
	// Flush所有RenderingCommands, 确保我们的shader已经执行了
	FlushRenderingCommands();
	ENQUEUE_RENDER_COMMAND(FReadbackOutputBuffer)([Resource, &pOutputVal](FRHICommandListImmediate& RHICmdList)
	{
		// LockBuffer并读取数据
		float* OutputGPUBuffer = static_cast<float*>(RHICmdList.LockBuffer(Resource->OutputBuffer.Buffer, 0, sizeof(float), RLM_ReadOnly));
		*pOutputVal = *OutputGPUBuffer;
	});
	// FlushRenderingCommands, 确保上面的RenderCommand被执行了
	FlushRenderingCommands();
	return OutputVal;

	// 下面是使用FRHIGPUBufferReadback的调用方式, 效果是相同的
	//FRHIGPUBufferReadback ReadbackBuffer(TEXT("ExampleComputeShaderReadback"));
	//FlushRenderingCommands();
	//ENQUEUE_RENDER_COMMAND(FEnqueueGPUReadback)([&ReadbackBuffer, Resource](FRHICommandListImmediate& RHICmdList)
	//{
	//	ReadbackBuffer.EnqueueCopy(RHICmdList, Resource->OutputBuffer.Buffer, sizeof(int32));
	//});
	//FlushRenderingCommands();
	//float OutputVal;
	//ENQUEUE_RENDER_COMMAND(FEnqueueGPUReadbackLock)([&ReadbackBuffer, Resource, &OutputVal](FRHICommandListImmediate& RHICmdList)
	//{
	//	float* pOutputVal = static_cast<float*>(ReadbackBuffer.Lock(sizeof(float)));
	//	checkf(pOutputVal!=nullptr, TEXT("ReadbackBuffer failed to lock"));
	//	OutputVal = *pOutputVal;
	//	ReadbackBuffer.Unlock();
	//});
	//FlushRenderingCommands();
	//return OutputVal;
}

```

# 2.6 创建Blueprint function并调用

这一步我们将创建一个Blueprint function，以在游戏内调用我们的compute shader。

创建ShaderFunctionLibrary模块：

```c#
// ShaderFunctionLibrary.Build.cs
using UnrealBuildTool;

public class ShaderFunctionLibrary : ModuleRules
{
	public ShaderFunctionLibrary(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
	
		PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore" });

		PrivateDependencyModuleNames.AddRange(new string[] { "ExampleComputeShader" });

		// Uncomment if you are using Slate UI
		// PrivateDependencyModuleNames.AddRange(new string[] { "Slate", "SlateCore" });
		
		// Uncomment if you are using online features
		// PrivateDependencyModuleNames.Add("OnlineSubsystem");

		// To include OnlineSubsystemSteam, add it to the plugins section in your uproject file with the Enabled attribute set to true
	}
}
```

```c++
// ShaderFunctionLibraryModule.h
#pragma once

#include "CoreMinimal.h"
#include "Kismet/BlueprintFunctionLibrary.h"
#include "ExampleComputeShaderModule.h"

#include "ShaderFunctionLibraryModule.generated.h"

class FShaderFunctionLibraryModule : public IModuleInterface {
    virtual void StartupModule() override;
    // virtual void ShutdownModule() override;
};

UCLASS(meta=(ScriptName="ShaderFunctionLibrary"), MinimalAPI)
class UShaderFunctionLibrary : public UBlueprintFunctionLibrary
{
	GENERATED_BODY()
public:
	UFUNCTION(BlueprintCallable, meta=(DisplayName="Execute ExampleComputeShader"), Category="My Shader Functions")
	static SHADERFUNCTIONLIBRARY_API float ExecuteExampleComputeShader(float InputVal, float Scale, float Translate) {
		float OutputVal;
		DispatchExampleComputeShader_GameThread(InputVal, Scale, Translate, FExampleComputeShaderResource::Get());
		return GetGPUReadback(FExampleComputeShaderResource::Get(), OutputVal);
	}
};
```

```c++
// ShaderFunctionLibraryModule.cpp
#include "ShaderFunctionLibraryModule.h"

void FShaderFunctionLibraryModule::StartupModule()
{ }
```

编译，运行项目，新建Blueprint Actor，打开Blueprint Editor，在**Event Beginplay**后连接我们的`Execute ExampleComputeShader`，同时输出结果。

![alt text](image-8.png)

将这个Actor放置到场景中，**保存所有项目**，以避免运行时引擎报错导致更改丢失。启动游戏，发现打印结果。

![alt text](image-10.png)

恭喜！你已经成功创建了一个Compute Shader，并且成功运行。

# Chapter 2 - 总结

创建Compute Shader的步骤：

- 编写usf文件（GPU/HLSL端）
- 创建shader和所需资源
- 创建blueprint函数接口，方便调用和调试

重要概念：

- SRV和UAV
- GameThread和RenderThread

# Chapter 2 - 作业

0. 运行示例代码，修改所有编译错误和运行错误，确保一切功能正常 (有的时候测试一次不够，需要多次测试)
1. 将传入一个变量修改为传入一个数组，由蓝图创建 (MakeArray) 并传入到函数中 (TArray)，同时输出数组 (提示：蓝图中使用MakeArray创建数组，在蓝图函数接口中使用TArray<>&接受，同时需要修改shader buffer相关代码)

# Chapter 3. 使用UE实现Global VS/PS

# 3.1 编写usf文件

```c
// ExampleGrapihcsShader.usf
#include "/Engine/Public/Platform.ush"
// HLSL代码的文档见https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-reference


// 顶点着色器的输入
struct VertexAttributes
{
	float v_position : ATTRIBUTE0; // 顶点位置。
	//通常情况下我们输入模型坐标系的顶点，经由view matrix和perspective projection matrix变换得到NDC。但为了演示方便起见我们直接输入NDC。
	float v_color : ATTRIBUTE1; // 顶点颜色，取值范围为0.0~1.0
};

// 顶点着色器的输出，同时也是pixel shader的interpolated输入
struct Varyings
{
	float4 p_position : SV_POSITION; // 顶点着色器输出，需要在NDC(Normalized device coordinate)坐标空间下; 片元着色器(pixel shader)的坐标位置输入
	float4 p_color : COLOR0; // 顶点着色器输出; 片元颜色输入
};

void MainVS(in VertexAttributes vertex, out Varyings v)
{
	v.p_position = vertex.v_position; // 此处由于输入的顶点坐标即是NDC，所以不作变换
	v.p_color = vertex.v_color;
}

void MainPS(in Varyings v, out float4 f_color : SV_Target0)
{
	f_color = v.p_color; // 每个片元的颜色输出到render target
}
```

> ## 重要概念：HLSL语义(Semantics)
> 在定义每个函数的输入输出参数时（包括输入输出参数是结构体），我们需要定义每个参数的语义。特定的语义只能在特定的shader中使用，而且特定的shader必须定义特定的语义。比如vertex shader必须有out SV_POSITION.
> 
> 参考：https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics
>
> 参考：https://stackoverflow.com/questions/22064165/why-does-hlsl-have-semantics


# 3.2 创建Global Shader

创建ExampleGraphicsShader模块

```c#
// ExampleGrpahicsShader.Build.cs
using UnrealBuildTool;

public class ExampleGraphicsShader : ModuleRules
{
	public ExampleGraphicsShader(ReadOnlyTargetRules Target) : base(Target)
	{
		PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;
	
		PublicDependencyModuleNames.AddRange(new string[] { "Core", "CoreUObject", "Engine", "InputCore" });

		PrivateDependencyModuleNames.AddRange(new string[] { "RHI", "RenderCore" });

		// Uncomment if you are using Slate UI
		// PrivateDependencyModuleNames.AddRange(new string[] { "Slate", "SlateCore" });
		
		// Uncomment if you are using online features
		// PrivateDependencyModuleNames.Add("OnlineSubsystem");

		// To include OnlineSubsystemSteam, add it to the plugins section in your uproject file with the Enabled attribute set to true
	}
}
```

定义Shader和ShaderResource

```c++
// ExampleGraphicsShaderModule.h
#pragma once

#include "Modules/ModuleManager.h"

class FExampleGraphicsShaderModule : public IModuleInterface {
    virtual void StartupModule() override;
    // virtual void ShutdownModule() override;
};


// 这个struct对应hlsl里面的VertexAttributes
EXAMPLEGRAPHICSSHADER_API struct VertexAttributes
{
	FVector4f Position; // Normally we use homogeneous coordinate so we declared Vec4 but here for demonstration we only use first 2 components to store NDC
	FVector4f Color; // RGBA
};


// 存VS和PS对应的渲染资源
class EXAMPLEGRAPHICSSHADER_API FExampleGraphicsShaderResource : public FRenderResource
{
	static FExampleGraphicsShaderResource* GInstance; // Singleton instance
public:
	virtual void InitRHI(FRHICommandListBase& RHICmdList) override;
	virtual void ReleaseRHI() override;
	
	FVertexDeclarationRHIRef VertexDeclarationRHI; // 定义vertex的数据是如何存储在buffer的。见InitRHI()
	FReadBuffer VertexBuffer; // GPU只读不写，所以定义为ReadBuffer
	static FExampleGraphicsShaderResource* Get(); // Singleton instance
};

EXAMPLEGRAPHICSSHADER_API void RenderExampleGraphicsShader_GameThread(UTextureRenderTarget2D* TextureRenderTarget2D, FExampleGraphicsShaderResource* Resource);
```

```c++
// ExampleGraphicsShaderModule.cpp
#include "ExampleGraphicsShaderModule.h"

#include "ClearQuad.h"
#include "SelectionSet.h"
#include "Engine/TextureRenderTarget2D.h"

IMPLEMENT_MODULE(FExampleGraphicsShaderModule, ExampleGraphicsShader)

void FExampleGraphicsShaderModule::StartupModule()
{
	// 在模块初始化后将项目目录的Shaders文件夹Mount到/MyGraphicsShader
	AddShaderSourceDirectoryMapping(TEXT("/MyGraphicsShaders"), FPaths::ProjectDir() / TEXT("Shaders"));
}

class FExampleGraphcisShaderVS: public FGlobalShader
{
public:
	DECLARE_EXPORTED_GLOBAL_SHADER(FExampleGraphcisShaderVS, EXAMPLEGRAPHICSSHADER_API);
	static bool ShouldCache(EShaderPlatform Platform)
	{
		return true;
	}
};

class FExampleGraphcisShaderPS: public FGlobalShader
{
public:
	DECLARE_EXPORTED_GLOBAL_SHADER(FExampleGraphcisShaderPS, EXAMPLEGRAPHICSSHADER_API);
	static bool ShouldCache(EShaderPlatform Platform)
	{
		return true;
	}
};


IMPLEMENT_SHADER_TYPE(, FExampleGraphcisShaderVS, TEXT("/MyGraphicsShaders/ExampleGraphicsShader.usf"), TEXT("MainVS"), SF_Vertex);
IMPLEMENT_SHADER_TYPE(, FExampleGraphcisShaderPS, TEXT("/MyGraphicsShaders/ExampleGraphicsShader.usf"), TEXT("MainPS"), SF_Pixel);



FExampleGraphicsShaderResource*  FExampleGraphicsShaderResource::GInstance = nullptr;
FExampleGraphicsShaderResource* FExampleGraphicsShaderResource::Get()
{
	if(GInstance==nullptr)
	{
		GInstance = new FExampleGraphicsShaderResource();
		ENQUEUE_RENDER_COMMAND(FInitExampleGraphicsShaderResource)([](FRHICommandList& RHICmdList)
		{
			GInstance->InitResource(RHICmdList);
		});
	}
	return GInstance;
}

void FExampleGraphicsShaderResource::InitRHI(FRHICommandListBase& RHICmdList)
{
	// 我们先hard code顶点数据
	TArray<VertexAttributes> Vertices;
	Vertices.Add({FVector4f(-1, 1, 1, 1), FVector4f(1,0,0,1)});
	Vertices.Add({FVector4f(1, 1,1, 1), FVector4f(0,0,1,1)});
	Vertices.Add({FVector4f(-1,-1,1, 1), FVector4f(1,1,0,1)});

	// 初始化buffer并拷贝
	uint32 NumBytes = sizeof(VertexAttributes)* Vertices.Num();
	FRHIResourceCreateInfo CreateInfo(TEXT("VertexBuffer"));
	VertexBuffer.Buffer = RHICmdList.CreateVertexBuffer(NumBytes, BUF_Static, CreateInfo);
	VertexAttributes* GPUBufferPtr = static_cast<VertexAttributes*>(RHICmdList.LockBuffer(VertexBuffer.Buffer, 0, sizeof(VertexAttributes) * Vertices.Num(), RLM_ReadOnly));
	FMemory::Memcpy(GPUBufferPtr, Vertices.GetData(), NumBytes);
	RHICmdList.UnlockBuffer(VertexBuffer.Buffer);
	// 到此为止，我们的buffer存储了3个vertices，每个vertex有4个float作为position和4个float作为color。所以一个vertex是4*sizeof(float) + 4*sizeof(float)==32字节。当然3个vertices总共96个字节。

	// 下面我们定义vertex buffer的数据排布。

	// 定义VertexDeclaration
	uint16 Stride = sizeof(VertexAttributes);
	FVertexDeclarationElementList Elements;
	Elements.Add(FVertexElement(0, STRUCT_OFFSET(VertexAttributes, Position),VET_Float4, 0, Stride));
	Elements.Add(FVertexElement(0, STRUCT_OFFSET(VertexAttributes, Color), VET_Float4, 1, Stride));
	VertexDeclarationRHI = PipelineStateCache::GetOrCreateVertexDeclaration(Elements);
	// 根据vertex的定义，前四个float是position，并且我们在hlsl将其定义为ATTRIBUTE0。所以它的InOffset应该是0（因为是结构体的第一个成员）；stride应该是sizeof(VertexAttributes)因为当前指针到下一个vertex指针的偏移量是sizeof(VertexAttributes)
	// Color同理，但它的InOffset是Position的大小，即4*sizeof(float)
	// 如果读者熟悉opengl API，FVertexElement与glVertexAttribPointer相似
	
}

void FExampleGraphicsShaderResource::ReleaseRHI()
{
	if(VertexBuffer.Buffer)
	{
		VertexBuffer.Release();
	}
	VertexDeclarationRHI.SafeRelease();
}

void RenderExampleGraphicsShader_RenderThread(FRHICommandList& RHICmdList, FExampleGraphicsShaderResource* Resource, FRHITexture* RenderTarget)
{
	// 调用shader进行渲染
	// 与compute shader不同，graphics shader需要我们定义：1）一个或多个Render Target；2）如何处理Rasterizer、Blend、Depth和Stencil的行为
	// 所以需要给到的初始化参数比compute shader多
	// 一个很好的例子是DrawClearQuad函数，里面详细写了从RHICmdList.BeginRenderPass到RHICmdList.EndRenderPass的所有步骤。如果你的shader无法工作，请使用DrawClearQuad函数进行测试。
	FRHIRenderPassInfo RPInfo(RenderTarget, ERenderTargetActions::Clear_Store);
	RHICmdList.BeginRenderPass(RPInfo, TEXT("ExampleGraphicsShaderRenderPass"));
	// 如果需要测试，可以在这里调用DrawClearQuad(RHICmdList, true, FLinearColor(FVector4(1, 0, 1, 1)), true, 1.0, true, 0);
	// 同时注释其他代码。
	auto ShaderMap = GetGlobalShaderMap(GMaxRHIFeatureLevel);

	TShaderMapRef<FExampleGraphcisShaderVS> VertexShader(ShaderMap);
	TShaderMapRef<FExampleGraphcisShaderPS> PixelShader(ShaderMap);

	FGraphicsPipelineStateInitializer GraphicsPSOInit;
	RHICmdList.ApplyCachedRenderTargets(GraphicsPSOInit);
	GraphicsPSOInit.RasterizerState = TStaticRasterizerState<>::GetRHI();
	GraphicsPSOInit.BlendState = TStaticBlendState<>::GetRHI();
	GraphicsPSOInit.DepthStencilState = TStaticDepthStencilState<true, CF_Always>::GetRHI();
	GraphicsPSOInit.PrimitiveType = PT_TriangleStrip;
	GraphicsPSOInit.BoundShaderState.VertexShaderRHI = VertexShader.GetVertexShader();
	GraphicsPSOInit.BoundShaderState.PixelShaderRHI = PixelShader.GetPixelShader();

	SetGraphicsPipelineState(RHICmdList, GraphicsPSOInit, 0);
	RHICmdList.SetStreamSource(0, Resource->VertexBuffer.Buffer, 0);
	RHICmdList.DrawPrimitive(0, 1, 1);
	RHICmdList.EndRenderPass();
}

void RenderExampleGraphicsShader_GameThread(UTextureRenderTarget2D* TextureRenderTarget2D, FExampleGraphicsShaderResource* Resource)
{
	ENQUEUE_RENDER_COMMAND(FRenderExampleGraphicsShader)([Resource, TextureRenderTarget2D](FRHICommandList& RHICmdList)
	{
		RenderExampleGraphicsShader_RenderThread(RHICmdList, Resource, TextureRenderTarget2D->GetResource()->GetTexture2DRHI());
	});
}
```

# 3.3 创建蓝图函数和Rendertarget

添加蓝图函数：
```c++
// ShaderFunctionLibraryModule.h

UFUNCTION(BlueprintCallable, meta=(DisplayName="Render ExampleGraphicsShader"),Category="My Shader Functions")
static SHADERFUNCTIONLIBRARY_API void RenderExampleGraphicsShader(UTextureRenderTarget2D* RenderTarget)
{
	FExampleGraphicsShaderResource::Get();
	RenderExampleGraphicsShader_GameThread(RenderTarget, FExampleGraphicsShaderResource::Get());
}
```

编译，启动Editor

Content Browser创建：1. Blueprint Actor 2. Material 3. TextureRenderTarget2D
Blueprint Actor在Event BeginPlay链接Render ExampleGraphicsShader, Variables创建Texture Render Target 2D的Object Reference，并设置成public，将这个变量拖入，与Render ExampleGraphicsShader函数输入链接

![alt text](image-12.png)

打开材质，拖入刚创建的TextureRenderTarget2D，将RGB连至Base Color

![alt text](image-13.png)

场景中放入刚创建的blueprint actor和一个static mesh cube，将blueprint actor的rendertarget设置成刚创建的。将static mesh cube材质设置成刚创建的。

![alt text](image-14.png)

![alt text](image-15.png)

**保存所有项目**，以避免运行时引擎报错导致更改丢失。

点击运行，你的cube应该如下图所示。

![alt text](image-16.png)

# Chapter 3 - 总结

创建Graphics Shader的步骤：

- 编写usf文件（GPU/HLSL端）
- 创建shader和所需资源，并定义Vertex Attributes
- 创建/传入RenderTarget，并调用RHICommand指令进行绘制
- 在编辑器中将材质与RenderTarget绑定，并执行渲染

重要概念：

- HLSL Semantics
- Vertex Attributes

# Chapter 3 - 作业

0. 修改并运行示例代码，处理所有的编译和运行错误，确保最后你的材质显示为彩色三角形。
1. 在Pixel shader创建参数，设置三角形的颜色
2. 在Vertex shader创建参数，设置三角形的平移、旋转和缩放变换（二维）
3. (可选) 修改vertex shader传入的mesh，创建一个cube，设置相机坐标，并使用透视投影变换和shading模型渲染该cube (提示：你需要修改vertex shader参数传入matrix，vertex attribute传入normal，并在vertex shader进行计算光照)
4. (可选) vertex shader传入时间参数，实现一个简单的动画效果
5. (可选) 在蓝图中新建camera，将camera的view matrix和projection matrix传入vertex shader，实现交互效果 (提示：使用FSceneInfo，并且注意matrix的row major或者column major)

# FAQ

## 使用什么Configuration编译引擎？
默认的Development Editor即可

## 参考材料

1. https://dev.epicgames.com/documentation/en-us/unreal-engine/creating-a-new-global-shader-as-a-plugin-in-unreal-engine
2. http://www.valentinkraft.de/compute-shader-in-unreal-tutorial/
