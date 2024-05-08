---
title: "Gfxreconstruct replay Linux/Windows game trace"
date: 2023-04-17T13:50:36+08:00
draft: false
---

最近在研究vkd3d的性能, 在开始研究之前, 有一个前置任务: 
用gfxreconstruct去replay game trace, 同时要确保这个trace既能在windows上面运行, 又能在linux上面运行. 
这样可以方便我们比较linux和windows的差异.
本文记录一下, 在enable replay trace的时候, 遇到的一些问题, 以及解决方案.

第一个尝试是: 在windows上面用vkd3d运行dx12 game, 然后把trace抓下来, 再拿到linux上面去replay.
这样的做法是保证: 游戏看到的vulkan extension在不同OS上是基本一致的(因为都要经过vkd3d)

但是当我试图在linux上面replay这个trace的时候, 果不其然失败了(事情总是没有那么容易).
遇到了以下错误.
```
  1 [gfxrecon] WARNING - Mismatch:
  2 [gfxrecon] WARNING - Captured application name: Cyberpunk2077.exe
  3 [gfxrecon] WARNING - Replayer process name: gfxrecon-replay.exe
  4 [gfxrecon] WARNING - This can lead to diverging driver behavior between the replayer and captured application
  5 [gfxrecon] WARNING - Recommendation: Rename gfxrecon-replay.exe to match the application's executable name
  6 [gfxrecon] DEBUG - Match: replay-time GPU vs capture-time GPU
  7 [gfxrecon] WARNING - ID3D12Device_CreateComputePipelineState returned S_OK, which does not match the value returned at capture E_INVALIDARG.
  8 [gfxrecon] WARNING - ID3D12Device_CreateComputePipelineState returned S_OK, which does not match the value returned at capture E_INVALIDARG.
  9 [gfxrecon] WARNING - Requested window size (1920x1080) exceeds current screen size (1280x800); replay may fail due to inability to create a window of the a    ppropriate size.
 10 [gfxrecon] WARNING - ID3D12Device_CreatePlacedResource returned E_INVALIDARG, which does not match the value returned at capture S_OK.

```
关键的错误信息是"ID3D12Device_CreatePlacedResource"的失败调用.

接下来, 我们打开log开关(VKD3D_DEBUG=trace, VKD3D_LOG_FILE=vkd3d_trace.log
), 然后在vkd3d里面发现了一句可疑log.
```
0128:err:d3d12_resource_create_placed: Heap too small for the texture (heap=3211264, res=3407872.
```
这个逻辑是对dx12 api - ID3D12Device::CreatePlacedResource的处理. 问题看起来比较清楚了: 当它想要把一个texture image放进heap的时候, 发现这个heap太小了, 于是就报错了.

DX12里面关于这个API的用法是这样的:
```
1. app调用getResourceAllocationInfo获取texture image的大小
2. app根据返回值, 决定自己要创建多大的heap, 然后调用ID3D12Device::CreateHeap
3. app调用ID3D12Device::CreatePlacedResource, 把指定的texture放入指定的heap中.
```

另外, 这里还有一个问题是: 为什么同样的trace在windows上面replay就不会遇到"heap too small"的情况呢.
通过以下的log, 可以发现: 对于个别texture, 它们的size在windows和linux上面是不一样的.
![image.png](/images/post_6/16134950-4f98f68f43954015.png)

明白了root cause之后, 我做了一个简单的workaround: 让getResourceAllocationInfo返回一个原来6倍的值, 这样的话, 抓下来的trace就能同时满足windows和linux的size了.

```
--- a/libs/vkd3d/device.c
+++ b/libs/vkd3d/device.c
@@ -5440,7 +5440,7 @@ static D3D12_RESOURCE_ALLOCATION_INFO* 
 STDMETHODCALLTYPE d3d12_device_GetResourc
              resource_info.Alignment = max(resource_info.Alignment, requested_alignment);
         }

-        resource_info.SizeInBytes = align(resource_info.SizeInBytes, resource_info.Alignment);
+        resource_info.SizeInBytes = align(resource_info.SizeInBytes*6, resource_info.Alignment);
          resource_offset = align(info->SizeInBytes, resource_info.Alignment);

          if (resource_infos)
```
这个改动是发生在vkd3d里面的, 所以需要在windows + vkd3d的环境下, 才可以抓到正确的trace.
但是有些游戏在windows + vkd3d下面不能正确运行, 于是有另一个想法: 在d3d12.dll外面再包一层, 写一个proxy dll, 游戏是先调用到proxy dll, 然后proxy dll再调用到系统的d3d12.dll. 在这个调用过程中, 我们可以修改getResourceAllocationInfo的返回值.

代码在这里: https://github.com/Zhiwei-Lii/dx12Wrapper
可以导入到Visual studio里面编译, 编出来的d3d12.dll是一个proxy dll, 它会调用d3d12_real.dll (需要把系统的dll重命名成d3d12_real.dll). 然后把d3d12.dll和d3d12_real.dll放到游戏目录下, 即可运行.
