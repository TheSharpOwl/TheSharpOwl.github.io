---
title: Compiling HLSL Shaders with Direct3D12 and CMake (With Error Reporting) Part 1
date: 2022-06-23 22:00:00 +/-0800
categories: [Computer_Graphics, Direct3D_12]
tags: [cpp, direct3d12, cmake, visual studio]
comments: false # temp TODO remove
---

Note: This post is still in progress (don't worry I am working on it almost everyday, I didn't abandon it and it will be ready very soon). However, feel free to message me about improving the current draft if you have any ideas

While learning Direct3D12 and working on my game engine [Mizu](https://github.com/TheSharpOwl/Mizu) I found that there are many ways to compile shaders but there is no clear documentation about the different shader compilation ways and how to apply them (especially if you are using CMake as a build system). Therefore, as usual no clear documentation? Time to write a post on this blog

# 1. Getting the intial HelloTriangle project

As a start I will be using Microsoft's Hello Triangle example from the [D3D12HelloWorld example](https://github.com/microsoft/DirectX-Graphics-Samples/tree/master/Samples/Desktop/D3D12HelloWorld).

> Visual Studio is needed of course (I am using 2022 version and 2015, 2017 should work for you as long as you have the suitable windows sdk and you can install the latest one from Visual Studio installer to be sure)
> {: .prompt-info }

I will isolate the [HelloTriangle](https://github.com/microsoft/DirectX-Graphics-Samples/tree/master/Samples/Desktop/D3D12HelloWorld/src/HelloTriangle) example alone and use CMake
as a build system for that standalone project.

> You don't have to do anything but I just wanted to explain how I got to the CMake file in the repository for the code of this post. Also, A true C++ programmer should be familiar with CMake :P
> {: .prompt-info }

1. Let's take only the source and header files in [HelloTriangle's](https://github.com/microsoft/DirectX-Graphics-Samples/tree/master/Samples/Desktop/D3D12HelloWorld/src/HelloTriangle) folder, put them in a folder called `HelloTriangle` (for example) and remove everything else.
2. Now we will create a `CMakeLists.txt` file:

```cmake
project("Hello Triangle")
cmake_minimum_required(VERSION 3.21)

add_definitions(-D_UNICODE)
add_definitions(-DUNICODE)

link_libraries(
d3d12.lib
dxguid.lib
DXGI.lib
d3dcompiler.lib
)

set(HelloTriangle1_SOURCES
D3D12HelloTriangle.cpp
DXSample.cpp
stdafx.cpp
Win32Application.cpp
)

set(HelloTriangle1_HEADERS
D3D12HelloTriangle.h
DXSample.h
DXSampleHelper.h
stdafx.h
Win32Application.h
)

set(HelloTriangle1_SHADERS
shaders.hlsl
)

set_source_files_properties(shaders.hlsl PROPERTIES VS_TOOL_OVERRIDE "None")

add_executable(HelloTriangle1 WIN32 Main.cpp ${HelloTriangle1_HEADERS} ${HelloTriangle1_SOURCES} ${HelloTriangle1_SHADERS})

target_include_directories(HelloTriangle1 PUBLIC ${CMAKE_CURRENT_LIST_DIR})
source_group("Shaders" FILES ${HelloTriangle1_SHADERS})
```

Explaination of the lines:

- 1 to 2: In any cmake file we should define a project name and the minimum version which the person who is building the project should have.
- 4 to 5: Because we use wide string aka `std::wstring` in C++ for Direct3D API, this definition should be added from CMake otherwise we will get errors because the project will not know if you should use ANSI or UNICODE so you should define which one is it for the project [More info in this question on StackOverFlow](https://stackoverflow.com/questions/6401036/unicode-compiler-error-on-simple-function).
- 7 to 12: We should link to Direct3D libraries but here I just used `link_libraries` to link to all the libraries and executables because the project is already small and that won't be an issue here.
- 14 to 19: Just make a list `HelloTriangle1_SOURCES` which contains the names of the `cpp` files.
- 21 to 27: same as previous point but a list for headers instead.
- 29 to 31: Making a list for Shader HLSL files (because we wanna tell Visual Studio to include them and treat them in a special way)
- 33: Visual Studio usually tries to compile the HLSL files in the default way automatically. For example, if we have a Pixel Shader it will detect its type and the HLSL compiler will make sure it is fine and without syntax errors. However, in this case we have put all shaders in one file and the default behaviour will cause compilation errors and settings should be changed. You can change it manually, but in this case you either have to upload the sln file and other related files with it, or you can just add this option in CMake which will make the project settings not using this default behaviour by default and not calling the shader compiler. We will talk about the shader compiler later in this post.
- 35: We add an executable called `HelloTriangle1` with type `WIN32` because otherwise it will think it's an console application by default and the way we write the main function in Direct3D applications will cause compiler errors and not work. Then, we add all our files from the 3 lists (there is no need usually to add headers but I just added everything all together with sources).

> If you don't add Shader files in the list of sources given to `add_executable`, they will not appear in the project in Visual Studio and it will be inconvinent to use another editor or window to modify their code.
> {: .prompt-danger }

- 37: We make the current `CMAKE_CURRENT_LIST_DIR` (where the current CMake file is) a directory visible to `#include` so that we can import our headers in any source file. Again, since the project is small I made everything visible to `#include`
- 38: `source_group` is used to make Visual Studio put the shader files inside a special folder (`Shaders` in this case) because it will be convinent for organization inside Visual Studio's solution explorer

> If you don't add a `source_group` for shaders, they will be in the same directory level of `source files` and `header files` in Visual Studio's solution explorer which works of course but feels weird and not organized in an elegant way
> {: .prompt-warning }

Now there are 3 main ways we can use shaders in our Direct3D program:

1. Using `D3DCompileFromFile` function
2. Let VS automatically do it for us
3. Using the [`DirectXShaderCompiler`](https://github.com/microsoft/DirectXShaderCompiler)

We will talk about the first and second one here (I will reference them here as I write them)

if you check the file [`D3D12HelloTriangle.cpp`](https://github.com/TheSharpOwl/Direct3DExamples/blob/main/HelloTriangle/D3D12HelloTriangle.cpp) line number [163](https://github.com/TheSharpOwl/Direct3DExamples/blob/2e469bb5a4470a5035ef41c742626c4a34517ebd/HelloTriangle/D3D12HelloTriangle.cpp#L163) and the one after it, you will see the following:

```cpp
ThrowIfFailed(D3DCompileFromFile(GetAssetFullPath(L"../../shaders.hlsl").c_str(), nullptr, nullptr, "VSMain", "vs_5_0", compileFlags, 0, &vertexShader, nullptr));
        ThrowIfFailed(D3DCompileFromFile(GetAssetFullPath(L"../../shaders.hlsl").c_str(), nullptr, nullptr, "PSMain", "ps_5_0", compileFlags, 0, &pixelShader, nullptr));

```

Here we use the function `ThrowIfFailed` which throws and exception if the expression inside it returns a failure through its `HLRESULT` return type.
Inside it, we got `D3DCompileFromFile` with the following signature:

```cpp
HRESULT D3DCompile(
  [in]            LPCVOID                pSrcData,
  [in]            SIZE_T                 SrcDataSize,
  [in, optional]  LPCSTR                 pSourceName,
  [in, optional]  const D3D_SHADER_MACRO *pDefines,
  [in, optional]  ID3DInclude            *pInclude,
  [in, optional]  LPCSTR                 pEntrypoint,
  [in]            LPCSTR                 pTarget,
  [in]            UINT                   Flags1,
  [in]            UINT                   Flags2,
  [out]           ID3DBlob               **ppCode,
  [out, optional] ID3DBlob               **ppErrorMsgs
);
```

Where:

1. `[in] pFileName`:
   A pointer to a constant null-terminated string that contains the name of the file that contains the shader code. it takes a wide string literall (usually passed like this: `L"content here"`).

   > Here I used `GetAssetFullPath function and passed the relative path so that it will return the full path like C:/Junk/shader.hlsl because the compiler takes the full path only (seems project relative one is different from the compiler relative one so GetAssetFullPath results gets evaluated relative to project then it's passed full to compiler. Remember that the compiler is in program files with visual studio compiler somewhere so it's relative folders and files are totally somewhere else). Also, I used `.c_str()` in the end because it accepts a literal not a wstring variable so I passed the literal out of the variable.
   > {: .prompt-warning }

2. `[in, optional] pDefines`:
   An optional array of D3D_SHADER_MACRO structures that define shader macros. Here we don't need that so we passed `nullptr` (or you can pass `NULL`)
3. `[in, optional] pInclude`:
   An optional pointer to an ID3DInclude interface that the compiler uses to handle include files. Here we didn't include any files so we pass `nullptr` too
4. `[in] pEntrypoint`:
   A pointer to a constant null-terminated string that contains the name of the main function of the shader (like `int main()` in C++ and C but here you don't have to call it `main` just tell the compiler where is it)
5. `[in] pTarget`: A pointer to a constant null-terminated string that specifies the shader target or set of shader features to compile against. So here I passed `vs_5_0` to first call to tell the function we wanna use version 5 and this is a vertex shader (vs) and same idea for second one to use the pixel shader (ps). You can check the list of such `compiler targets` [here](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/specifying-compiler-targets)
6. `[in] Flags1`:
   Shader compile options and we can use OR operator between many of them if we wanna use more than one. Full list of such options [here](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/d3dcompile-constants). Here we used `D3DCOMPILE_DEBUG | D3DCOMPILE_SKIP_OPTIMIZATION` in case of debug and nothing in case of Release and stored them in `compileFlags` variable.
7. `[in] Flags2`:
   Effect compile options (`Effect options` here and `Compiler options` in the ones before them). Full list [here](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/d3dcompile-effect-constants). They are not needed here so we passed 0.
8. `[out] ppCode`:
   A pointer to a variable that receives a pointer to the `ID3DBlob` interface that you can use to access the compiled code. So we passed here the reference to `ComPtr<ID3DBlob> vertexShader` which is `&vertexShader`.

> Don't forget that you have for ComPtr: RelaseAndGetAddressOf()which is the same as & or you can use GetAddressOf() function if you don't wanna release unlike what we did here because we wanna pass it to store the returned info so it does not matter
> {: .prompt-warning }

10. `[out, optional] ppErrorMsgs`
    An optional pointer to a variable that receives a pointer to the ID3DBlob interface and we can use it to get the compiler error messages. We passed `nullptr` for now and will show later how it works. I advise you to put it as a good practice to detect errors and understand them easier.

To be continued...

References:

1. [D3DCompileFromFile function Microsoft docs](https://learn.microsoft.com/en-us/windows/win32/api/d3dcompiler/nf-d3dcompiler-d3dcompilefromfile)
<!---
You can find the current project inside this folder in `Direct3DExamples` repository in this [link](https://github.com/TheSharpOwl/Direct3DExamples/tree/main/HelloTriangle)

Hope everything is clear and if you have any feedback feel free to contact me on discord, comment here or email.

TODO add advantages and disadvantages of this way and finish the tutorial for now
Thanks for reading !
-->
