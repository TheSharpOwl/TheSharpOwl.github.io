---
title: Compiling HLSL Shaders with Direct3D12 and CMake (With Error Report) [IN PROGRESS]
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
{: .prompt-info }

I will isolate the [HelloTriangle](https://github.com/microsoft/DirectX-Graphics-Samples/tree/master/Samples/Desktop/D3D12HelloWorld/src/HelloTriangle) example alone and use CMake
as a build system for that standalone project.

> You don't have to do anything but I just wanted to explain how I got to the CMake file in the repository for the code of this post. Also, A true C++ programmer should be familiar with CMake :P
{: .prompt-info }

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

# f
set(HelloTriangle1_HEADERS
D3D12HelloTriangle.h
DXSample.h
DXSampleHelper.h
stdafx.h
Win32Application.h
)

# g
set(HelloTriangle1_SHADERS
shaders.hlsl
)

# h
set_source_files_properties(shaders.hlsl PROPERTIES VS_TOOL_OVERRIDE "None")

# i
add_executable(HelloTriangle1 WIN32 Main.cpp ${HelloTriangle1_HEADERS} ${HelloTriangle1_SOURCES} ${HelloTriangle1_SHADERS})
# j
target_include_directories(HelloTriangle1 PUBLIC ${CMAKE_CURRENT_LIST_DIR})
source_group("Shaders" FILES ${HelloTriangle1_SHADERS})
```

Explaination of the lines:

* 1 to 2: In any cmake file we should define a project name and the minimum version which the person who is building the project should have.
* 4 to 5: Because we use wide string aka `std::wstring` in C++ for Direct3D API, this definition should be added from CMake otherwise we will get errors because the project will not know if you should use ANSI or UNICODE so you should define which one is it for the project [More info in this question on StackOverFlow](https://stackoverflow.com/questions/6401036/unicode-compiler-error-on-simple-function).
* 7 to 12: We should link to Direct3D libraries but here I just used `link_libraries` to link to all the libraries and executables because the project is already small and that won't be an issue here.
* 


To be continued...