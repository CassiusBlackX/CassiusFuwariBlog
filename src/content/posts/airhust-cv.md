---
title: airhust视觉追踪入门
published: 2024-09-21
description: '华中科技大学airhust无人机视觉追踪入门新生实践课v1'
image: ''
tags: ["opencv", "ros"]
category: 'deployment'
draft: false 
lang: ''
---

# 简介
## 任务简介
## yolo简介
## 提前准备
1. conda环境
2. python基础语法
3. tmux的基本用法

# 云端环境部署
## darknet-yolo, pjreddie版本 
### 克隆和编译darknet仓库
```shell
git clone https://github.com/pjreddie/darknet
cd darknet
```
> 知识点补充： `git`是一个版本管理器。一些常用的git命令是必须要能够掌握的。

进入到`darknet`文件夹以后，可以看到文件夹的结构如下
```plaintext
/darknet
    cfg/ 模型参数配置文件
    data/ 训练相关参数
    examples/
    include/
    python/
    scripts/
    src/
    LICENSE*
    Makefile  工程项目构建文件
    README.md  最重要，必须认真阅读！
```
#### README.md
一般来说，在上手一个github项目的时候，必须先阅读的是这个项目中的`README.md`文件，该文件会告诉你本项目中的一些基础、必要的说明，甚至还有一些情况下会直接告诉你如何快速开始、如何跑起本项目的一个测试用例等。

#### Makefile
`make`是一个工程项目构建文件。实际作用类似`cmake`。当然，这里其实倒反天罡了，正确的描述应该是cmake像make，因为make比cmake早很多。现在cmake使用的会比make更频繁一些，也是因为make的语法比较抽象，不是特别容易写和读。相比之下cmake就会简单很多。当然，如果仔细观察过cmake之后的build文件夹的话，会发现里面其实仍然是有Makefile的。

在这里我们必须要去修改Makefile文件，不然如果使用默认编译的话是无法在较快的时间内训练出一个能够使用的模型的。
```Makefile
GPU=0
CUDNN=0
OPENCV=0
OPENMP=0
DEBUG=0
```
先来看前面几行，这里就是编译时候的简单配置。0就是不使用，1就是使用。在以上几个参数中，`GPU`是必须要设置成为1的，否则如果使用CPU的话，又烧电脑，又训不出来有用的东西。

`CUDNN`是nvidia开发的一个深度学习加速库，理论上来说可以加速训练的速度，当然，这里不是必须的。但是如果感兴趣可以自己尝试去为服务器配置cudnn。

`OPENCV`是目前最广泛使用的计算机视觉库，在后续的视觉相关任务中是必然绕不开的一部分。但是在这里不一定需要打开，这里使用opencv最主要的目的一般是提高darknet对于数据集图片格式的兼容性，但是在绝大多数情况下是不会遇到非常古怪格式的数据集的。

接下来还必须修改`ARCH`参数。这个参数是和你使用的设备是强相关的。你使用的是什么显卡，在这里就应该使用该显卡架构对应的参数。如果遇到不确定的，就根据显卡的名称去网上查。

在以上参数都改好了以后，就可以直接编译了，在这里我们可是直接使用命令`make`就可以了。需要注意的是，使用make命令的前提是Makefile文件必须和运行make指令时候所在的路径一致，否则make命令会找不到Makefile的

#### cfg&data
接下来修改模型相关的配置文件，进入到文件夹`cfg/`下。首先我们要改参数`batch`。batch参数是一次性进行训练、推理的图片数量，这个参数如果太大可能会超出GPU的显存，但是适当大一些可以提高训练速度。至于其它参数一般来说也不需要修改，当然，也可以参照着网上的一些解说，适当的自己尝试着去修改。

然后最重要的就是修改classes参数。我们需要明确的告诉yolo，我们希望训练的模型是希望能够检测多少个类别的模型，对应的参数名称是`yolo/classes`，但是仅修改classes是不够的，还需要把和classes强相关的`convolutional/filters`参数给修改，对应公式是
$$ filters = 3 \times ( 5 + len(classes)) $$
并且应当注意，在yolov3.cfg中，这样的参数有三组。

然后我们需要自己写一个属于我们本次项目的.data文件，具体的写法可以参照着`cfg/`下的其它.data文件来写。
```plaintext
classes = ${num_classes}
train = ${path-to-trainlist.txt}
valid = ${path-to-vallist.txt}
backup = ${path-to-backup-dir}
names = ${path-to-.names-file}
```
最后，我们还需要写一个适合我们比赛项目中会用到的.names文件。对应着我们每一个类的名字。在这里需要非常注意的是.names文件中每个类的名字和对应的行数，在后面会影响我们的数据集的标注。

### 数据集的处理
一个完整的“数据集”，其实应该同时包括图片和标签，这样才能够训练。

#### labelimg
labelimg是一款数据集标注工具。我们需要使用pip来安装。但是同时因为labelimg已经比较具有“年代感”了，所以对于太新版本的python不是特别适应。这就是我们需要conda的时候了。conda可以方便的创建多个python虚拟环境，每个虚拟环境之间互相不影响，在我们当前的需求下非常适用。
```shell
conda create -n labelimg python=3.6
```
以上命令可以帮助我们创建一个名为`labelimg`的虚拟环境，根据命令提示，
```shell
conda activate labelimg
```
然后使用命令
```shell
pip install labelImg
```
最后使用`lablimg`启动。

#### xml格式的标签转yolo格式的标签
以下代码能够把VOC格式的标签转换为YOLO格式的标签，具体细节根据需要自行修改
```python
import os
import xml.etree.ElementTree as ET

def convert(size, box):
    dw = 1. / size[0]
    dh = 1. / size[1]
    x = (box[0] + box[1]) / 2.0
    y = (box[2] + box[3]) / 2.0
    w = box[1] - box[0]
    h = box[3] - box[2]
    x = x * dw
    w = w * dw
    y = y * dh
    h = h * dh
    return (x, y, w, h)


def convert_annotation(input_list, output_path, classes):
    for xml_file in input_list:
        image_id = os.path.splitext(os.path.basename(xml_file))[0]
        out_file = open(os.path.join(output_path, image_id + '.txt'), 'w')
        tree = ET.parse(xml_file)
        root = tree.getroot()
        size = root.find('size')
        w = int(size.find('width').text)
        h = int(size.find('height').text)

        for obj in root.iter('object'):
            difficult = obj.find('difficult').text
            cls = obj.find('name').text
            if cls not in classes or int(difficult) == 1:
                continue
            cls_id = classes.index(cls)
            xmlbox = obj.find('bndbox')
            b = (float(xmlbox.find('xmin').text), float(xmlbox.find('xmax').text), float(xmlbox.find('ymin').text),
                 float(xmlbox.find('ymax').text))
            bb = convert((w, h), b)
            out_file.write(str(cls_id) + " " + " ".join([str(a) for a in bb]) + '\n')

classes = []  # the classes name
images_path = "../dataset5/images"
xml_path = "../dataset5/labels_xml"
tar_images_path = "../dataset5/images"
tar_label_path = "../dataset5/labels_txt"
if not os.path.exists(tar_images_path):
    os.makedirs(tar_images_path)
if not os.path.exists(tar_label_path):
    os.makedirs(tar_label_path)

xml_ids = [os.path.splitext(f)[0] for f in os.listdir(xml_path) if f.endswith('.xml')]
image_ids = [os.path.splitext(f)[0] for f in os.listdir(images_path)
                if f.endswith('.jpg') or f.endswith('JPG') or f.endswith('jpeg') or f.endswith('JPEG')]
assert xml_ids == image_ids  # if xml_ids != immage_ids, than annotations are not complete!

xmls_path = [os.path.join(xml_path, f) for f in os.listdir(xml_path) if f.endswith('.xml')]

convert_annotation(xmls_path, tar_label_path, classes)

images_path = [os.path.join(images_path, f) for f in os.listdir(images_path)
                if f.endswith('.jpg') or f.endswith('JPG') or f.endswith('jpeg') or f.endswith('JPEG')]

# for image in images_path:
#     shutil.copy(image, tar_images_path)
```
#### 划分train/val
```python
import os
import shutil
import random

def copy_images_and_labels(src_path, dst_path):
    os.makedirs(f'../{dst_path}/images', exist_ok=True)
    os.makedirs(f'../{dst_path}/labels', exist_ok=True)

    for dir in ['train', 'val']:
        os.makedirs(f'../{dst_path}/images/' + dir, exist_ok=True)
        os.makedirs(f'../{dst_path}/labels/' + dir, exist_ok=True)
    
    for image_id in image_ids:
        shutil.copy(f"../{src_path}/images/{image_id}.jpg",
                    f"../{dst_path}/images/train/{image_id}.jpg")
        shutil.copy(f"../{src_path}/labels/{image_id}.txt",
                    f"../{dst_path}/labels/train/{image_id}.txt")
    
    for image_id in val_ids:
        shutil.copy(f"../{src_path}/images/{image_id}.jpg",
                    f"../{dst_path}/images/val/{image_id}.jpg")
        shutil.copy(f"../{src_path}/labels/{image_id}.txt",
                    f"../{dst_path}/labels/val/{image_id}.txt")

src_path = "dataset_all"
dst_path = "dataset6"
image_ids = [os.path.splitext(f)[0] for f in os.listdir(f'../{src_path}/labels') if
                f.endswith('.txt')]

# Shuffle the image ids and split into train and val
random.shuffle(image_ids)
split = int(0.1 * len(image_ids))
val_ids = image_ids[:split]
train_ids = image_ids[split:]
copy_images_and_labels(src_path, dst_path)
```
#### 生成trainlist.txt & vallist.txt
```python
import os

# 指定需要遍历的文件夹
folder_path = '../dataset_darknet/images/'

# 打开文件，准备写入
with open('trainlist.txt', 'w') as file:
    # 遍历文件夹
    for root, dirs, files in os.walk(folder_path):
        # 遍历所有文件
        for name in files:
            # 检查文件是否为.jpg文件
            if name.endswith(".jpg"):
                # 获取文件的绝对路径
                abs_path = os.path.abspath(os.path.join(root, name))
                # 写入文件
                file.write(abs_path + '\n')
```

### 训练
#### 获取预训练权重
```shell
wget https://pjreddie.com/media/files/darknet53.conv.74
```
#### 开始训练
使用tmux新开一个窗口，然后运行
```shell
./darknet detector train cfg/${your-.data-file} cfg/${your-cfg-file} darknet53.conv.74
```

## [darknet-hank.ai版本](https://github.com/hank-ai/darknet)
### 克隆hank.ai版本darknet
```shell
git clone https://github.com/hank-ai/darknet
cd darknet
```
进入到工程项目中，首先阅读README.md和CMakeLists.txt，查看项目说明以及如何构建项目，发现项目要求cmake版本大于等于3.24。使用`cmake --version`查看当前cmake版本，如果版本过低，需要使用`snap`或者`apt`更新cmake。

### 编译darknet
为了能够加速训练，我们有必要使用cuda，因此首先检查本机环境是否有cuda，并且是否与nvidia驱动匹配。
```shell
nvidia-smi
# 使用这个命令查看驱动对应的cuda版本
nvcc -V
# 使用这个命令查看cuda版本
```
如果找不到nvcc，则应该需要考虑`$PATH`中是不是没有包含cuda环境变量；同时最好检查一下是否设置`LD_LIBRARY_PATH`或者在`/etc/ld.so.conf`下配置动态链接库路径。

当确定cuda存在以后，先新建目录`build`用来存放编译时候产生的各种文件，并进入到目录中，依次使用`cmake`,`make`编译项目
```shell
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j4 package
```
阅读CMakeLists我们可以发现，该工程项目的cmake不是为了创建一个可执行文件，而是为了CPack

在编译成功之后，应该能够在`build`文件夹中找到一个`darknet-<INSERT-VERSION-YOU-BUILT-HERE>.deb`的文件，是一个debian系统的软件安装包。使用命令
```shell
sudo dpkg -i darknet-<INSERT-VERSION-YOU-BUILT-HERE>.deb
```
来安装编译生成的安装包。

安装之后，查找是否存在文件`/usr/bin/darknet`，如果不存在，则一定说明没有安装成功。

### 数据集的处理
关于labelimg和VOC格式转YOLO格式详见[darknet-yolo, pjreddie版本](#数据集的处理)

### 训练
#### 获取预训练权重
根据项目说明，到 https://github.com/hank-ai/darknet/releases/download/v2.0/yolov3.weights 处下载预训练权重。





## yolov5
yolov5是一个比darknet要新很多的一个仓库,可以从仓库的更新时间就可以发现.一般来说,yolov5相比darknet可以有更快的推理速度和更高的精度.

当然,实际上yolo到了这个地步以后,已经不仅仅有yolov5了,同时还有yolov7,yolov8,yolox等等版本.但是这些版本使用的都是python,而且在使用上来说和yolov5区别不大,所以本次我们只详细介绍yolov5

### 部署
从github上克隆[yolov5](https://github.com/ultralytics/yolov5.git)

同之前提到过的一样.当克隆下来一个仓库以后,仍然非常建议从README.md开始入手这个仓库.但是在这里我们就不仔细阅读了.我们主要辨析一下yolov5和darknet的区别.

首先,因为yolov5几乎完全都是使用python构成的,所以这里我们最好创建一个conda虚拟环境,专门用来做yolo相关的东西.

创建conda虚拟环境
```shell
conda create -n yolo python=3.10
```
安装虚拟环境yolo所需要的相关依赖
```shell
pip install -r requirements.txt
```

### 数据集的处理
所有yolo使用的数据集的格式都是一样的.但是不同的是darknet在训练的时候需要使用`trainlist.txt`来找到所有的训练图片和对应标注,在yolov5中,需要使用的是yaml格式的文件.

数据集中的图片和标签的摆放需要使用符合yolov5的默认规范,否则就得自己手动去改源码,这样是比较麻烦的,但是也不至于不推荐.多看看大佬的代码也是好的.

```yaml
path: /path/to/dataset
train: images/train
val: images/val

names: 
  0: ${your_classes}
  1: ${your_classes}
  2: ${your_classes}
  3: ${your_classes}
nc: ${num_classes}
```

在训练yolo的时候,需要自己写一个如上格式的yaml文件,用来告诉yolo训练数据集的基本信息.

其中,`path`就是数据集的路径,数据集路径下的文件树必须要是以下形式的

```
/dataset
    /images
        /train
        /val
    /labels
        /train
        /val
```
并且要保证images中的图片和labels中的标签是互相对应的.train和val中的图片、标签也要是对应的才可以.

### 训练
yolov5的训练不需要自己手动去下载预训练权重文件,在训练的时候yolov5会自动检测是否有预训练权重文件,当然也可以做到在已经训练过的模型上继续训练.

训练的一般命令是如下格式的
```shell
python train.py --data /path/to/mydata.yaml --epochs 1000 --weights ' ' --cfg yolov5n.yaml --batch-size 32
```

+ `data`后面就是要跟着自己写的yaml文件
+ `epochs`就是训练轮数.这里设置一个比较大的值也挺好的,因为yolo自己会根据学习率、损失率来自动判断什么时候停止,防止模型过拟合,也可以防止浪费一些不必要的时间.
+ `weights`是权重文件的路径,如果是使用官方的预训练权重文件的话这里就什么都不用填.
+ `cfg`后面跟这都是对模型的细致描述.yolov5整体上来说虽然是yolo的一大版本,但是内部还是有一些不同的模型,具体可以查看README.md里面的说明.一般来说更大的模型会有更好的精度,但是不可避免的推理时间也会更长.需要注意的是,出了在`mydata.yaml`中指定了模型的目标推理类型数量之后,还得渠道对应的`yolov5n.yaml`中的对应位置也要修改推理类型数量.
+ `batch-size`就是每次同时训练多少张图片,只要不要让显存爆炸就可以了


# 端侧部署
## darknet_ros
端侧部署，darknet_ros的部署应该是根据着脚本文件一步一步修改过来。

首先，在`darknet_ros.launch`中，可以看到两个非常显眼的两个文件名，`ros.yaml`和`yolov2-tiny.yaml`。那么让我们进入到两个文件分别一探究竟。
### `ros.yaml`
`ros.yaml`中，我们可以发现是一些`darknet_ros`程序本身相关的参数设置。在没有特殊情况下不要动这个文件。

### `yolov2-tiny.yaml`
`yolov2-tiny.yaml`就是我们需要修改的文件了。

在其中可以看到又有两个值得注意的文件名。cfg文件和weights文件。

联想到我们之前在云端训练darknet_ros的时候，我们就修改过了cfg文件夹中的关于模型参数的文件，所以这里我们应当把我们当时在云端部署模型时候训练对应的cfg文件复制到端侧设备上，并把yaml文件中的config_file的name改为我们自己使用的文件名。

同理，weights文件就是darknet训练出来的权重文件，我们也应当做和cfg文件相似的操作。

参数`threshold`是用来调整yolo的置信度的。当yolo对于一个框的可信度低于`threshold`的时候，yolo将不会显示这个框，也不会发布这个框的任何消息，减少干扰。这个参数的调整应当结合比赛的实际情况。如果在一个较插的情况下，应该宁可自己在后续的程序中多手动过滤一些不必要的信息，也要防止直接识别不出来。

最后，names部分也应当直接复制之前在云端训练时候的names文件部分的内容。

这里需要提醒大家注意的是，不管你在云侧上面获得了怎么样的训练效果，或者是你使用`darknet detect`命令测试了多少张图片，大家都应当始终要把模型落实到开发板上来。根据我们过往的经验总结，在云侧使用`darknet detect`命令获得的效果和在开发板上实际的效果通常是不太一样的。