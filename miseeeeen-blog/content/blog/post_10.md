---
title: "pytorch模型转换到onnx的踩坑记录"
date: 2025-05-30T08:13:49+08:00
draft: false
---
作为AI初学者, 我在将FlashOcc模型从PyTorch移植到OpenVINO的过程中遇到了各种问题.
虽然仍然没有成功转成openvino, 但是本文会列出一些自己踩过的坑, 希望能帮助类似问题的开发者.

## 环境准备
- **硬件**：NVIDIA RTX 4090
- **模型**：https://github.com/Yzichen/FlashOCC
- **软件**：pytorch 1.10, cuda 11.1(软件版本和FlashOCC的需求对齐)

## 步骤概述
1. 在 NVIDIA 4090 上运行FlashOcc
2. 将 PyTorch 模型转换为 ONNX
3. 将 ONNX 转换为 OpenVINO IR
4. 在设备上进行 IR 推理(NPU、GPU)

## 阶段一：在RTX 4090上运行FlashOcc

### 问题1：CUDA求解器内部错误
错误堆栈
```
Traceback (most recent call last):
  File "run_flashocc.py", line 87, in <module>
    main()
  File "run_flashocc.py", line 73, in main
    output = model(input_data)
  File "/opt/conda/lib/python3.8/site-packages/torch/nn/modules/module.py", line 1102, in _call_impl
    return forward_call(*input, **kwargs)
  File "/project/models/flashocc.py", line 215, in forward
    bev_feat = self.bev_encoder(bev_feat)
  File "/opt/conda/lib/python3.8/site-packages/torch/nn/modules/module.py", line 1102, in _call_impl
    return forward_call(*input, **kwargs)
  File "/project/models/backbones.py", line 178, in forward
    x = layer(x)
  File "/opt/conda/lib/python3.8/site-packages/torch/nn/modules/module.py", line 1102, in _call_impl
    return forward_call(*input, **kwargs)
  File "/project/models/backbones.py", line 104, in forward
    x = self.conv(x)
  File "/opt/conda/lib/python3.8/site-packages/torch/nn/modules/module.py", line 1102, in _call_impl
    return forward_call(*input, **kwargs)
  File "/project/ops/bev_pool_v2/bev_pool.py", line 152, in forward
    output = bev_pool_v2(depth, feat, ranks_depth, ranks_feat, ranks_bev)
  File "/project/ops/bev_pool_v2/bev_pool.py", line 84, in forward
    inv = torch.inverse(extrin)
RuntimeError: cusolver error: CUSOLVER_STATUS_INTERNAL_ERROR,
when calling `cusolverDnCreate(handle)`
```
原因分析
怀疑是RTX 4090的CUDA架构与PyTorch 1.10的torch.inverse()函数存在兼容性问题.

解决方案
有两个方法:
(1) 升级torch版本, 副作用是要解决可能存在的依赖问题
(2) 尝试一下cpu版本的inverse
这里博主尝试了方案2, 将矩阵求逆操作回退到CPU执行.
```
修改前:
global2keyego = torch.inverse(keyego2global.double())   # (B, 1, 4, 4)

修改后:
keyego2global_cpu = keyego2global.double().to('cpu')
global2keyego_cpu = torch.inverse(keyego2global_cpu)
global2keyego = global2keyego_cpu.to('cuda')

```
解决这个问题之后, FlashOcc就能够成功完成推理了.

## 阶段二：PyTorch转ONNX模型
### 问题2：kwargs参数的支持问题
onnx export的过程是需要一个dummy input的, 因为目前主流是使用tracing的办法去完成pytorch->onnx的转换. 它需要一个dummpy input, 来把模型实际跑一遍, 同时生成一个计算图. 
但是flashocc的forward函数是下面这样的, 
```
def forward(self, return_loss=True, **kwargs)
``` 
所以我们需要重写模型forward函数, 把参数展开, 消除kwargs的出现.
PS: 在pytorch 2.5.0里面, export函数已经支持kwargs的参数了.

### 问题3：Numpy数组转换错误
错误堆栈
```
Traceback (most recent call last):
  File "tools/test.py", line 314, in <module>
    main()
  File "tools/test.py", line 261, in main
    onnx_program = torch.onnx.export(model, onnx_input, "model.onnx")
  File "/myenv/lib/python3.8/site-packages/torch/onnx/__init__.py", line 316, in export
    return utils.export(model, args, f, export_params, verbose, training,
  File "/myenv/lib/python3.8/site-packages/torch/onnx/utils.py", line 107, in export
    _export(model, args, f, export_params, verbose, training, input_names, output_names,
  File "/myenv/lib/python3.8/site-packages/torch/onnx/utils.py", line 724, in _export
    _model_to_graph(model, args, verbose, input_names,
  File "/myenv/lib/python3.8/site-packages/torch/onnx/utils.py", line 493, in _model_to_graph
    graph, params, torch_out, module = _create_jit_graph(model, args)
  File "/myenv/lib/python3.8/site-packages/torch/onnx/utils.py", line 437, in _create_jit_graph
    graph, torch_out = _trace_and_get_graph_from_model(model, args)
  File "/myenv/lib/python3.8/site-packages/torch/onnx/utils.py", line 388, in _trace_and_get_graph_from_model
    torch.jit._get_trace_graph(model, args, strict=False, _force_outplace=False, _return_inputs_states=True)
  File "/myenv/lib/python3.8/site-packages/torch/jit/_trace.py", line 1166, in _get_trace_graph
    outs = ONNXTracedModule(f, strict, _force_outplace, return_inputs, _return_inputs_states)(*args, **kwargs)
  File "/myenv/lib/python3.8/site-packages/torch/nn/modules/module.py", line 1102, in _call_impl
    return forward_call(*input, **kwargs)
  File "/myenv/lib/python3.8/site-packages/torch/jit/_trace.py", line 127, in forward
    graph, out = torch._C._create_graph_by_tracing(
  File "/myenv/lib/python3.8/site-packages/torch/jit/_trace.py", line 121, in wrapper
    out_vars, _ = _flatten(outs)
RuntimeError: Only tuples, lists and Variables are supported as JIT inputs/outputs. Dictionaries and strings are also accepted, but their usage is not recommended. Here, received an input of unsupported type: numpy.ndarray
```
错误发生在: 将模型转化成torch.jit.ScriptModule的过程中. 这里的信息比较明确, 不支持numpy.ndarray这种类型. 所以需要做如下的改动:
```
# occ_res = occ_res.cpu().numpy().astype(np.uint8) 
occ_res = occ_res.to(torch.uint8)
```
### 问题4：小心中间结果常量化
pytorch转成onnx的过程中需要注意两点:
1. 模型的输入尽量使用tensor类型
2. 任何基于input的中间计算结果也使用tensor类型

我们已知转化是基于一组示例输入去进行的, pytorch会把示例输入带入到模型的forward函数中, 然后把整个模型结构转换成计算图. 假设有这么一个输入: torch.tensor(3). 然后forward过程中有一句代码是从tensor(3)转成int 3: a = torch.tensor(3).item(). 那么export过程无法记录tensor->int的转换, 于是只有a = 3被记录在模型中. 下次换一个input, 比如说torch.tensor(4), a仍然等于3, 这样就破坏了模型本身的结构.

所幸的是, 如果转换的过程中可能出现常量化, pytorch会打印出相关的warning. 我们只需要依照warning去修正对应的代码就好了. 例如下面这样的warning：
```
/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/necks/view_transformer.py:340: TracerWarning: Using len to get tensor shape might cause the trace to be incorrect. Recommended usage would be tensor.shape[0]. Passing a tensor of different shape might lead to errors or silently give incorrect results.
  if len(kept)==0:

/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/necks/view_transformer.py:361: TracerWarning: Using len to get tensor shape might cause the trace to be incorrect. Recommended usage would be tensor.shape[0]. Passing a tensor of different shape might lead to errors or silently give incorrect results.
  if len(interval_starts)==0:

/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/necks/view_transformer.py:289: TracerWarning: Converting a tensor to a Python integer might cause the trace to be incorrect. We can't record the data flow of Python values, so this value will be treated as a constant in the future. This means that the trace might not generalize to other inputs!
  bev_feat_shape=(depth.shape[0], int(self.grid_size[2]))
```

Pytorch官方文档上有更详细的说明: 
Avoid NumPy and built-in Python types
PyTorch models can be written using NumPy or Python types and functions, but during tracing, any variables of NumPy or Python types (rather than torch.Tensor) are converted to constants, which will produce the wrong result if those values should change depending on the inputs.
https://docs.pytorch.org/docs/stable/onnx_torchscript.html

PS: List of Tensor, Tuple of Tensor似乎也是支持的.

### 问题5：不支持的OP
在下面的调用栈里面, 遇到了"不支持的OP"的问题. 问题也比较清楚, bev_pool_v2是一个自定义的pytorch op, 由c++和cuda实现. 
```
forward(/flash_occ/FlashOCC/mmdetection3d/mmdet3d/models/detectors/base.py)
-> forward_test(/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/detectors/bevdet.py)
-> simple_test(/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/detectors/bevdet_occ.py)
-> extract_feat(/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/detectors/bevdet_occ.py)
-> extract_img_feat(/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/detectors/bevdet_occ.py)
-> forward(/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/necks/view_transformer.py)
-> view_transform(/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/necks/view_transformer.py)
-> view_transform_core(/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/necks/view_transformer.py)
-> bev_pool_v2(/flash_occ/FlashOCC/projects/mmdet3d_plugin/models/necks/view_transformer.py)
```
如果想解决这个问题, 需要做两件事:
1. 写一下bev_pool_v2(pytorch)->bev_pool_v2(onnx)的映射规则. 这个我理解应该是比较简单的. 因为最后onnx会转成openvino去部署, 所以不需要在onnx runtime层面上支持bev_pool_v2.
2. 在openvino那边支持bev_pool_v2这个op. 这个问题是初学者解决不了的Orz.

