# GR-ConvNet笔记

仓库链接：https://github.com/skumra/robotic-grasping

环境配置和GGCNN通用，不配了

## 1.下载数据集

跳过，ggcnn下过了

## 2.克隆代码

```
git clone https://github.com/skumra/robotic-grasping.git
```

## 3.模型训练

可以使用 `train_network.py` 脚本训练模型。运行 `train_network.py --help` 以查看选项的完整列表。

Example for Cornell dataset:
康奈尔大学数据集示例：

```
python train_network.py --dataset cornell --dataset-path /home/wjj/GG-CNN/ggcnn/archive --description training_cornell
```

```
python train_network.py --dataset cornell --dataset-path /home/wjj4090/Datasets/cornell --description training_cornell
```

Example for Jacquard dataset:
提花数据集示例：

```
python train_network.py --dataset jacquard --dataset-path <Path To Dataset> --description training_jacquard --use-dropout 0 --input-size 300
```

## 4.模型评估
可以使用 `evaluate.py` 脚本评估训练的网络。运行 `evaluate.py --help` 以获取全套选项。

Example for Cornell dataset:
康奈尔大学数据集示例：

使用前新建一个results文件夹用来存放图片：

运行后马上ctrl+c,不然弹出一堆图片

```
python evaluate.py --network /home/wjj/GR-ConvNet/robotic-grasping/trained-models/cornell-randsplit-rgbd-grconvnet3-drop1-ch16/epoch_30_iou_0.97 --dataset cornell --dataset-path /home/wjj/GG-CNN/ggcnn/archive --iou-eval --vis
```

<img src="../assests/GR-ConvNet笔记/image-20251120111814193.png" alt="image-20251120111814193" style="zoom:33%;" />

Example for Jacquard dataset:
提花数据集示例：

```
python evaluate.py --network <Path to Trained Network> --dataset jacquard --dataset-path <Path to Dataset> --iou-eval --use-dropout 0 --input-size 300
```



# 改造GR-convnet-wjj：

## 1.修改数据集的划分

把jacquard从使用整个数据集改成了seen和unseen的划分方式

训练指令：

```
python train_network.py --dataset jacquard --dataset-path /home/wjj4090/Datasets/Jacquard --descripti
on training_jacquard --use-dropout 0 --input-size 224 --seen 1
```

## 2.加入对graspanything数据集的支持

从LGD代码里面把grasp_anything_data.py和language_grasp_data.py复制过来

```
python train_network.py --dataset grasp-anything --dataset-path /home/wjj4090/Datasets/Grasp-Anything --description training_graspanything --use-dropout 0 --use-depth 0 --input-size 224
```



# 1.SE-ResUNet

https://github.com/BIT-robot-group/SE-ResUNet

cornell:98.2 jacquard:95.7

```
python train_network.py --dataset cornell --dataset-path /home/wjj4090/Datasets/Jacquard --description training_cornell
```

