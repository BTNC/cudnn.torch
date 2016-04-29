cudnn.torch
===========

Torch7 FFI bindings for NVIDIA cuDNN (R4) kernels!

Modules are API compatible their [`nn`](https://github.com/torch/nn) equivalents. Fully unit-tested against `nn` implementations.
Conversion between `nn` and `cudnn` is available through `cudnn.convert` function.

#### Installation

* Install cuDNN (version R4 EA)
* Have at least CUDA 7.0
* Have `libcudnn.so` in your library path (Install it from https://developer.nvidia.com/cuDNN )

#### Modules

```lua
-- All inputs have to be 3D or 4D(batch-mode), except ReLU, Tanh, Sigmoid, and BatchNormalization
cudnn.SpatialConvolution(nInputPlane, nOutputPlane, kW, kH, [dW = 1], [dH = 1], [padW = 0], [padH = 0], [groups = 1])
cudnn.SpatialMaxPooling(kW, kH, dW, dH, padW, padH)
cudnn.SpatialAveragePooling(kW, kH, dW, dH, padW, padH)

-- the pointwise functions take an additional optional argument. if inplace=true then they do operations in-place without using any extra memory for themselves
cudnn.ReLU(inplace[=false])
cudnn.Tanh(inplace[=false])
cudnn.Sigmoid(inplace[=false])

-- SoftMax can be run in fast mode or accurate mode. Default is accurate mode.
cudnn.SoftMax(fastMode [= false])          -- SoftMax across each image (just like nn.SoftMax)
cudnn.LogSoftMax()                         -- LogSoftMax across each image (just like nn.LogSoftMax)
cudnn.SpatialSoftMax(fastMode [= false])   -- SoftMax across feature-maps (per spatial location)
cudnn.SpatialLogSoftMax()                  -- LogSoftMax across feature-maps (per spatial location)

cudnn.SpatialCrossEntropyCriterion()       -- A spatial version of LogSoftMax + ClassNLLCriterion in one shot

-- Volumetric inputs (4D or 5D batched mode)
cudnn.VolumetricConvolution(nInputPlane, nOutputPlane, kT, kW, kH, dT, dW, dH, padT, padW, padH)
cudnn.VolumetricMaxPooling(kT, kW, kH, dT, dW, dH, padT, padW, padH)
cudnn.VolumetricAveragePooling(kT, kW, kH, dT, dW, dH, padT, padW, padH)
```

### Modes
There are two globally availabe modes useful for tuning performance:
```lua
require 'cudnn'
cudnn.benchmark = true -- uses the inbuilt cudnn auto-tuner to find the fastest convolution algorithms.
                       -- If this is set to false, uses some in-built heuristics that might not always be fastest.
```
by default `cudnn.benchmark` is set to `false`.  Setting to `true` will improve performance, at the expense of using more
memory.  The input shape should be the same for each batch, otherwise autotune will re-run for each batch,
causing a huge slow-down.

```lua
cudnn.fastest = true -- this is like the :fastest() mode for the Convolution modules,
                     -- simply picks the fastest convolution algorithm, rather than tuning for workspace size
```
by default, `cudnn.fastest` is set to `false`.  You should set to `true` if memory is not an issue, and you
want the fastest performance


```lua
cudnn.verbose = true -- this prints out some more verbose information useful for debugging
```
by default, `cudnn.verbose` is set to `false`.

### Conversion between `cudnn` and `nn`

Conversion is done by `cudnn.convert` function which takes a network and backend arguments and goes over
network modules recursively substituting equivalents. No memory copy is done, just metatables are swapped.
If you don't want to convert all modules you can pass a function as the third argument to `cudnn.convert`.
It will be called at each step, with a module that is currently converted.  It is meant to exclude
modules i.e. if it returns `true`, they will be left untouched, otherwise they will be subject to conversion.

```lua
net = nn.Sequential()
net:add(nn.SpatialConvolution(3,96,11,11,3,3))
net:add(nn.ReLU())
cudnn.convert(net, cudnn)
print(net)

net = nn.Sequential()
net:add(nn.SpatialConvolution(3,96,11,11,3,3))
net:add(nn.ReLU())
cudnn.convert(net, cudnn, function(module)
   return torch.type(module):find('ReLU')
end)
print(net)
```

will result in:
```
nn.Sequential {
  [input -> (1) -> (2) -> output]
  (1): cudnn.SpatialConvolution(3 -> 96, 11x11, 3,3)
  (2): cudnn.ReLU
}
nn.Sequential {
  [input -> (1) -> (2) -> output]
  (1): cudnn.SpatialConvolution(3 -> 96, 11x11, 3,3)
  (2): nn.ReLU
}
```

### Older versions
For version CuDNN R1, checkout the branch **R1**
For version CuDNN R2, checkout the branch **R2**
For version CuDNN R3, checkout the branch **R3**


R4 Release Notes:
- Rather than resolving v3-v4 diffs, I have imported new cudnn.h with its entirety and converted comments and defines. This should be less error-prone.
- addTensor_v2 uses changed to new AddTensor API.
