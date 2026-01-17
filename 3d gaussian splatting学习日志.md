# 3d gaussian splatting学习日志

## 1.高斯库中conver.py的作用

`convert.py` 是一个**“自动化管家”**脚本。

它的核心作用是：**把一堆杂乱的 2D 照片，翻译成 3D Gaussian Splatting (3DGS) 能听懂的 3D 语言。**

如果没有这个脚本，你需要手动输入十几行复杂的 COLMAP 命令，还得小心翼翼地整理文件结构。这个脚本把这一切都自动化了。

具体来说，它按顺序干了以下 **4 件大事**：

### 1. 特征提取与匹配 (Feature Extraction & Matching)

- **代码对应**：`feature_extractor` 和 `exhaustive_matcher`
- **在做什么**：它在每张照片里找“特征点”（比如卡车的车灯角、地上的石头缝），然后把所有照片里相同的点连起来。
- **通俗理解**：玩“连连看”。它告诉电脑：“第一张照片里的这个点，和第二张照片里的那个点，是同一个东西。”

### 2. 稀疏重建 (Structure from Motion / SfM)

- **代码对应**：`mapper`
- **在做什么**：这是最核心的数学计算。通过上面找到的共同点，利用三角测量原理，反推出**每一张照片拍摄时相机的确切位置（位姿）**，以及物体在空间中的**稀疏 3D 点云**。
- **结果**：生成了 `sparse` 文件夹，里面存着相机的坐标和一堆稀疏的 3D 点。

### 3. 图像去畸变 (Image Undistortion)

- **代码对应**：`image_undistorter`
- **在做什么**：真实世界的相机镜头是有畸变的（比如广角镜头的边缘会弯曲）。3DGS 的算法假设光线是直线传播的（针孔相机模型）。
- **作用**：这个步骤把刚才算好的照片进行拉伸和矫正，把弯曲的线条变直，生成“完美的”照片供训练使用。

### 4. 数据整理 (File Organization)

- **代码对应**：最后的 `shutil.move` 等 Python 代码
- **在做什么**：把生成好的数据搬运到标准的文件夹结构里（比如 `images`, `sparse/0`）。
- **为什么**：因为接下来的 `train.py` 很“死板”，它只认这种固定的文件夹结构，如果不整理好，训练脚本就会报错找不到文件。

------

### 💡 总结

**`convert.py` 的输入：**

- 一堆照片 (`input` 文件夹)

**`convert.py` 的输出：**

1. **cameras.json**：所有相机的参数。
2. **images.json**：每一张照片的拍摄位置和旋转角度。
3. **points3D.json**：一个稀疏的 3D 点云（作为训练的初始种子）。
4. **images 文件夹**：去畸变后的照片。

这就是为什么跑完这个脚本后，您就可以直接运行 `python train.py` 了，因为它已经把所有“食材”都切好备齐了。

## 2.colmap重建全过程指南

如果您的 `python convert.py ...` 命令已经跑完并且没有报错（最后显示 `Undistorting images... Done.`），那么恭喜您，**稀疏重建（Sparse Reconstruction）已经自动完成了！**

现在您有两个选择：

1. **直接开始 3DGS 训练**（这是最快的，直接生成 3D 高斯模型）。
2. **继续跑 COLMAP 的稠密重建**（生成传统的致密点云，看起来更像实体模型，适合测试 COLMAP 效果）。

既然您之前提到想看 COLMAP 的效果，我带您用 **RTX 5090** 跑一次完整的**稠密重建 (Dense Reconstruction)**。这一步非常吃显卡，正好能发挥您 5090 的威力。

请**依次执行**下面的命令：

------

### 第一步：图像去畸变 (Image Undistortion)

COLMAP 需要先把照片里的畸变（鱼眼效应等）拉直，才能进行深度计算。

Bash

```
# 1. 创建存放稠密重建的文件夹
mkdir -p /root/autodl-tmp/data/truck/tandt/truck/dense

# 2. 运行去畸变 (这一步很快)
colmap image_undistorter \
    --image_path /root/autodl-tmp/data/truck/tandt/truck/input \
    --input_path /root/autodl-tmp/data/truck/tandt/truck/distorted/sparse/0 \
    --output_path /root/autodl-tmp/data/truck/tandt/truck/dense \
    --output_type COLMAP \
    --max_image_size 2000
```

------

### 第二步：深度图计算 (Patch Match Stereo) —— **5090 的主场**

这一步通过对比不同照片的像素来计算深度。它是最慢的一步，但对于 5090 来说就是“洒洒水”。

Bash

```
colmap patch_match_stereo \
    --workspace_path /root/autodl-tmp/data/truck/tandt/truck/dense \
    --workspace_format COLMAP \
    --PatchMatchStereo.geom_consistency true
```

- **观察**：您会看到显卡在高速运转。如果是普通显卡可能要跑几十分钟，5090 应该非常快。

------

### 第三步：点云融合 (Stereo Fusion)

把上面算出来的所有深度图融合成一个巨大的点云文件。

Bash

```
colmap stereo_fusion \
    --workspace_path /root/autodl-tmp/data/truck/tandt/truck/dense \
    --workspace_format COLMAP \
    --input_type geometric \
    --output_path /root/autodl-tmp/data/truck/tandt/truck/dense/fused.ply
```

------

### 第四步：下载并查看结果

运行完上述命令后，您会在 `/root/autodl-tmp/data/truck/tandt/truck/dense/` 目录下得到一个名为 **`fused.ply`** 的文件。

1. **下载**：请把这个 `fused.ply` 文件下载到您的 Mac 上（文件可能几百 MB）。
2. **查看**：
   - 推荐使用网页版查看器（无需安装软件）：[PlayCanvas SuperSplat](https://playcanvas.com/supersplat/editor) 或 [Spline](https://spline.design/)。
   - 或者下载 Mac 版的 **MeshLab** 软件打开。

您将看到的效果：

相比于之前的稀疏点（只有几千个点），这个稠密点云会有几百万个点，卡车的车身细节、地面的纹理都会非常清晰地呈现出来，就像一个雕塑一样。

------

### 💡 跑完这个之后呢？

看爽了 COLMAP 的效果后，别忘了我们要做的正事是 **3D Gaussian Splatting**。

回到项目根目录，直接开始训练：

Bash

```
cd ~/tf-logs/gaussian-splatting
python train.py -s /root/autodl-tmp/data/truck/tandt/truck
```

5090 跑这个训练只需要几分钟，生成的模型效果通常比 COLMAP 的点云更逼真（因为带有光影和半透明效果）。

----------------------------------------------------------------------------------------------------------------









## 3.github管理代码

简单来说，**Repo** 是 **Repository**（仓库）的缩写。

你可以把它想象成**云端的项目文件夹**，但它比普通文件夹更强大。

### 1. 什么是 Repo (仓库)？

- **不仅仅是文件：** 它不仅存放你的代码、图片和文档，还记录了**每一次的修改历史**（谁在什么时候改了哪一行代码）。
- **协作中心：** 它允许你和他人共同协作同一个项目，而不会把文件搞乱。
- **公开 vs 私有：** 你可以把它设为 Public（所有人可见，开源项目）或 Private（仅自己可见）。
- ![git workflow diagram的图片](https://encrypted-tbn2.gstatic.com/licensed-image?q=tbn:ANd9GcRCshdgXxxFhCO_AH1iRz3kFi9fPfof3EZ5VSQsbg1jWn30bKydT7m1UN-n61d9llxMewcQLB0sAMZNscpPNrCKIm0YG6J1tAClWDRZRjJKi8y6f6E)Shutterstock

------

### 2. 如何上传代码到 GitHub (最通用方法)

对于开发者（特别是像你这样使用 Python 环境的用户），最标准、最稳定的方式是使用 **命令行 (Git Bash / Terminal)**。

假设你的电脑上已经安装了 Git，且在 GitHub 网站上注册了账号，请按以下步骤操作：

#### 第一步：在 GitHub 网页上建一个空仓库

1. 登录 GitHub，点击右上角的 **+** 号，选择 **New repository**。
2. **Repository name**：起个名字（比如 `my-project`）。
3. **注意**：为了避免冲突，**不要**勾选 "Add a README file" 或 "Add .gitignore"（我们稍后在本地添加）。
4. 点击 **Create repository**。

#### 第二步：在本地准备你的代码

打开你的终端（Terminal 或 Cmd），进入你的项目文件夹：

Bash

```
cd /path/to/your/project  # 把这里换成你实际的文件夹路径
```

#### 第三步：初始化并提交 (三板斧)

依次运行以下命令：

1. **初始化仓库** (让 Git 开始管理这个文件夹)：

   Bash

   ```
   git init
   ```

2. **添加文件** (把所有文件放入“暂存区”)：

   Bash

   ```
   git add .
   ```

   *(注意 `add` 后面有个点，代表当前目录所有文件)*

3. **提交更改** (把文件存入本地历史记录)：

   Bash

   ```
   git commit -m "第一次提交代码"
   ```

#### 第四步：推送到 GitHub (连接云端)

回到你刚才在 GitHub 创建的网页，你会看到一段代码提示。复制并在终端运行类似下面的命令：

1. **关联远程仓库** (替换成你自己的地址)：

   Bash

   ```
   git remote add origin https://github.com/你的用户名/你的仓库名.git
   ```

2. **推送到云端**：

   Bash

   ```
   git push -u origin main
   ```

*注：如果是第一次使用，终端可能会提示你输入 GitHub 的用户名和密码（现在通常需要使用 Personal Access Token 而不是登录密码）。*

------

### 3. 极简方法 (适合只有几个小文件)

如果你不想用命令行，且文件很少：

1. 在 GitHub 上创建仓库时，勾选 **"Add a README file"**。
2. 创建好后，点击页面上的 **Add file** -> **Upload files**。
3. 直接把你的文件拖拽进去，然后在下方填写 Commit changes (提交说明)，点击绿色按钮即可。

### 给你的特别建议 (关于 Python)

因为你经常处理 Python/环境配置，上传前请务必检查目录里有没有 **`.gitignore`** 文件。你需要用它来**忽略**不需要上传的大文件或临时文件，例如：

- 虚拟环境文件夹 (`env/`, `venv/`, `anaconda3/`)
- `__pycache__/`
- 大型数据集 (`.csv`, 图片集等)

下一步：

如果你的文件比较多，或者包含敏感配置（如 API Key），需要我帮你写一个适合 Python 项目的 .gitignore 模板吗？

----------------------------------------------------------------------------------------------------------------