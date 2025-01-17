# PULC 有无广告码分类模型

------


## 目录

- [1. 模型和应用场景介绍](#1)
- [2. 模型快速体验](#2)
    - [2.1 安装 paddlepaddle](#2.1)
    - [2.2 安装 paddleclas](#2.2)
    - [2.3 预测](#2.3)
- [3. 模型训练、评估和预测](#3)
    - [3.1 环境配置](#3.1)
    - [3.2 数据准备](#3.2)
      - [3.2.1 数据集来源](#3.2.1)
      - [3.2.2 数据集获取](#3.2.2)
    - [3.3 模型训练](#3.3)
    - [3.4 模型评估](#3.4)
    - [3.5 模型预测](#3.5)
- [4. 模型压缩](#4)
  - [4.1 SKL-UGI 知识蒸馏](#4.1)
    - [4.1.1 教师模型训练](#4.1.1)
    - [4.1.2 蒸馏训练](#4.1.2)
- [5. 超参搜索](#5)
- [6. 模型推理部署](#6)
  - [6.1 推理模型准备](#6.1)
    - [6.1.1 基于训练得到的权重导出 inference 模型](#6.1.1)
    - [6.1.2 直接下载 inference 模型](#6.1.2)
  - [6.2 基于 Python 预测引擎推理](#6.2)
    - [6.2.1 预测单张图像](#6.2.1)
    - [6.2.2 基于文件夹的批量预测](#6.2.2)
  - [6.3 基于 C++ 预测引擎推理](#6.3)
  - [6.4 服务化部署](#6.4)
  - [6.5 端侧部署](#6.5)
  - [6.6 Paddle2ONNX 模型转换与预测](#6.6)


<a name="1"></a>

## 1. 模型和应用场景介绍

该案例提供了用户使用 PaddleClas 的超轻量图像分类方案（PULC，Practical Ultra Lightweight image Classification）快速构建轻量级、高精度、可落地的有无广告码的分类模型(这里定义广告码包含：二维码、条形码、小程序码)。该模型可以广泛应用于直播场景、审核场景、海量数据过滤场景等。

下表列出了判断图片中有无广告码二分类模型的相关指标，前两行展现了使用 SwinTranformer_tiny 和 MobileNetV3_small_x0_35 作为 backbone 训练得到的模型的相关指标，第三行至第六行依次展现了替换 backbone 为 PP-LCNet_x1_0、使用 SSLD 预训练模型、使用 SSLD 预训练模型 + EDA 策略、使用 SSLD 预训练模型 + EDA 策略 + SKL-UGI 知识蒸馏策略训练得到的模型的相关指标。


| 模型 | Accuracy（%） | 延时（ms） | 存储（M） | 策略 |
|-------|-----------|----------|---------------|---------------|
| SwinTranformer_tiny  | 95.06 | 95.30  | 111 | 使用 ImageNet 预训练模型 |
| MobileNetV3_small_x0_35  | 86.67 | 2.85  | 2.6 | 使用 ImageNet 预训练模型 |
| PPLCNet_x1_0  | 93.64 | 2.13  | 7.0 | 使用 ImageNet 预训练模型 |
| PPLCNet_x1_0  | 94.44 | 2.13  | 7.0 | 使用 SSLD 预训练模型 |
| PPLCNet_x1_0  | 94.62 | 2.13  | 7.0 | 使用 SSLD 预训练模型+EDA 策略|
| <b>PPLCNet_x1_0<b>  | <b>94.94<b> | <b>2.13<b>  | <b>7.0<b> | 使用 SSLD 预训练模型+EDA 策略+SKL-UGI 知识蒸馏策略|

从表中可以看出，backbone 为 SwinTranformer_tiny 时精度较高，但是推理速度较慢。将 backbone 替换为轻量级模型 MobileNetV3_small_x0_35 后，速度可以大幅提升，但是会导致精度大幅下降。将 backbone 替换为速度更快的 PPLCNet_x1_0 时，精度较 MobileNetV3_small_x0_35 高 7 个百分点，与此同时速度依旧可以快 20% 以上。在此基础上，使用 SSLD 预训练模型后，在不改变推理速度的前提下，精度可以提升约 0.8 个百分点，进一步地，当融合EDA策略后，精度可以再提升 0.2 个百分点，最后，在使用 SKL-UGI 知识蒸馏后，精度可以继续提升 0.3 个百分点。此时，PPLCNet_x1_0 达到了 SwinTranformer_tiny 模型的精度，但是速度快 40 多倍。关于 PULC 的训练方法和推理部署方法将在下面详细介绍。

**备注：**

* 延时是基于 Intel(R) Xeon(R) Gold 6148 CPU @ 2.40GHz 测试得到，开启 MKLDNN 加速策略，线程数为10。
* 关于PP-LCNet的介绍可以参考[PP-LCNet介绍](../ImageNet1k/PP-LCNet.md)，相关论文可以查阅[PP-LCNet paper](https://arxiv.org/abs/2109.15099)。


<a name="2"></a>

## 2. 模型快速体验

<a name="2.1"></a>  

### 2.1 安装 paddlepaddle

- 您的机器安装的是 CUDA9 或 CUDA10，请运行以下命令安装

```bash
python3 -m pip install paddlepaddle-gpu -i https://mirror.baidu.com/pypi/simple
```

- 您的机器是CPU，请运行以下命令安装

```bash
python3 -m pip install paddlepaddle -i https://mirror.baidu.com/pypi/simple
```

更多的版本需求，请参照[飞桨官网安装文档](https://www.paddlepaddle.org.cn/install/quick)中的说明进行操作。

<a name="2.2"></a>  

### 2.2 安装 paddleclas

使用如下命令快速安装 paddleclas

```  
pip3 install paddleclas
```

<a name="2.3"></a>

### 2.3 预测

点击[这里](https://paddleclas.bj.bcebos.com/data/PULC/pulc_demo_imgs.zip)下载 demo 数据并解压，然后在终端中切换到相应目录。

* 使用命令行快速预测

```bash
paddleclas --model_name=code_exists --infer_imgs=pulc_demo_imgs/code_exists/contains_code_demo.jpg
```

结果如下：
```
>>> result
class_ids: [1], scores: [0.9955421453341842], label_names: ['contains_code'], filename: pulc_demo_imgs/code_exists/contains_code_demo.jpg
Predict complete!
```

**备注**： 更换其他预测的数据时，只需要改变 `--infer_imgs=xx` 中的字段即可，支持传入整个文件夹。


* 在 Python 代码中预测
```python
import paddleclas
model = paddleclas.PaddleClas(model_name="code_exists")
result = model.predict(input_data="pulc_demo_imgs/code_exists/contains_code_demo.jpg")
print(next(result))
```

**备注**：`model.predict()` 为可迭代对象（`generator`），因此需要使用 `next()` 函数或 `for` 循环对其迭代调用。每次调用将以 `batch_size` 为单位进行一次预测，并返回预测结果, 默认 `batch_size` 为 1，如果需要更改 `batch_size`，实例化模型时，需要指定 `batch_size`，如 `model = paddleclas.PaddleClas(model_name="code_exists",  batch_size=2)`, 使用默认的代码返回结果示例如下：

```
>>> result
[{'class_ids': [1], 'scores': [0.9955421453341842], 'label_names': ['contains_code'], 'filename': 'pulc_demo_imgs/code_exists/contains_code_demo.jpg'}]
```

<a name="3"></a>

## 3. 模型训练、评估和预测

<a name="3.1"></a>  

### 3.1 环境配置

* 安装：请先参考文档[环境准备](../../installation.md) 配置 PaddleClas 运行环境。

<a name="3.2"></a>

### 3.2 数据准备

<a name="3.2.1"></a>

#### 3.2.1 数据集来源

由于目前没有二维码/小程序码/条形码的公开数据集，所以第一步，我们需要收集可能带广告码的数据。收集方式可以写一个简单的爬取数据的脚本，收集好数据之后，我们需要进行一个简单的标注，即标注图像中是否含有广告码。

<a name="3.2.2"></a>  

#### 3.2.2 数据集获取

当数据集收集好之后，可以适当处理成 PaddleClas 的要求的数据格式。处理后的图片如下图所示：

筛选出的图片如下图所示：
<center><img src='https://user-images.githubusercontent.com/45199522/197466050-d4a1cf60-85f0-4ffe-9cad-14dbfc197d41.png' width=800></center>

其中，0 表示图像没有广告码，1 表示图中有广告码。

此处提供了经过上述方法处理好的数据的子集，可以直接下载得到。


进入 PaddleClas 目录。

```
cd path_to_PaddleClas
```

进入 `dataset/` 目录，下载并解压有无广告码场景的数据。

```shell
cd dataset
wget https://paddleclas.bj.bcebos.com/data/PULC/code_exists.tar
tar -xf code_exists.tar
cd ../
```

执行上述命令后，`dataset/` 下存在 `code_exists` 目录，该目录中具有以下数据：

```
├── train
│   ├── 0
│   │   ├── 10021.jpg
│   │   ├── 10038.jpg
│   │   ├── ...
│   └── 1
│       ├── 10016.jpg
│       ├── 10028.jpg
│       ├── ...
├── train_list.txt
├── val
│   ├── 0
│   │   ├── 10077.jpg
│   │   ├── 10184.jpg
│   │   ├── ...
│   └── 1
│       ├── 10091.jpg
│       ├── 10098.jpg
│       ├── ...
└── val_list.txt
```

其中 `train/` 和 `val/` 分别为训练集和验证集。`train_list.txt` 和 `val_list.txt` 分别为训练集和验证集的标签文件。
**备注：**

* 关于 `train_list.txt`、`val_list.txt`的格式说明，可以参考 [PaddleClas 分类数据集格式说明](../../training/single_label_classification/dataset.md#1-数据集格式说明) 。

* 关于如何得到蒸馏的标签文件可以参考[知识蒸馏标签获得方法](../../training/advanced/ssld.md#3.2)。


<a name="3.3"></a>

### 3.3 模型训练


在 `ppcls/configs/PULC/code_exists/PPLCNet_x1_0.yaml` 中提供了基于该场景的训练配置，可以通过如下脚本启动训练：

```shell
export CUDA_VISIBLE_DEVICES=0,1,2,3
python3 -m paddle.distributed.launch \
    --gpus="0,1,2,3" \
    tools/train.py \
        -c ./ppcls/configs/PULC/code_exists/PPLCNet_x1_0.yaml
```

验证集的最佳指标在 `0.93-0.94` 之间（数据集较小，容易造成波动）。

**备注：** 由于此处使用的是全量数据的子集，所以指标会略低，下同。本案例提供的预训练模型和 inference 模型是经过全量数据训练得到的，您可以直接下载使用。

<a name="3.4"></a>

### 3.4 模型评估

训练好模型之后，可以通过以下命令实现对模型指标的评估。

```bash
python3 tools/eval.py \
    -c ./ppcls/configs/PULC/code_exists/PPLCNet_x1_0.yaml \
    -o Global.pretrained_model="output/PPLCNet_x1_0/best_model"
```

其中 `-o Global.pretrained_model="output/PPLCNet_x1_0/best_model"` 指定了当前最佳权重所在的路径，如果指定其他权重，只需替换对应的路径即可。

<a name="3.5"></a>

### 3.5 模型预测

模型训练完成之后，可以加载训练得到的预训练模型，进行模型预测。在模型库的 `tools/infer.py` 中提供了完整的示例，只需执行下述命令即可完成模型预测：

```python
python3 tools/infer.py \
    -c ./ppcls/configs/PULC/code_exists/PPLCNet_x1_0.yaml \
    -o Global.pretrained_model=output/PPLCNet_x1_0/best_model
```

输出结果如下：

```
[{'class_ids': [1], 'scores': [0.9999976], 'label_names': ['contains_code'], 'file_name': 'deploy/images/PULC/code_exists/contains_code_demo.jpg'}]
```

**备注：**

* 这里`-o Global.pretrained_model="output/PPLCNet_x1_0/best_model"` 指定了当前最佳权重所在的路径，如果指定其他权重，只需替换对应的路径即可。

* 默认是对 `deploy/images/PULC/code_exists/contains_code_demo.jpg` 进行预测，此处也可以通过增加字段 `-o Infer.infer_imgs=xxx` 对其他图片预测。


<a name="4"></a>

## 4. 模型压缩

<a name="4.1"></a>

### 4.1 SKL-UGI 知识蒸馏

SKL-UGI 知识蒸馏是 PaddleClas 提出的一种简单有效的知识蒸馏方法，关于该方法的介绍，可以参考[SKL-UGI 知识蒸馏](../../training/advanced/ssld.md)。

<a name="4.1.1"></a>

#### 4.1.1 教师模型训练

复用 `ppcls/configs/PULC/code_exists/PPLCNet/PPLCNet_x1_0.yaml` 中的超参数，训练教师模型，训练脚本如下：

```shell
export CUDA_VISIBLE_DEVICES=0,1,2,3
python3 -m paddle.distributed.launch \
    --gpus="0,1,2,3" \
    tools/train.py \
        -c ./ppcls/configs/PULC/code_exists/PPLCNet_x1_0.yaml \
        -o Arch.name=ResNet101_vd
```

验证集的最佳指标为 `0.95-0.96` 之间，当前教师模型最好的权重保存在 `output/ResNet101_vd/best_model.pdparams`。

<a name="4.1.2"></a>

####  4.1.2 蒸馏训练

配置文件`ppcls/configs/PULC/code_exists/PPLCNet_x1_0_distillation.yaml`提供了`SKL-UGI知识蒸馏策略`的配置。该配置将`ResNet101_vd`当作教师模型，`PPLCNet_x1_0`当作学生模型，使用ImageNet数据集的验证集作为新增的无标签数据。训练脚本如下：

```shell
export CUDA_VISIBLE_DEVICES=0,1,2,3
python3 -m paddle.distributed.launch \
    --gpus="0,1,2,3" \
    tools/train.py \
        -c ./ppcls/configs/PULC/code_exists/PPLCNet_x1_0_distillation.yaml \
        -o Arch.models.0.Teacher.pretrained=output/ResNet101_vd/best_model
```

验证集的最佳指标为 `0.945-0.955` 之间，当前模型最好的权重保存在 `output/DistillationModel/best_model_student.pdparams`。


<a name="5"></a>

## 5. 超参搜索

在 [3.3 节](#3.3)和 [4.1 节](#4.1)所使用的超参数是根据 PaddleClas 提供的 `超参数搜索策略` 搜索得到的，如果希望在自己的数据集上得到更好的结果，可以参考[超参数搜索策略](../../training/PULC.md#4-超参搜索)来获得更好的训练超参数。

**备注：** 此部分内容是可选内容，搜索过程需要较长的时间，您可以根据自己的硬件情况来选择执行。如果没有更换数据集，可以忽略此节内容。

<a name="6"></a>

## 6. 模型推理部署

<a name="6.1"></a>

### 6.1 推理模型准备

Paddle Inference 是飞桨的原生推理库， 作用于服务器端和云端，提供高性能的推理能力。相比于直接基于预训练模型进行预测，Paddle Inference可使用 MKLDNN、CUDNN、TensorRT 进行预测加速，从而实现更优的推理性能。更多关于 Paddle Inference 推理引擎的介绍，可以参考 [Paddle Inference官网教程](https://www.paddlepaddle.org.cn/documentation/docs/zh/guides/infer/inference/inference_cn.html)。

当使用 Paddle Inference 推理时，加载的模型类型为 inference 模型。本案例提供了两种获得 inference 模型的方法，如果希望得到和文档相同的结果，请选择[直接下载 inference 模型](#6.1.2)的方式。

<a name="6.1.1"></a>

### 6.1.1 基于训练得到的权重导出 inference 模型

此处，我们提供了将权重和模型转换的脚本，执行该脚本可以得到对应的 inference 模型：

```bash
python3 tools/export_model.py \
    -c ./ppcls/configs/PULC/code_exists/PPLCNet_x1_0.yaml \
    -o Global.pretrained_model=output/DistillationModel/best_model_student \
    -o Global.save_inference_dir=deploy/models/PPLCNet_x1_0_code_exists_infer
```
执行完该脚本后会在 `deploy/models/` 下生成 `PPLCNet_x1_0_code_exists_infer` 文件夹，`models` 文件夹下应有如下文件结构：

```
├── PPLCNet_x1_0_code_exists_infer
│   ├── inference.pdiparams
│   ├── inference.pdiparams.info
│   └── inference.pdmodel
```

**备注：** 此处的最佳权重是经过知识蒸馏后的权重路径，如果没有执行知识蒸馏的步骤，最佳模型保存在`output/PPLCNet_x1_0/best_model.pdparams`中。

<a name="6.1.2"></a>

### 6.1.2 直接下载 inference 模型

[6.1.1 小节](#6.1.1)提供了导出 inference 模型的方法，此处也提供了该场景可以下载的 inference 模型，可以直接下载体验。

```
cd deploy/models
# 下载 inference 模型并解压
wget https://paddleclas.bj.bcebos.com/models/PULC/code_exists_infer.tar && tar -xf code_exists_infer.tar
```

解压完毕后，`models` 文件夹下应有如下文件结构：

```
├── code_exists_infer
│   ├── inference.pdiparams
│   ├── inference.pdiparams.info
│   └── inference.pdmodel
```

<a name="6.2"></a>

### 6.2 基于 Python 预测引擎推理


<a name="6.2.1"></a>  

#### 6.2.1 预测单张图像

返回 `deploy` 目录：

```
cd ../
```

运行下面的命令，对图像 `./images/PULC/code_exists/contains_code_demo.jpg` 进行有无广告码分类。

```shell
# 使用下面的命令使用 GPU 进行预测
python3.7 python/predict_cls.py -c configs/PULC/code_exists/inference_code_exists.yaml
# 使用下面的命令使用 CPU 进行预测
python3.7 python/predict_cls.py -c configs/PULC/code_exists/inference_code_exists.yaml -o Global.use_gpu=False
```

输出结果如下。

```
contains_code.jpg:	class id(s): [1], score(s): [1.00], label_name(s): ['contains_code']
```

<a name="6.2.2"></a>  

#### 6.2.2 基于文件夹的批量预测

如果希望预测文件夹内的图像，可以直接修改配置文件中的 `Global.infer_imgs` 字段，也可以通过下面的 `-o` 参数修改对应的配置。

```shell
# 使用下面的命令使用 GPU 进行预测，如果希望使用 CPU 预测，可以在命令后面添加 -o Global.use_gpu=False
python3.7 python/predict_cls.py -c configs/PULC/code_exists/inference_code_exists.yaml -o Global.infer_imgs="./images/PULC/code_exists/"
```

终端中会输出该文件夹内所有图像的分类结果，如下所示。

```
no_code_demo.jpg:	class id(s): [0], score(s): [1.00], label_name(s): ['no_code']
contains_code_demo.jpg:	class id(s): [1], score(s): [1.00], label_name(s): ['contains_code']
```

其中，`contains_code` 表示该图里存在广告码，`no-code` 表示该图里不存在广告码。

<a name="6.3"></a>

### 6.3 基于 C++ 预测引擎推理

PaddleClas 提供了基于 C++ 预测引擎推理的示例，您可以参考[服务器端 C++ 预测](../../deployment/image_classification/cpp/linux.md)来完成相应的推理部署。如果您使用的是 Windows 平台，可以参考[基于 Visual Studio 2019 Community CMake 编译指南](../../deployment/image_classification/cpp/windows.md)完成相应的预测库编译和模型预测工作。

<a name="6.4"></a>

### 6.4 服务化部署

Paddle Serving 提供高性能、灵活易用的工业级在线推理服务。Paddle Serving 支持 RESTful、gRPC、bRPC 等多种协议，提供多种异构硬件和多种操作系统环境下推理解决方案。更多关于Paddle Serving 的介绍，可以参考[Paddle Serving 代码仓库](https://github.com/PaddlePaddle/Serving)。

PaddleClas 提供了基于 Paddle Serving 来完成模型服务化部署的示例，您可以参考[模型服务化部署](../../deployment/image_classification/paddle_serving.md)来完成相应的部署工作。

<a name="6.5"></a>

### 6.5 端侧部署

Paddle Lite 是一个高性能、轻量级、灵活性强且易于扩展的深度学习推理框架，定位于支持包括移动端、嵌入式以及服务器端在内的多硬件平台。更多关于 Paddle Lite 的介绍，可以参考[Paddle Lite 代码仓库](https://github.com/PaddlePaddle/Paddle-Lite)。

PaddleClas 提供了基于 Paddle Lite 来完成模型端侧部署的示例，您可以参考[端侧部署](../../deployment/image_classification/paddle_lite.md)来完成相应的部署工作。

<a name="6.6"></a>

### 6.6 Paddle2ONNX 模型转换与预测

Paddle2ONNX 支持将 PaddlePaddle 模型格式转化到 ONNX 模型格式。通过 ONNX 可以完成将 Paddle 模型到多种推理引擎的部署，包括TensorRT/OpenVINO/MNN/TNN/NCNN，以及其它对 ONNX 开源格式进行支持的推理引擎或硬件。更多关于 Paddle2ONNX 的介绍，可以参考[Paddle2ONNX 代码仓库](https://github.com/PaddlePaddle/Paddle2ONNX)。

PaddleClas 提供了基于 Paddle2ONNX 来完成 inference 模型转换 ONNX 模型并作推理预测的示例，您可以参考[Paddle2ONNX 模型转换与预测](../../deployment/image_classification/paddle2onnx.md)来完成相应的部署工作。
