---
layout: post
title:  "Hot Reloading Shaders with DXC"
date:   2024-11-15	 09:00:07 -0700
categories: rendering 
excerpt: <img src="/assets/HotReload/HotReload.jpg">
tags: [real-time, rendering]
---

Hot reloading shaders is the ability to modify the some shaders and reload them at runtime without having to close out your application. When is this useful? Iteration time. Where this comes in handy is when I'm trying to debug problems in my shader code and I'm dealing with a extremely heavy scene like the [Moana Island data set][Disney]. That scene has over 50 million instances and even with a good amount of optimization my renderer still takes 2-3 minutes to load. And that's only for rendering related props! 

![Output Image](/assets/HotReload/MoanaIsland.jpg)

In production game engines, scenes are way more complicated with things like gameplay logic/physics/etc that all need be loaded and it's not uncommon to see editor level loads taking greater than 30 minutes, at which point you can't afford to restart your application every time you need to tweak a shader. Enter shader hot reloading!

For this post we will be focusing on compiling our shaders through the [DirectX Shader Compiler][DXC] on DX12. There is an older shader compiler called FXC and you can ALSO do hot shader reloading using *D3DCompileFromFile*, however if you want support for raytracing and other modern shader compiler features on DX12, you'll need to use the DirectX Shader Compiler (also called DXC).


To do this is relatively straightforward:
 * We add some UI to trigger a recompile request
 * Point DXC to your shader files (the shader files need to be accessible from the executable for this to work)
 * Pass the shader file contents to DXC's compiler
 * If it compiled successfully, we swap out the runtime PSO for the shader we just compiled
	
The end result is something like this where you can live edit shaders in your text editor of choice and then hit a recompile button that will live reload the shader without needing to restart your app or load up the level:

![Output Image](/assets/HotReload/HotReload.jpg)

One quick note: If you want to live edit bindings in the shader, you'll also need to update your root signature as well as binding code so that will take extra work not covered here.

# Project Setup:
In order to call DXC at runtime you'll need to do a few things:
 * You'll want the [Windows SDK][WinSDK]. The chances are if you're using DX12 you already have this 
 * Link in the static libraries in "dxcompiler.lib" (part of the WinSDK)
 * And finally you'll want to grab "dxcompiler.dll" and "dxil.dll" from the latest on [DXC's release page][dxcrelease] and make sure these are packaged with your app so that they can be loaded at runtime. 
 
Technically you can build your own DXC compiler from their [Github page][DXC]. Unless you have very specific needs I don't recommend that and you'll bump into issues with shader signing.

# The code:
Luckily the code is actually quite small, here's a direct excerpt from my code. For context my project is a path tracer where everything is done from a single shader that's stored in m_pRayTracingPSO. In all likelihood your project is more complicated but it does mean we can reference an extremely isolated piece of code that's simple enough that you can probably copy-paste it and get it working for you with some tweaks. Hopefully the comments speak for themself

{% highlight c %}
#include "dxcapi.h"

#define HANDLE_FAILURE() assert(false);
#define VERIFY(x) if(!(x)) HANDLE_FAILURE();
#define VERIFY_HRESULT(x) VERIFY(SUCCEEDED(x))

void TracerBoy::RecompileShaders()
{
    LPCWSTR ShaderFile = L"..\\..\\TracerBoy\\RaytraceCS.hlsl";

    // Load up the compiler
    ComPtr<IDxcCompiler3> pCompiler;
    VERIFY_HRESULT(DxcCreateInstance(CLSID_DxcCompiler, IID_PPV_ARGS(pCompiler.GetAddressOf())));

    // Load up DXCUtils, not strictly necessary but has lots of useful helper functions
    ComPtr<IDxcUtils> pUtils;
    VERIFY_HRESULT(DxcCreateInstance(CLSID_DxcUtils, IID_PPV_ARGS(pUtils.GetAddressOf())));
    
    // Load the HLSL file into memory
    UINT32 CodePage = DXC_CP_UTF8;
    ComPtr<IDxcBlobEncoding> pSourceBlob;
    VERIFY_HRESULT(pUtils->LoadFile(ShaderFile, &CodePage, pSourceBlob.GetAddressOf()));
     
    // The include handler will handle the #include's in your HLSL file so that 
    // you don't need to handle bundling them all into a single blob. Technically
    // the include handler is optional not specifying this will cause the compiler to 
    ComPtr<IDxcIncludeHandler> pDefaultIncludeHandler;
    VERIFY_HRESULT(pUtils->CreateDefaultIncludeHandler(pDefaultIncludeHandler.GetAddressOf()));

    DxcBuffer SourceBuffer;
    BOOL unused;
    SourceBuffer.Ptr = pSourceBlob->GetBufferPointer();
    SourceBuffer.Size = pSourceBlob->GetBufferSize();
    pSourceBlob->GetEncoding(&unused, &SourceBuffer.Encoding);

    // Compiler args where you specify the entry point of your shader and what kind of shader it is (i.e. compute/pixel/vertex/etc)
    ComPtr<IDxcCompilerArgs> pCompilerArgs;
    VERIFY_HRESULT(pUtils->BuildArguments(ShaderFile, L"main", L"cs_6_5", nullptr, 0, nullptr, 0, pCompilerArgs.GetAddressOf()));

    // Finally, time to compile!
    ComPtr<IDxcResult> pCompiledResult;
    HRESULT hr = pCompiler->Compile(&SourceBuffer, pCompilerArgs->GetArguments(), pCompilerArgs->GetCount(), pDefaultIncludeHandler.Get(), IID_PPV_ARGS(pCompiledResult.GetAddressOf()));

    if (SUCCEEDED(hr))
    {
        // Output errors are all put in a buffer. Warning will also be output here, so don't assume that 
        // the existence of an error buffer means the shader compile failed
        ComPtr<IDxcBlobEncoding> pErrors;
        if (SUCCEEDED(pCompiledResult->GetErrorBuffer(pErrors.GetAddressOf())))
        {
            std::string ErrorMessage = std::string((const char*)pErrors->GetBufferPointer());
            OutputDebugString(ErrorMessage.c_str());
        }

        ComPtr<IDxcBlob> pOutput;
        if (SUCCEEDED(pCompiledResult->GetResult(pOutput.GetAddressOf())))
        {
            // Take the compiled shader code and compile it into a PSO
            D3D12_COMPUTE_PIPELINE_STATE_DESC psoDesc = {};
            psoDesc.pRootSignature = m_pRayTracingRootSignature.Get();
            psoDesc.CS = CD3DX12_SHADER_BYTECODE(pOutput->GetBufferPointer(), pOutput->GetBufferSize());

            // Check that the PSO compile worked, failure here could mean issues with root signature mismatches or you're using
            // features that the driver doesn't support
            ComPtr<ID3D12PipelineState> pNewRaytracingPSO;
            if (SUCCEEDED(m_pDevice->CreateComputePipelineState(&psoDesc, IID_GRAPHICS_PPV_ARGS(pNewRaytracingPSO.ReleaseAndGetAddressOf()))))
            {
                // Overwrite the old PSO. MAKE SURE THE OLD PSO ISN'T BEING REFERENCED ANYMORE!!
                // Particularly ensure the GPU isn't running any shaders that are referencing it.
                m_pRayTracingPSO = pNewRaytracingPSO;
            }
        }
    }
}
{% endhighlight %}

After that you're done. Hook that up to some UI (might I suggest [ImGUI][ImGUI]?) and reload away!

For more interesting DXC info:
 * [DXC docs][DXC]
 * [Using the DirectXShaderCompiler C++ API][DXCBlog]
 * [Two Shader Compilers of Direct3D 12 (FXC vs DXC)][asawicki]

[asawicki]: https://asawicki.info/news_1719_two_shader_compilers_of_direct3d_12
[DXCBlog]: https://simoncoenen.com/blog/programming/graphics/DxcCompiling
[WinSDK]: https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/
[DXC]: https://github.com/microsoft/DirectXShaderCompiler	
[ImGUI]: https://github.com/ocornut/imgui
[Disney]: https://disneyanimation.com/resources/moana-island-scene/
[DXCRelease]: https://github.com/microsoft/DirectXShaderCompiler/releases