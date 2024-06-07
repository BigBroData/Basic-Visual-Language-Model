# 从零搭建自己的多模态大模型

For the English version of the README, please refer to [README.md](README.md).

## 代码说明 💻

- **数据预处理**：相关代码位于 `dataprocess` 文件夹下，数据集相关代码在 `dataset` 文件夹中。数据预处理主要包括路径合并、QA 数据拼接、特征插入 token 处理等。
- **LLM模型**：使用 Qwen-7B 作为主体，相关代码在 `qwen` 文件夹中。通过重写 `QWenModel` 的 `forward` 方法，实现多模态特征的注入。
- **视觉模型**：使用 `CLIP_VIT` 和 `SIGLIP_VIT`，相关代码在 `visual` 文件夹中，其中还包含其他主干网络。
- **VLM模型**：相关代码在 `model` 文件夹下的 `model.py` 文件中。

## 数据集 🌏

我们使用了多语言数据集，主要包括 COCO2017 数据集和 AI Challenger 图像中文描述数据集：
- COCO 数据集的标注使用了 LLAVA 的 `detail_23k` 和 `complex_reasoning_77k`，这些标注可以有效提升模型的描述丰富度。
- AI Challenger 数据集使用原始标注，并使用固定的 prompt。

## 模型架构 🤖

在 VLM 中，视觉部分采用已经实现初步语义对齐的 `CLIP` 或 `SIGLIP` 模型，并使用两层 MLP 进行特征映射。通过重写 `QWenModel` 的 `forward` 方法，将对应的 `image` 标记替换为视觉特征。

如果你希望替换模型架构，请修改[这部分](https://github.com/xinyanghuang7/Basic-Vision-Language-Model/blob/main/train.py#L41)。

## 如何开始部署 🔧

### 下载相关数据

| AI Challenger | COCO | complex_reasoning_77k.json | detail_23k.json |
| --- | --- | --- | --- |
| [AI Challenger](https://tianchi.aliyun.com/dataset/145781) | [COCO 2017](http://images.cocodataset.org/zips/train2017.zip) | [complex_reasoning_77k.json](https://huggingface.co/datasets/liuhaotian/LLaVA-Instruct-150K/resolve/main/complex_reasoning_77k.json) | [detail_23k.json](https://huggingface.co/datasets/liuhaotian/LLaVA-Instruct-150K/resolve/main/detail_23k.json) |

请按照[配置文件](https://github.com/xinyanghuang7/Basic-Vision-Language-Model/blob/main/dataprocess/config.yaml)中的路径存放数据集。当然，路径可以自定义。

请注意，此路径需要与[data/](https://github.com/xinyanghuang7/Basic-Vision-Language-Model/blob/main/train.py#L29)保持一致，以便模型进行读取。

数据下载完毕后，使用 `process_image.py` 进行预处理。

### 安装运行环境

使用 `pip install` 安装 `requirements.txt`：

```shell
pip install -r requirements.txt
```

### 开始训练

模型训练采用 image model 冻结的方式进行，LLM 使用 Lora 方式训练以减少训练压力。需要训练的参数包括视觉特征映射层以及 LLM 中 Lora 的参数。由于映射层是未训练的初始化参数，为了平衡模型参数优化速度，这里为映射层设定了比 Lora 部分更大的学习率。

运行根目录的 `train.sh`，可自行配置相关参数进行试验。

```shell
sh train.sh
```

通过上述步骤，您可以启动训练过程并进行多模态模型的训练。

模型权重将会保存在`--output_dir`中，同样，这个路径可以进行自定义。

#### `train.sh` 脚本解析

```sh
CUDA_VISIBLE_DEVICES=0 torchrun --nproc_per_node=1 --master_port=25642 train.py \
    --lora_rank 128 \
    --lora_dropout 0.10 \
    --per_device_train_batch_size 4 \
    --gradient_accumulation_steps 1 \
    --num_train_epochs 2 \
    --save_steps 1000 \
    --save_total_limit 5 \
    --learning_rate 3e-5 \
    --seed 42 \
    --ddp_find_unused_parameters False \
    --feature_proj_lr 1e-4 \
    --remove_unused_columns false \
    --logging_steps 100 \
    --output_dir ./weights/train_V1_5 \
    --target_modules "c_attn|w1|w2" \
    --image_map /home/u2023111315/Basic-Vision-Language-Model/data/image_map_b.json \
    --captions_file /home/u2023111315/Basic-Vision-Language-Model/data/captions_b.json
```

#### 解释

1. **CUDA_VISIBLE_DEVICES=0**: 使用ID为0的GPU。
2. **torchrun**: PyTorch的分布式训练工具。
3. **--nproc_per_node=1**: 每个节点运行1个进程。
4. **--master_port=25642**: 设置进程间通信端口。
5. **train.py**: 主训练脚本。

#### 传递给 `train.py` 的参数

1. **--lora_rank 128**: LoRA层的秩为128。
2. **--lora_dropout 0.10**: LoRA层的dropout率为10%。
3. **--per_device_train_batch_size 4**: 每个设备的训练批次大小为4。
4. **--gradient_accumulation_steps 1**: 梯度累积步数为1。
5. **--num_train_epochs 2**: 训练2个epoch。
6. **--save_steps 1000**: 每1000步保存一次模型。
7. **--save_total_limit 5**: 最多保存5个检查点。
8. **--learning_rate 3e-5**: 学习率为3e-5。
9. **--seed 42**: 随机种子为42。
10. **--ddp_find_unused_parameters False**: 禁用DDP查找未使用的参数。
11. **--feature_proj_lr 1e-4**: 特征投影层的学习率为1e-4。
12. **--remove_unused_columns false**: 保留未使用的列。
13. **--logging_steps 100**: 每100步记录一次日志。
14. **--output_dir ./weights/train_V1_5**: 输出目录。
15. **--target_modules "c_attn|w1|w2"**: LoRA适配的目标模块。
16. **--image_map /home/u2023111315/Basic-Vision-Language-Model/data/image_map_b.json**: 图像映射文件路径。
17. **--captions_file /home/u2023111315/Basic-Vision-Language-Model/data/captions_b.json**: 标注文件路径。

### 测试模型 

运行根目录的 `test.sh`，可自行配置相关参数进行试验。

```shell
sh test.sh
```

代码会读取文件夹下的图片进行问答。

#### 预训练模型

如果你只希望进行测试，可以使用以下预训练模型文件进行加载，将其放入`--model_weights`所指定的路径中即可。

| 模型1 | 模型2 |
| --- | --- |
| [预训练7000轮](https://huggingface.co/xinyanghuang/Basic-Visual-Language-Model/tree/main/checkpoint-7000/checkpoint-7000) | TODO |

#### `test.sh` 脚本解析

```sh
python test.py --base_language_model Qwen/Qwen-7B-Chat --base_value_model openai/clip-vit-large-patch14 --model_weights ./weights/train_V1_5/checkpoint-10000/ --image_path ./test_img/1.jpg --prompt "使用语言描述一下图中出现了那些颜色<|extra_0|>"
```

#### 传递给 `test.py` 的参数

1. **--base_language_model Qwen/Qwen-7B-Chat**: 指定基础语言模型的路径，这里使用的是 `Qwen/Qwen-7B-Chat`。
2. **--base_value_model openai/clip-vit-large-patch14**: 指定基础视觉模型的路径，这里使用的是 `openai/clip-vit-large-patch14`。
3. **--model_weights ./weights/train_V1_5/checkpoint-10000/**: 指定模型权重的路径，这里使用的是训练过程中保存的检查点 `checkpoint-10000`。
4. **--image_path ./test_img/1.jpg**: 指定输入图像的路径，这里使用的是 `./test_img/1.jpg`。
5. **--prompt "使用语言描述一下图中出现了那些颜色<|extra_0|>"**: 指定模型的提示语，这里要求模型用语言描述图中出现的颜色。

## 参考 📚

感谢以下项目的伟大工作🙌：

- https://github.com/WatchTower-Liu/VLM-learning/tree/main
- https://github.com/QwenLM/Qwen
- https://github.com/haotian-liu/LLaVA

## 联系 ✉

如果你有任何疑问或者想法，十分欢迎随时联系我😊：

hsinyanghuang7@gmail.com

我会在看到邮件的第一时间回复！
