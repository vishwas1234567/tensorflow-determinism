# TensorFlow Determinism

This repository serves three purposes:

  1. Provide up-to-date information (in this file) about non-determinism
     sources and solutions in TensorFlow and beyond, with a focus on determinism
     when running on GPUs.
  2. Provide a patch to attain various levels of GPU-specific determinism in
     stock TensorFlow, via the installation of the `tensorflow-determinism` pip
     package.
  3. Be the location where a TensorFlow determinism debug tool will be released
     as part of the `tensorflow-determinism` pip package.

For more information, please watch the video of the GTC 2019 talk
[_Determinism in Deep Learning_][1]. The desciption under that video also
includes links to the slides from the talk and to a poster presentation on this
topic.

## Installation

Use `pip` to install:

```
pip install tensorflow-determinism
```

This will install a package that can be imported as `tfdeterminism`. The
installation of `tensorflow-determinism` will not automatically install
TensorFlow. The intention of this is to allow you to install your chosen
version of TensorFlow. You will need to install your chosen version of
TensorFlow before you can import and use `tfdeterminism`.

## Deterministic TensorFlow Solutions

There are currently two main ways to access GPU-deterministic functionality in
TensorFlow for most deep learning applications. The first way is to use an
NVIDIA NGC TensorFlow container. The second way is to use version 1.14, 1.15,
or 2.0 of stock TensorFlow with GPU support, plus the application of a patch
supplied in this repo.

The longer-term intention and plan is to upstream all solutions into stock
TensorFlow.

Determinism is not guaranteed when XLA JIT compilation is enabled.

### NVIDIA NGC TensorFlow Containers

NGC TensorFlow containers, starting with version 19.06, implement
GPU-deterministic TensorFlow functionality. In Python code running inside the
container, this can be enabled as follows:

```
import tensorflow as tf
import os
os.environ['TF_DETERMINISTIC_OPS'] = '1'
# Now build your graph and train it
```

The following table shows which version of TensorFlow each NGC container
version is based on:

 NGC Container Version | TensorFlow Version |
:----------------------|:-------------------|
 19.06                 | 1.13               |
 19.07 - 19.10         | 1.14               |
 19.11 - 19.12         | 1.15 / 2.0         |

For information about pulling and running the NVIDIA NGC containers, see [these
instructions][2].

### Stock TensorFlow

Versions 1.14, 1.15, and 2.0 of stock TensorFlow implement a reduced form of GPU
determinism, which must be supplemented with a patch provided in this repo.
The following Python code is running on a machine in which `pip` package
`tensorflow-gpu=2.0.0` has been installed correctly and on which
`tensorflow-determinism` has also been installed (as shown in the
[installation](#installation) section above).

```
import tensorflow as tf
from tfdeterminism import patch
patch()
# use tf as normal
```

Stock TensorFlow with GPU support can be installed as follows:

```
pip install tensorflow-gpu=2.0.0
```

The TensorFlow project includes [detailed instructions][3] for installing
TensorFlow with GPU support.

### Additional Ingredients in the Determinism Recipe

#### Seeds ####

You'll also need to set any and all appropriate random seeds:

```
os.environ['PYTHONHASHSEED']=str(SEED)
random.seed(SEED)
np.random.seed(SEED)
tf.set_random_seed(SEED)
```

#### Dataset Sharding ####

If you're using `tf.data.Dataset`, you should not shard the dataset. This
is achieved by either not calling the `shard()` method, or by setting its
`num_shards` parameter to 1.

#### Gradient Gating ####

For deterministic functionality, some types of models may require
`gate_gradients=tf.train.Optimizer.GATE_OP` in the session config.

#### Multi-GPU with Horovod ####

If you're using Horovod for multi-GPU training, you may need to disable Tensor
Fusion (assuming that the non-determinism associated with Tensor Fusion has not
yet been resolved):

```
os.environ['HOROVOD_FUSION_THRESHOLD']='0'
```

#### CPU ####

If you want to obtain determinism when your ops are running on the CPU, you may
need to limit the number of CPU threads used:

```
session_config.intra_op_parallelism_threads = 1
session_config.inter_op_parallelism_threads = 1
```

It should not be necessary to do this when your ops are not running on the CPU
(e.g. when they're running on a GPU).

## Detailed Status of Determinism in TensorFlow and Beyond

Confirmed and likely sources of non-determinism, along with any existing
solutions, are being tracked here.

### GPU-Specific Sources of Non-Determinism

#### Historic GPU-Specific Sources of Non-Determinism

In the past, `tf.math.reduce_sum` and `tf.math.reduce_mean` operated
non-deterministically when running on a GPU. This was resolved before
TensorFlow version 1.12. These ops now function deterministically
by default when running on a GPU.

#### Confirmed Current GPU-Specific Sources of Non-Determinism (With Solutions)

 Source                                                               | NGC 19.06+ / TF 2.1 | TF 1.14, 1.15, 2.0  |
:---------------------------------------------------------------------|:--------------------|:--------------------|
 TF auto-tuning of cuDNN convolution algorithms (see multi-algo note) | TCD or TDO          | TCD or TDP          |
 cuDNN convolution backprop to weight gradients                       | TCD or TDO          | TCD or TDP          |
 cuDNN convolution backprop to data gradients                         | TCD or TDO          | TCD or TDP          |
 cuDNN max-pooling backprop                                           | TCD or TDO          | TCD or TDP          |
 `tf.nn.bias_add` backprop (see XLA note)                             | TDO                 | TDP                 |
 `tf.image.resize_bilinear` fwd and bwd                               | NS1                 | NS1                 |

Key to the solutions refenced above:

 Solution | Description                                                                                                                                                                                     |
:---------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
 TCD      | Set environment variable `TF_CUDNN_DETERMINISTIC` to '1' or 'true'. Also *do not* set environment variable `TF_USE_CUDNN_AUTOTUNE` at all (and particularly *do not* set it to '0' or 'false'). |
 TDO      | Set environment variable `TF_DETERMINISTIC_OPS` to '1' or 'true'. Also *do not* set environment variable `TF_USE_CUDNN_AUTOTUNE` at all (and particularly *do not* set it to '0' or 'false').   |
 TDP      | Apply `tfdeterminism.patch`. Note that solution TDO will be in stock TensorFlow v2.1 (see [PR 31465](https://github.com/tensorflow/tensorflow/pull/31465)).                                     |
 NS1      | There is currently no solution available for this, but one is under development.                                                                                                                |

Notes:
  * multi-algo: From NGC 19.12 onwards, the cuDNN forward and backward
    convolution algorithms are selected deterministically from several
    deterministic algorithms. Prior to this (i.e. NGC 19.11 and earlier, and all
    currently released versions of stock TensorFlow), there is only one
    deterministic algorithm selected for each of the forward and two backward
    paths. In those versions of TensorFlow, some layer configurations
    are not supported (resulting in an exception). The multi-algorithm support
    is not currently available in stock TensorFlow, but is being added by
    PR [34951](https://github.com/tensorflow/tensorflow/pull/34951).
  * XLA: These solutions will not work when XLA JIT compilation is enabled.

#### Other Possible GPU-Specific Sources of Non-Determinism

Going beyond the above-mentioned sources, in version 1.12 of TensorFlow (and
also in the master branch on 2019-03-03, afer release 1.31.1), the following
files call CUDA `atomicAdd` either directly or indirectly. This makes them
candidates for the injection of non-determinism.

* `crop_and_resize_op_gpu.cu.cc`
* `scatter_functor_gpu.cu.h`
* `scatter_nd_op_gpu.cu.cc`
* `sparse_tensor_dense_matmul_op_gpu.cu.cc`
* `resize_nearest_neighbor_op_gpu.cu.cc`
* `segment_reduction_ops.h`
* `segment_reduction_ops_gpu.cu.cc`
* `dilation_ops_gpu.cu.cc`
* `maxpooling_op_gpu.cu.cc`
* `svd_op_gpu.cu.cc`
* `cuda_kernel_helper_test.cu.cc`
* `depthwise_conv_op_gpu.h`
* `resampler_ops_gpu.cu.cc`
* `histogram_op_gpu.cu.cc`
* `stateful_random_ops_gpu.cu.cc`

Unless you are using TensorFlow ops that depend on these files (i.e. ops with
similar names), then your model will not be affected by these potential sources
of non-determinism.

Beyond `atomicAdd`, there are ten other CUDA [atomic functions][4] whose use
could lead to the injection of non-determinism, such as `atomicCAS` (the most
generic, atomic compare and swap). Note also that the word 'atomic' was present
in 167 files in the TensorFlow repo and some of these may be related to the use
of CUDA atomic operations. It's important to remember that it's possible to use
CUDA atomic operations without injecting non-determinism, and that, therefore,
when CUDA atomic operations are present in op code, it doesn't guarantee that
the op injects non-determinism into the computation.

### Sources of Non-Determinism in TensorFlow Unrelated to GPU

* [Issue 29101](https://github.com/tensorflow/tensorflow/issues/29101): Random
  seed not set in graph context of `Dataset#map`. This may have been resolved
  in version 1.14 of TensorFlow.
* `tf.data.Dataset` with more than one shard (aka worker). The work-around is to
  use only one shard.

### Sources of Non-Determinism Beyond TensorFlow

* TensorRT timing-based kernel schedule. Each time an inference engine is
  generated, it could be slightly different, particularly if there is varying
  load on the machine used to run TensorRT. There is a solution planned for
  this.
* Horovod Tensor Fusion. Work-around: disable Tensor Fusion by setting the
  environment variable `HOROVOD_FUSION_THRESHOLD` to '0'. This issue may have
  been resolved by Horovod
  [pull-request 1130](https://github.com/horovod/horovod/pull/1130) (not yet
  confirmed).

## Relevant Links

This section catalogs relevant links.

### TensorFlow Issues

Number                                                         | Title                                                                                 | Updated    |
--------------------------------------------------------------:|:--------------------------------------------------------------------------------------|:-----------|
 [2652](https://github.com/tensorflow/tensorflow/issues/2652)  | Backward pass of broadcasting on GPU is non-deterministic                             | 2019-10-08 |
 [2732](https://github.com/tensorflow/tensorflow/issues/2732)  | Mention that GPU reductions are nondeterministic in docs                              | 2019-10-08 |
[13932](https://github.com/tensorflow/tensorflow/issues/13932) | Non-determinism from `tf.data.Dataset.map` with random ops                            |            |
[16889](https://github.com/tensorflow/tensorflow/issues/16889) | Problems Getting TensorFlow to behave Deterministically                               | 2019-10-08 |
[18096](https://github.com/tensorflow/tensorflow/issues/18096) | Feature Request: Support for configuring deterministic options of cuDNN conv routines | 2019-10-08 |
[29101](https://github.com/tensorflow/tensorflow/issues/29101) | Random seed not set in graph context of `Dataset#map`                                 |            |

### TensorFlow Pull Requests

Number                                                       | Title                                                         | Status | Date Merged | Version |
------------------------------------------------------------:|:--------------------------------------------------------------|:-------|:------------|:--------|
[10636](https://github.com/tensorflow/tensorflow/pull/10636) | Non-determinism Docs (see note 1)                             | closed |             |         |
[24273](https://github.com/tensorflow/tensorflow/pull/24273) | Enable dataset.map to respect seeds from the outer context    | closed |             |         |
[24747](https://github.com/tensorflow/tensorflow/pull/24747) | Add cuDNN deterministic env variable (only for convolution).  | merged | 2019-01-15  | 1.14    |
[25269](https://github.com/tensorflow/tensorflow/pull/25269) | Add deterministic cuDNN max-pooling                           | merged | 2019-01-30  | 1.14    |
[25796](https://github.com/tensorflow/tensorflow/pull/25796) | Added tests for `TF_CUDNN_DETERMINISTIC`                      | merged | 2019-02-22  | 1.14    |
[c2790][1001]<sup>2</sup>                                    | Add a decorator to disable autotuning during test executions. | merged | 2019-03-13  | 1.14    |
[29667](https://github.com/tensorflow/tensorflow/pull/29667) | Add release note about `TF_CUDNN_DETERMINISTIC`               | merged | 2019-08-06  | 1.14    |
[31389](https://github.com/tensorflow/tensorflow/pull/31389) | Enhance release notes related to `TF_CUDNN_DETERMINISTIC`     | merged | 2019-08-07  | 1.14    |
[31465](https://github.com/tensorflow/tensorflow/pull/31465) | Add GPU-deterministic `tf.nn.bias_add`                        | merged | 2019-10-17  | 2.1     |
[32979](https://github.com/tensorflow/tensorflow/pull/32979) | Fix typo in release note                                      | closed |             |         |
[33483](https://github.com/tensorflow/tensorflow/pull/33483) | Fix small typo in v2.0.0 release note                         | merged | 2019-10-25  | 2.1     |
[33803](https://github.com/tensorflow/tensorflow/pull/33803) | Enable tf.nn.bias_add python op tests to work in eager mode   | open   |             |         |
[33900](https://github.com/tensorflow/tensorflow/pull/33900) | Address problems with use_deterministic_cudnn test decorator  | closed |             |         |
[34887](https://github.com/tensorflow/tensorflow/pull/34887) | Add info about `TF_DETERMINISTIC_OPS` to v2.1 release notes   | merged | 2019-12-09  | 2.1     |
[34951](https://github.com/tensorflow/tensorflow/pull/34951) | Add multi-algorithm deterministic cuDNN convolutions          | open   |             |         |
[35006](https://github.com/tensorflow/tensorflow/pull/35006) | Fix version 2.1 release note regarding TF_DETERMINISTIC_OPS   | merged | 2019-12-20  | 2.1     |

[1001]: https://github.com/tensorflow/tensorflow/commit/c27909ea80e8823dbf4f7176ab69991a630356a1

Notes:
  1. Updated on 2019-10-08
  2. This was effectively a stand-alone commit

### Miscellaneous

* Two Sigma: [A Workaround for Non-Determinism in
  TensorFlow](http://bit.ly/two-sigma-determinism)
* Keras [issue 12800](https://github.com/keras-team/keras/issues/12800):
  Unable to get reproducible results using Keras with TF backend on GPU (updated
  on 2019-10-08)
* PyTorch [Reproducibility](http://bit.ly/pytorch-determinism) (from the
  official documentation)
* Chainer [PR 2710](https://github.com/chainer/chainer/pull/2710): cuDNN
  Deterministic mode
* Stack Overflow: [Tensorflow: Different results with the same random seed][501]
* Stack Overflow: [Are tensorflow random values guaranteed to be the same inside a single run? (comment)][502] (updated 2019-10-10).

## Credits

Here are the names of some of the people who have helped out with this project.
If any names are missing, then please let us know.

Ben Barsdell, Kevin Brown, Carl Case, Bryan Catanzaro, Sharan Chetlur,
Joey Conway, Luke Durant, Marc Edgar, Mostafa Hagog, Tero Karras, Bob Keating,
Andrew Kerr, Xiang Bo Kong, Nicolas Koumchatzky, Jorge Albericio Latorre,
Simon Layton, Jose Alvarez Lopez, Nathan Luehr, Conrado Silva Miranda,
John Montrym, Michael O'Connor, Lauri Peltonen, Rakesh Ranjan,
Jussi Rasanen, Duncan Riach (PIC), Mikko Ronkainen, Dilip Sequeria,
Matthijs De Smedt, Kevin Vincent, Stephen Warren, Hao Wu, Yifang Xu, Tim Zaman,
William Zhang.

[501]: https://stackoverflow.com/questions/54047654/tensorflow-different-results-with-the-same-random-seed
[502]: https://stackoverflow.com/questions/52213325/are-tensorflow-random-values-guaranteed-to-be-the-same-inside-a-single-run#comment91376212_52213325

[1]: http://bit.ly/determinism-in-deep-learning
[2]: https://ngc.nvidia.com/catalog/containers/nvidia:tensorflow
[3]: https://www.tensorflow.org/install/gpu
[4]: https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#atomic-functions
