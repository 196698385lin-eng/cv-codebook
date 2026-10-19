# 我的常用代码字典（模板版）

更新：2026-09-19（新增模板 25–29：CNN、数据增强、分拣流程、YOLO 训练与推理）

更新记录：9/17 第一版（模板 1–18）→ 9/17 晚加入机器学习（19–22）→ 9/18 加入神经网络与 PyTorch（23–24）→ 9/19 加入 CNN、数据增强、分拣流程、YOLO（25–29）。以后新模板从 30 号接着排，旧编号不动，这样课程文档里引用“模板 15”永远指的是同一个东西。

用法：每个模板都标出了要改的位置（①图片路径、②颜色条件、③保存文件名）。把 ① 换成你的图片路径，按需要改 ②③，直接从上到下复制运行即可。

抄代码三步走：跟着敲一遍 → 故意改一个数字（阈值、尺寸、k）看结果怎么变 → 把有用的新代码加进这本字典。改动之后最好把结果图存下来，方便以后对比。

## 数据流总图（各组分是怎么连起来的）

① 输入层：模板 1、2  —— 读图/裁剪/灰度，把图变成标准格式。

② 看得见：模板 3、11、12、13、14 —— 颜色分割（RGB 或 HSV）、去噪、边缘、形态学。这一层的输出只有一个东西：干净的 0/255 掩码。

③ 量得准：模板 4、5、7、8、9、16 —— 掩码 → 轮廓 → 面积/形状/中心/角度。这一层输出“每个物体的几何描述”。

④ 对得上：模板 30 —— 像素坐标 → 毫米/机械臂坐标（标定）。

⑤ 认得出：模板 17、18、19–26、28、29 —— 模板匹配、特征匹配、机器学习、神经网络、CNN、增强、YOLO。这一层与②③是“并行路线”，用来处理颜色法搞不定的场景。

⑥ 做决定：模板 27 —— 把几何信息和类别合成抓取顺序、目标筐、指令表。

⑦ 动起来：模板 31 —— 逆运动学 + 动作序列，仿真或真机执行。

一句话：②③是“眼睛和尺子”，④是“翻译”，⑤是“备用的更强眼睛”，⑥⑦是“手”。任何一环出错，后面全错——所以顺序调试：先用模板 3 看掩码、再用 4 看轮廓、再用 16 看中心、最后才接线。

## 冗余与性能：哪些步骤可以砍（都带实测数字）

| 可以砍掉的步骤 | 实测依据 | 怎么砍 |
| --- | --- | --- |
| 中值滤波 与 形态学开运算 二选一 | 噪点图：中值滤波后轮廓 2125 → 4；再加开运算仍是 4 | 只留中值滤波，省一次全图遍历 |
| 有颜色掩码时不需要 Canny | Canny 不滤波 24765 个边缘、滤波后 3808，但都不如直接分割干净 | 颜色法能分时删掉 Canny 整段 |
| 模板匹配 与 ORB 二选一 | 同款方块模板匹配 1.000，但旋转 15° 就掉到 0.674；ORB 反而更稳 | 先问“目标会不会转”：不会转用 17，会转用 18 |
| 批量任务不必保存中间图 | 写盘是批量流程的主要耗时 | 只要统计数字时，plt.imsave 那行注掉 |
| 先裁 ROI 再处理 | 像素数减少后，后面每一步耗时线性下降 | 在模板 1 之后打印 shape 确认范围，再裁 |
| 小数据别硬上大模型 | 手写数字：kNN 98.70% > 小 CNN 97.96%；MLP 8 个神经元只有 92.59% | 先跑模板 22 的模型对比，谁好用谁 |
| 检测优先用预训练权重 | 30 张训练图：从零 mAP50 = 0.0000，预训练 = 0.7061 | 模板 28 里把 model 换成 yolov8n.pt |
| 训练轮数不必盲加 | CNN 20 轮 96.30%、60 轮 97.96%，再往后几乎不动 | 看模板 23/25 的曲线，平了就停 |

## 拿到一个新任务，按这 7 步走（每一步都能单独验证，别一口气全写）

第 1 步：先看清楚——用模板 1 读图，打印 shape，确认值域是 0~1 还是 0~255。

第 2 步：再分割——用模板 3（RGB）或 11（HSV）出掩码，把掩码存图看一眼，确认没漏没多。

第 3 步：清掩码——模板 12 去噪、14 开运算、9 面积过滤；每一步都打印轮廓数。

第 4 步：量几何——模板 4 找轮廓、16 求中心和角度、8 判形状；打印前 5 个物体的数字，肉眼对一下。

第 5 步：对坐标——模板 30 标定，把像素换成毫米；留几个点验证残差 <2~3 毫米。

第 6 步：做决定——模板 27 排顺序、定目标筐、导出 CSV。

第 7 步：执行——模板 31 仿真跑通，再换成真机指令；每次只改一个变量。

如果第 2 步就做不到（颜色分不开、物体是金属/透明），直接跳到模板 17/18/28 的“学习路线”。

## 正确性自检（这本字典里的代码都过了这两关）

① 语法自检：把字典里每个代码块抽出来用 Python 编译一遍，必须 0 个错误。② 冒烟测试：造一张全新的场景图，跑一遍模板 6 的流程，看能不能正确数出物体个数。

实测结果：31 个模板、33 个代码块，语法自检 0 个错误；冒烟测试用一张全新的 800×560 场景（5 个零件 + 1 个金属件 + 3 处噪声、背景还带渐变），结果是 5/5 全部正确识别、金属件被颜色过滤忽略、噪声被开运算+面积过滤清掉，平均定位误差 0.10 毫米、最大 0.26 毫米。测试脚本：scripts\字典自检_新图冒烟测试.py（每次改完字典都跑一遍它）。

## 选工具速查：要干什么，翻哪个模板

| 要干的事 | 用哪个模板 | 一句话提醒 |
| --- | --- | --- |
| 读一张图、看形状、保存 | 模板 1 | 每次任务都从它开始 |
| 变暗、转灰度、裁剪 | 模板 2 | 乘的系数越小越暗 |
| 只保留某一种颜色 | 模板 3 | RGB 是 0~1 小数阈值 |
| 找物体轮廓、量面积 | 模板 4 | 先显示 mask8 黑白图确认 |
| 给物体画定位框、求中心 | 模板 5 | 中心 = 左上角 + 宽高的一半 |
| 整个流程一条龙（最常用） | 模板 6 | 改 ①②③ 就能跑 |
| 求某个颜色的总面积 | 模板 7 | 数像素 or 轮廓面积求和 |
| 判断是圆还是方 | 模板 8 | 圆度≈0.785，方形≈1.0 |
| 去掉小噪点，只留大物体 | 模板 9 | 阈值取灰尘和目标的空档 |
| 没有测试图，自己画一张 | 模板 10 | PIL 画矩形和圆 |
| 光照变化大，RGB 不稳 | 模板 11 | HSV：H 管颜色，V 管明暗 |
| 图有噪点，掩码很脏 | 模板 12 | 中值滤波首选，核要小于最小物体 |
| 不知道颜色，想找边界 | 模板 13 | 先滤波再 Canny |
| 掩码有洞、边缘断线 | 模板 14 | 开运算去噪，闭运算补洞 |
| 一次处理很多张图 + 出表格 | 模板 15 | 结果名 = 原名 + _结果 |
| 物体倾斜，要朝向和真实长宽 | 模板 16 | minAreaRect 旋转框 |
| 零件外观位置都固定 | 模板 17 | 模板匹配，怕旋转缩放 |
| 目标会转、会缩放、有纹理 | 模板 18 | ORB，纯色物体不好用 |
| 让程序自己学（分类/识别） | 模板 19 | 切 → 训 → 测 → 打分 |
| 看清模型错在哪两类之间 | 模板 20 | 混淆矩阵，行真实列预测 |
| 训练一次，以后直接用 | 模板 21 | joblib 存模型 |
| 不知道用哪个模型，想比一比 | 模板 22 | 同一份数据，只换模型那一行 |
| 分界线是弯的（一层学不会） | 模板 23 | 加隐藏层：加权求和 + 激活 |
| 用 PyTorch 训练（工业界做法） | 模板 24 | 前向 → 损失 → 反向 → 更新 |
| 图片里的东西会平移、会转 | 模板 25、26 | CNN 卷积核滑动看；训练时随机转转再喂进去 |
| 要做“识别 + 定位 + 分拣决策” | 模板 27 | HSV→轮廓→形状→毫米→指令表 |
| 让模型自己找出物体在哪（检测） | 模板 28、29 | YOLO：训练 → 评估 → 推理 |

## 课程 ↔ 模板对照（复习时按课程找工具）

| 课程 | 主题 | 对应模板 |
| --- | --- | --- |
| L1–L3 | 环境、读图、显示、保存、裁剪、颜色提取 | 模板 1–3 |
| L4–L6 | 找轮廓、画定位框、量面积 | 模板 4–6 |
| 随时用 | 求某个颜色的总面积、自己造测试图 | 模板 7、10 |
| L7–L8 | 形状分类、面积过滤 | 模板 8、9 |
| L9–L10 | HSV 抗光变、滤波去噪 | 模板 11、12 |
| L11–L12 | Canny 边缘、膨胀腐蚀 | 模板 13、14 |
| L13 | 批量处理多张图片 + 出 CSV | 模板 15 |
| L14–L15 | 旋转框（中心/宽高/角度）、像素换算毫米 | 模板 16（配 15） |
| L16–L18 | 模板匹配、ORB 特征匹配 | 模板 17、18 |
| L19–L20 | 机器学习思想、sklearn 手写数字 | 模板 19–22 |
| L21–L22 | 神经网络、PyTorch 张量 | 模板 23、24 |
| L23–L24 | CNN 识别手写数字、数据增强 | 模板 25、26 |
| L26–L27 | YOLO 检测、自建数据集与调优 | 模板 28、29 |
| 项目一 | 桌面零件定位器（综合） | 模板 6 + 15 + 16 |
| 项目二 | 红绿零件智能分拣（视觉端） | 模板 27（配 11、16） |

## 第一组 · 读图和基础变换

### 模板 1：读图、显示、保存（每次任务的固定开头）

```python
from matplotlib import image as mpimg
import matplotlib.pyplot as plt
# ① 图片路径：改成你自己的图片文件名
img = mpimg.imread(r"D:\Codex\cv-learning\images\我的第一张图.png")
# 如果打印出的形状第 3 个数字是 4，就运行下一行去掉透明通道
img = img[:, :, :3]
print("图片形状：", img.shape)   # (高, 宽, 3)
# 显示
plt.imshow(img)
plt.axis("off")
plt.show()
# ② 保存：把 新名字.png 改成你想要的图片名
plt.imsave(r"D:\Codex\cv-learning\images\新名字.png", img)
print("保存成功！")
```

· 为什么这么写：matplotlib 读图是 0~1 浮点 RGB，OpenCV 要 0~255 整数，所以开头就要把值域定下来；[:, :, :3] 去掉 alpha 通道，否则后面 cv2 会报 4 通道错误。

· 干什么用的：每个任务的固定开头：把“图”变成后面所有算法都能吃的格式。

· 怎么优化：批量任务先 resize 再处理，速度能快几倍；不要每次 plt.show()，直接 imsave。

· 能省哪一步：只看结果时，imshow / axis / show 三行可以全删（省渲染时间）。

· 和谁配合：输出 img 给模板 2/3；输出 0~255 的 rgb 给模板 4/11/12。

· 下一步改进：换 cv2.imread + cvtColor(BGR2RGB) 读图更快，路径改用 pathlib 更稳。

### 模板 2：图片变换——变暗/变亮、灰度、裁剪

三个小工具，哪个需要就复制哪一段。

```python
# 变暗/变亮：乘的数字越小越暗
dark = img * 0.2      # 0.2 很暗，0.8 只是稍微暗一点
plt.imshow(dark); plt.axis("off"); plt.show()
# 彩色变黑白：把红绿蓝三个数平均成一个数
gray = img.mean(axis=2)
plt.imshow(gray, cmap="gray"); plt.axis("off"); plt.show()
# 裁剪：img[上下起点:上下终点, 左右起点:左右终点]
h, w = img.shape[:2]
left = img[:, : w // 2]                # 左半边
right_down = img[h // 2 :, w // 2 :]   # 右下角四分之一
middle = img[h // 4 : h * 3 // 4, w // 4 : w * 3 // 4]  # 中间一块
```

· 为什么这么写：灰度就是三通道求平均；裁剪是数组切片，返回的是视图，不复制数据，几乎不花时间。

· 干什么用的：造对照图、缩小处理范围（ROI）、数据增强的基础操作。

· 怎么优化：要快就用 cv2.cvtColor(rgb, COLOR_RGB2GRAY)（C 实现），比 img.mean(axis=2) 快。

· 能省哪一步：“只为看一眼”的中间图不用落盘；正式跑流程时整段可以删。

· 和谁配合：常当模板 1 的预处理：先裁 ROI，再喂给模板 3/4 做分割。

· 下一步改进：先 ROI 再算法是提速最直接的一招：像素越少，后面每一步都越快。

### 模板 3：只挑一种颜色（颜色提取）

```python
# 拆出三个颜色层
red = img[:, :, 0]
green = img[:, :, 1]
blue = img[:, :, 2]
# ② 颜色条件：数字是 0~1 的小数，调阈值时看颜色深浅
red_mask = (red > 0.5) & (green < 0.35) & (blue < 0.35)      # 红色物体
green_mask = (green > 0.4) & (red < 0.3) & (blue < 0.3)      # 绿色物体
# 只保留一种颜色：把不满足条件的像素涂黑
only_red = img.copy()
only_red[red_mask == False] = 0
plt.imshow(only_red); plt.axis("off"); plt.show()
# 数像素 ≈ 量面积
print("红色像素数量：", red_mask.sum())
print("绿色像素数量：", green_mask.sum())
```

· 为什么这么写：阈值 + 逻辑与生成布尔掩码；掩码求和就等于面积，因为每个 True 就是一个像素。

· 干什么用的：把纯色目标从背景里分出来，是最快的分割方法。

· 怎么优化：阈值别猜，先看直方图；掩码用 astype('uint8')*255 交给 OpenCV。

· 能省哪一步：如果后面只用掩码，不必生成 only_red 那张彩色图。

· 和谁配合：掩码喂模板 4（找轮廓）和模板 7（数面积）；和模板 11（HSV）二选一。

· 下一步改进：光照不稳时换 HSV；再不稳就加白平衡或自适应阈值。

## 第二组 · 找轮廓、画框、量面积

### 模板 4：OpenCV 找轮廓——描边 + 量面积

```python
import cv2
import numpy as np
# matplotlib 读的是 0~1 小数，OpenCV 喜欢 0~255 整数，先转一下
img_float = mpimg.imread(r"D:\Codex\cv-learning\images\我的第一张图.png")
rgb = (img_float[:, :, :3] * 255).astype("uint8")
r = rgb[:, :, 0].astype(int)
g = rgb[:, :, 1].astype(int)
b = rgb[:, :, 2].astype(int)
# ② 颜色条件：现在数值是 0~255，所以阈值要用 100、80 这种大数字
red_mask = (r > 150) & (g < 100) & (b < 100)
green_mask = (g > 100) & (r < 80) & (b < 80)
# 把 True/False 变成 OpenCV 喜欢的 255/0 黑白图
mask8 = ((red_mask | green_mask).astype("uint8")) * 255
# 找轮廓：不同 OpenCV 版本返回值不同，用这一行最稳
found = cv2.findContours(mask8, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
contours = found[0] if len(found) == 2 else found[1]
print("找到几个物体：", len(contours))
# 画轮廓：-1 = 全部画，(255,0,0) = 红色，3 = 线宽
outline = rgb.copy()
cv2.drawContours(outline, contours, -1, (255, 0, 0), 3)
plt.imshow(outline); plt.axis("off"); plt.show()
# 量面积（像素）
for c in contours:
    print("轮廓面积：", round(cv2.contourArea(c)))
```

· 为什么这么写：findContours 在二值图上沿边界跟踪；RETR_EXTERNAL 只取最外层，避免嵌套轮廓被重复计数。

· 干什么用的：把“掩码”变成“可测量的几何对象”（面积、位置、形状）。

· 怎么优化：先开运算再找轮廓：实测能把 2125 个假轮廓降到 4 个，后面所有步骤都跟着省力。

· 能省哪一步：只做统计时，画轮廓那三行可以删。

· 和谁配合：吃模板 3/11 的掩码；结果喂模板 5（画框）、8（形状）、9（面积过滤）、16（旋转框）。

· 下一步改进：用 CHAIN_APPROX_TC89 或 approxPolyDP 减少点数，轮廓多时更快。

### 模板 5：给每个物体画定位框（外接矩形）

```python
# 接着模板 4，已经拿到了 contours
boxes = []
for c in contours:
    x, y, w, h = cv2.boundingRect(c)
    boxes.append((x, y, w, h))
    # x,y 是左上角坐标，w,h 是宽高；想算中心点就加下面一行
    cx, cy = x + w // 2, y + h // 2
    print("物体：x =", x, " y =", y, " 宽 =", w, " 高 =", h, " 中心 =", (cx, cy))
# 在原图上画框
result = rgb.copy()
for x, y, w, h in boxes:
    cv2.rectangle(result, (x, y), (x + w, y + h), (255, 0, 0), 3)
# ③ 保存文件名
plt.imsave(r"D:\Codex\cv-learning\images\定位框结果.png", result)
print("定位框已保存！")
```

· 为什么这么写：boundingRect 给的是“横平竖直”的最小外接矩形，计算极快。

· 干什么用的：给每个物体一个位置和编号，做统计与可视化。

· 怎么优化：先按面积或位置排序再编号，结果才稳定可复现。

· 能省哪一步：只需要坐标时，boxes 列表可以不要，直接遍历 contours。

· 和谁配合：吃模板 4 的 contours；输出喂模板 15（批量统计）和模板 16（需要角度时）。

· 下一步改进：物体倾斜时必须换模板 16 的 minAreaRect，否则框会大一圈、中心也会偏。

### 模板 6：完整流程一条龙（最常用的套用模板）

把 ①图片路径、②颜色条件、③保存文件名改掉，其他地方不用动。

```python
import cv2
import matplotlib.pyplot as plt
from matplotlib import image as mpimg
# ① 图片路径
path = r"D:\Codex\cv-learning\images\我的第一张图.png"
img_float = mpimg.imread(path)
rgb = (img_float[:, :, :3] * 255).astype("uint8")
r = rgb[:, :, 0].astype(int)
g = rgb[:, :, 1].astype(int)
b = rgb[:, :, 2].astype(int)
# ② 颜色条件（想找什么颜色就改哪行）
red_mask = (r > 150) & (g < 100) & (b < 100)
green_mask = (g > 100) & (r < 80) & (b < 80)
all_mask = red_mask | green_mask          # | = 或者
mask8 = (all_mask.astype("uint8")) * 255
found = cv2.findContours(mask8, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
contours = found[0] if len(found) == 2 else found[1]
print("找到几个物体：", len(contours))
boxes = []
for c in contours:
    x, y, w, h = cv2.boundingRect(c)
    boxes.append((x, y, w, h))
    cx, cy = x + w // 2, y + h // 2
    print("面积 =", round(cv2.contourArea(c)), " 中心 =", (cx, cy))
result = rgb.copy()
for x, y, w, h in boxes:
    cv2.rectangle(result, (x, y), (x + w, y + h), (255, 0, 0), 3)
# ③ 保存文件名
plt.imsave(r"D:\Codex\cv-learning\images\我的结果.png", result)
print("完成，请打开 images 文件夹查看 我的结果.png")
```

· 为什么这么写：把“读图 → 掩码 → 去噪 → 轮廓 → 过滤 → 画框 → 保存”串成一条流水线，参数集中放在开头，改三处就能换任务。

· 干什么用的：新任务的默认入口：改 ①图片路径 ②颜色条件 ③保存名 就能用。

· 怎么优化：加上开运算和面积过滤，假轮廓从几千降到几个；把参数写成函数参数便于调。

· 能省哪一步：调试用的 print 和 imshow 在正式跑的时候可以全注释掉。

· 和谁配合：它是模板 1 + 3/11 + 14 + 4 + 9 + 5 的组合，本质是“把前几块拼起来”。

· 下一步改进：包成 locate(path) 函数返回零件列表，机械臂流程直接调用。

### 模板 7：求某一颜色的总面积（两种方法）

方法一：数像素就是量面积，最快。

```python
# 前提：上面已经算好了 red_mask、green_mask
# mask 里 True 的个数 = 这个颜色占的像素总数
print("红色总面积：", red_mask.sum())
print("绿色总面积：", green_mask.sum())
```

方法二：按颜色单独找轮廓，再把每个物体的面积加起来（可以顺便核对单个物体）。

```python
import cv2
# 每个颜色单独转成 255/0 黑白图
red_m8 = (red_mask.astype("uint8")) * 255
green_m8 = (green_mask.astype("uint8")) * 255
def total_area(mask8):
    found = cv2.findContours(mask8, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    contours = found[0] if len(found) == 2 else found[1]
    total = 0
    for c in contours:
        total = total + round(cv2.contourArea(c))
    return total
print("红色总面积：", total_area(red_m8))
print("绿色总面积：", total_area(green_m8))
```

两种方法结果会非常接近但不一定完全相同：像素数量是“点人头”，轮廓面积是几何算法。比较“谁更大”时用哪一种都可以。

· 为什么这么写：像素计数 = 目标占了多少个像素；轮廓面积求和 = 几何面积，两者接近但不等价。

· 干什么用的：统计产量、覆盖率、或某个颜色的总占比。

· 怎么优化：只要总量就用掩码求和（最快）；要单件面积才需要逐个轮廓算。

· 能省哪一步：两种方法不必都留在正式代码里，留一种即可（省一次遍历）。

· 和谁配合：依赖模板 3/11 的掩码。

· 下一步改进：乘比例尺平方得到毫米²，和模板 21 的标定配合使用。

## 第三组 · 判断形状、过滤噪点、自己造测试图

### 模板 8：形状分类（判断“圆”和“方”）

前提：已经拿到 contours（模板 4/6）。宽高比接近 1 是方形；圆度（充满度）约 0.785 是圆、约 1.0 是方形。

```python
for i, c in enumerate(contours):
    x, y, w, h = cv2.boundingRect(c)
    area = cv2.contourArea(c)
    ratio = w / h            # 宽高比：接近 1 是方形
    fill = area / (w * h)    # 圆度：圆约 0.785，方形约 1.0
    if fill < 0.85:
        shape = "近似圆"
    elif ratio > 1.3:
        shape = "近似长方形"
    else:
        shape = "近似正方"
    print("物体%d: 宽高比=%.2f 圆度=%.2f -> %s" % (i + 1, ratio, fill, shape))
```

· 为什么这么写：填充度 = 轮廓面积 / 外接矩形面积：圆 ≈ π/4 ≈ 0.785，方块 ≈ 0.98~0.99。

· 干什么用的：按形状分类，是分拣决策的直接依据。

· 怎么优化：阈值取 0.85 比 0.95 稳（实测方块只有 0.98~0.99）；用 minAreaRect 的宽高比更抗旋转。

· 能省哪一步：没有冗余步骤，但判据顺序有讲究：先判圆，再判长条。

· 和谁配合：吃模板 4/5 的结果，输出给综合项目的分拣决策。

· 下一步改进：再加圆度 4πA/P²（P 为周长）能更稳地区分圆和椭圆。

### 模板 9：面积过滤（去掉小噪点，只留大物体）

前提：已经拿到 contours。面积小于 min_area 的当噪声丢掉。

```python
min_area = 10000          # 先打印所有面积，再定这个数
kept = []
for c in contours:
    area = cv2.contourArea(c)
    if area < min_area:
        continue              # 跳过小噪点
    x, y, w, h = cv2.boundingRect(c)
    kept.append((x, y, w, h, area))
print("过滤前：", len(contours))
print("过滤后：", len(kept))
for item in kept:
    print("保留物体：", item)
biggest = max(contours, key=cv2.contourArea)
print("最大物体面积：", round(cv2.contourArea(biggest)))
```

min_area 怎么定：先打印所有轮廓面积，找“最大灰尘”和“最小目标物体”之间的空档，阈值取在空档中间；分不开时再加颜色、形状、位置判据。

· 为什么这么写：面积阈值把“灰尘”和“零件”分开，是最省事的去噪手段。

· 干什么用的：只保留真正的目标，避免噪声污染统计。

· 怎么优化：阈值取“最大噪点”和“最小目标”之间的空档中间；配合开运算效果最好。

· 能省哪一步：已经做了开运算时，形态学核可以小一号（3×3 够用），省时间。

· 和谁配合：吃模板 4 的 contours，输出给模板 5/8/16。

· 下一步改进：用面积分布自动定阈值（例如取中位数的 1/10），换场景时不必手改。

### 模板 10：自己画测试图（PIL）+ 形状验证完整脚本

用途：不会找测试图时，用 PIL 自己画一张白底图（长方形、正方形、圆都可以），再自动判断每个物体的形状。改颜色就改 fill，改大小就改中括号里的四个数字。

```python
from matplotlib import image as mpimg
from matplotlib import pyplot as plt
from PIL import Image, ImageDraw
import cv2
IMG = r"D:\Codex\cv-learning\images\练习_细长长方形.png"
OUT = r"D:\Codex\cv-learning\images\练习_细长长方形_结果.png"
# 1) 画测试图
img = Image.new("RGB", (700, 300), "white")          # 白纸：宽700 高300
d = ImageDraw.Draw(img)                               # 拿起画笔
d.rectangle([40, 60, 360, 120], fill=(255, 0, 0))     # 细长红色长方形
d.rectangle([450, 70, 550, 170], fill=(255, 0, 0))    # 红色正方形
# 想画圆就换成：d.ellipse([左, 上, 右, 下], fill=(0, 200, 0))
img.save(IMG)
# 2) 判断形状
img_float = mpimg.imread(IMG)
rgb = (img_float[:, :, :3] * 255).astype("uint8")
r = rgb[:, :, 0].astype(int)
g = rgb[:, :, 1].astype(int)
b = rgb[:, :, 2].astype(int)
red_mask = (r > 150) & (g < 100) & (b < 100)
mask8 = (red_mask.astype("uint8")) * 255
found = cv2.findContours(mask8, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
contours = found[0] if len(found) == 2 else found[1]
result = rgb.copy()
for i, c in enumerate(contours):
    x, y, w, h = cv2.boundingRect(c)
    area = cv2.contourArea(c)
    ratio = w / h            # 宽高比
    fill = area / (w * h)    # 圆度
    if fill < 0.85:
        shape = "近似圆"
    elif ratio > 1.3:
        shape = "近似长方形"
    else:
        shape = "近似正方"
    print("物体%d: 宽高比=%.2f 圆度=%.2f -> %s" % (i + 1, ratio, fill, shape))
    cv2.rectangle(result, (x, y), (x + w, y + h), (255, 0, 0), 3)
plt.imsave(OUT, result)
print("完成：", OUT)
```

画图形小抄：rectangle([左, 上, 右, 下]) 画长方形/正方形；ellipse([左, 上, 右, 下]) 画圆/椭圆；fill=(红, 绿, 蓝) 是颜色，范围 0~255。

· 为什么这么写：用 PIL 画图就是“用程序造数据”：形状、颜色、位置、角度都可控，还能顺便记下真值。

· 干什么用的：没有测试图时验证算法；也能造干净的“教科书数据”。

· 怎么优化：批量造图时固定随机种子，方便复现；直接在内存里画（不落盘）更快。

· 能省哪一步：单元测试阶段可以不保存图片，只留数组。

· 和谁配合：产出的图喂模板 1/6；记录的真值可以用来算定位误差。

· 下一步改进：加随机光照、噪声、模糊，当成数据增强的离线版。

## 第四组 · 抗光变、去噪、边缘、形态学

### 模板 11：HSV 颜色空间（挑颜色不怕明暗变化）

什么时候用：颜色阈值在不同光线/亮度下不稳定时，用 HSV 代替 RGB。H 管颜色种类、S 管浓淡、V 管明暗；变暗主要降 V，H 几乎不变。

```python
import cv2
from matplotlib import image as mpimg
img = mpimg.imread(path)
rgb = (img[:, :, :3] * 255).astype("uint8")
# matplotlib 读的是 RGB；若用 cv2.imread 读图，改成 COLOR_BGR2HSV
hsv = cv2.cvtColor(rgb, cv2.COLOR_RGB2HSV)
H = hsv[:, :, 0].astype(int)   # 0~179：红≈0/180，绿≈60，蓝≈120
S = hsv[:, :, 1].astype(int)   # 0~255：越大颜色越浓
V = hsv[:, :, 2].astype(int)   # 0~255：越大越亮
# 红色跨 0，所以要写两段
red_mask = ((H <= 10) | (H >= 170)) & (S > 100) & (V > 50)
green_mask = (H >= 40) & (H <= 85) & (S > 80) & (V > 50)
print("红色像素：", red_mask.sum())
print("绿色像素：", green_mask.sum())
```

选值方法：print(hsv[y, x]) 看已知物体的 HSV，H 上下放宽 ±10，S、V 设下限把白色背景（S 小）和黑影（V 小）排除。

· 为什么这么写：HSV 把“颜色种类（H）”和“明暗（V）”解耦，所以变暗时 H 几乎不动。

· 干什么用的：光照会变、RGB 阈值不稳时的首选分割方法。

· 怎么优化：先用 print(hsv[y, x]) 看已知物体的 HSV 再定阈值；红色跨 0 必须写两段。

· 能省哪一步：不需要整张 HSV 图长期保留，算完掩码即可释放，省内存。

· 和谁配合：替代模板 3，喂给模板 4/12。

· 下一步改进：前面加白平衡或 CLAHE，能进一步提升抗光变能力。

### 模板 12：滤波去噪（中值 / 高斯）

什么时候用：不加滤波时噪点会变成成百上千个小轮廓（实测 2125 个），滤波后只剩真正的物体（4 个）。核必须是奇数，而且必须明显小于最小的目标物体。

```python
import cv2
from matplotlib import image as mpimg
img = mpimg.imread(path)
rgb = (img[:, :, :3] * 255).astype("uint8")   # 滤波要用 0~255 整数图
# 中值滤波：去孤立小点（椒盐噪声），首选
clean = cv2.medianBlur(rgb, 5)                 # 核 3 / 5 / 7，越大越狠
# 高斯滤波：平滑渐变色噪点，但边缘会略糊（二选一）
# clean = cv2.GaussianBlur(rgb, (5, 5), 0)
# 然后照常做掩码（模板 3 或模板 11）→ 找轮廓（模板 4）
hsv = cv2.cvtColor(clean, cv2.COLOR_RGB2HSV)
H = hsv[:, :, 0].astype(int)
S = hsv[:, :, 1].astype(int)
V = hsv[:, :, 2].astype(int)
mask8 = ((((H <= 10) | (H >= 170)) & (S > 100) & (V > 50)) |
         ((H >= 40) & (H <= 85) & (S > 80) & (V > 50))).astype("uint8") * 255
found = cv2.findContours(mask8, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
contours = found[0] if len(found) == 2 else found[1]
print("找到轮廓数：", len(contours))
```

核大小规则：先量最小目标物体的宽度，核 ≤ 该宽度的 1/2～1/3（小螺丝很小时用 3）。若连 3 都伤到小物体，就不要整图滤波，改用“面积过滤”（模板 9）或在掩码上过滤。

· 为什么这么写：中值滤波用邻域中位数替换中心像素，专治椒盐噪声；高斯滤波更平滑但会糊边。

· 干什么用的：分割前降噪，避免噪声被打成目标。

· 怎么优化：核必须是奇数且明显小于最小目标；小目标多时别整图滤波。

· 能省哪一步：实测中值滤波 5 已经把轮廓从 2125 降到 4，此时再加开运算是重复劳动，那一步可以省（省一次全图遍历）。

· 和谁配合：吃模板 1/11 的图，输出喂模板 4/14。

· 下一步改进：要保边就换双边滤波；或者只在掩码上做形态学，代价更小。

### 模板 13：Canny 边缘检测（不靠颜色找轮廓）

什么时候用：不知道物体颜色、或需要找物体边界时。边缘 = 亮度突变处。阈值越低边缘越多（含噪点），越高边缘越少；一定要先去噪再 Canny。

```python
import cv2
import matplotlib.pyplot as plt
from matplotlib import image as mpimg
img = mpimg.imread(path)
rgb = (img[:, :, :3] * 255).astype("uint8")
gray = cv2.cvtColor(rgb, cv2.COLOR_RGB2GRAY)   # Canny 只看灰度
blur = cv2.GaussianBlur(gray, (5, 5), 0)        # 先滤波，减少假边缘
edges = cv2.Canny(blur, 50, 150)               # 低阈值, 高阈值
plt.imshow(edges, cmap="gray")
plt.axis("off")
plt.show()
print("边缘像素数：", int((edges > 0).sum()))
```

实测：干净图约 1900 个边缘；噪点图不滤波约 24700，滤波 50/150 约 3800。边缘常是断线，用模板 14 的膨胀/闭运算把断线接起来。

· 为什么这么写：Canny = 高斯平滑 + 梯度 + 非极大值抑制 + 双阈值滞后，输出细边缘。

· 干什么用的：不知道物体颜色、或需要边界轮廓时使用。

· 怎么优化：务必先滤波（实测边缘像素 24765 → 3808）；高低阈值比 2:1 或 3:1。

· 能省哪一步：已经有颜色掩码时，Canny 这一步可以整段删掉——颜色法更直接。

· 和谁配合：输出喂模板 14（接断线）和模板 4（找轮廓）。

· 下一步改进：用中位数自动定阈值的自适应 Canny，换光照不用手改参数。

### 模板 14：膨胀与腐蚀（形态学修掩码）

什么时候用：掩码有小噪点、小孔、断线时。开运算 = 先腐蚀（去小噪点）；闭运算 = 先膨胀（补小洞、接断线）。核要小于物体间距和最小目标物体。

```python
import cv2
import numpy as np
# mask8 是前面做好的 0/255 黑白掩码
kernel = np.ones((3, 3), np.uint8)              # 3x3 方块核，可用 5x5 更强
eroded  = cv2.erode(mask8, kernel, iterations=1)              # 腐蚀：白色变小
dilated = cv2.dilate(mask8, kernel, iterations=1)             # 膨胀：白色变大
opened  = cv2.morphologyEx(mask8, cv2.MORPH_OPEN,  kernel)    # 开运算：去小噪点
closed  = cv2.morphologyEx(mask8, cv2.MORPH_CLOSE, kernel)    # 闭运算：补洞/接断线
```

实测：噪点掩码原 2125 个轮廓，开运算 3x3 后只剩 4 个（四个积木）。圆形核写法：cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (5, 5))

· 为什么这么写：腐蚀让白区变小、膨胀让白区变大；开运算 = 先腐蚀后膨胀（去小噪点），闭运算 = 先膨胀后腐蚀（补小洞、接断线）。

· 干什么用的：修掩码三件事：去掉碎点、把断线接上、填掉小孔。

· 怎么优化：3×3 够用就别上 5×5；顺序是“先开运算去噪，再面积过滤兜底”。

· 能省哪一步：和中值滤波功能重叠，两个都做没有额外收益（实测都是 4 个轮廓），留一个即可。

· 和谁配合：吃模板 3/11/13 的输出，喂模板 4。

· 下一步改进：按噪声形状选核（随机噪点用椭圆核、横竖条纹用十字核）。

## 第五组 · 批量处理、旋转框、匹配定位

### 模板 15：批量处理多张图片（循环 + CSV 统计表）

什么时候用：一次实验拍了很多图，要统一处理、统一出数据表。输入输出分文件夹，原图不覆盖；结果名 = 原图名 + _结果；统计表用 utf-8-sig，Excel 打开不乱码。

```python
from pathlib import Path
import csv
import cv2
import numpy as np
from matplotlib import image as mpimg
from matplotlib import pyplot as plt
in_dir = Path(r"D:\Codex\cv-learning\images\batch_in")     # 原图文件夹
out_dir = Path(r"D:\Codex\cv-learning\images\batch_out")   # 结果文件夹
out_dir.mkdir(exist_ok=True)
rows = []
for path in sorted(in_dir.glob("*.png")):
    img = mpimg.imread(str(path))
    rgb = (img[:, :, :3] * 255).astype("uint8")
    hsv = cv2.cvtColor(rgb, cv2.COLOR_RGB2HSV)
    H = hsv[:, :, 0].astype(int)
    S = hsv[:, :, 1].astype(int)
    V = hsv[:, :, 2].astype(int)
    red_mask = ((H <= 10) | (H >= 170)) & (S > 100) & (V > 50)
    green_mask = (H >= 40) & (H <= 85) & (S > 80) & (V > 50)
    mask8 = ((red_mask | green_mask).astype("uint8")) * 255
    mask8 = cv2.morphologyEx(mask8, cv2.MORPH_OPEN, np.ones((3, 3), np.uint8))
    found = cv2.findContours(mask8, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    contours = found[0] if len(found) == 2 else found[1]
    big = [c for c in contours if cv2.contourArea(c) > 5000]
    areas = [round(cv2.contourArea(c)) for c in big]
    result = rgb.copy()
    for c in big:
        x, y, w, h = cv2.boundingRect(c)
        cv2.rectangle(result, (x, y), (x + w, y + h), (255, 0, 0), 3)
    plt.imsave(out_dir / (path.stem + "_结果.png"), result)
    rows.append({"文件名": path.name, "物体数": len(big),
                 "红色像素": int(red_mask.sum()), "绿色像素": int(green_mask.sum()),
                 "最大面积": max(areas, default=0), "总面积": sum(areas)})
    print(path.name, "->", len(big), "个物体", "面积", areas)
fields = ["文件名", "物体数", "红色像素", "绿色像素", "最大面积", "总面积"]
with open(r"D:\Codex\cv-learning\images\batch_results.csv", "w",
          newline="", encoding="utf-8-sig") as fh:
    writer = csv.DictWriter(fh, fieldnames=fields)
    writer.writeheader()
    writer.writerows(rows)
```

· 为什么这么写：遍历目录 + 统一处理 + 汇总 CSV，是“把一次实验跑完”的标准结构。

· 干什么用的：一次处理几十上百张图并出统计表。

· 怎么优化：输入输出分开目录、结果名 = 原名 + 后缀、CSV 用 utf-8-sig（Excel 不乱码）。

· 能省哪一步：只要统计数字时可以不保存结果图，能省掉一大半时间（写盘是瓶颈）。

· 和谁配合：内部循环体就是模板 6 的流程。

· 下一步改进：加多进程/多线程（计算密集用进程），再接 tqdm 显示进度。

### 模板 16：minAreaRect 旋转框（中心 + 真实宽高 + 角度）

什么时候用：物体可能倾斜、需要朝向或真实长宽时（机械臂夹取最常用）。boundingRect 是横平竖直的框；minAreaRect 是最小面积的旋转框。

```python
# c 是已经找好的轮廓（建议先按面积过滤）
bx, by, bw, bh = cv2.boundingRect(c)                    # 普通框
(cx, cy), (w, h), angle = cv2.minAreaRect(c)            # 旋转框
box = cv2.boxPoints(((cx, cy), (w, h), angle)).astype(np.intp)
print("普通框: 位置=(%d,%d) 宽高=(%d,%d)" % (bx, by, bw, bh))
print("旋转框: 中心=(%.1f,%.1f) 宽高=(%.1f,%.1f) 角度=%.1f" % (cx, cy, w, h, angle))
result = rgb.copy()
cv2.rectangle(result, (bx, by), (bx + bw, by + bh), (0, 200, 0), 3)  # 绿色普通框
cv2.drawContours(result, [box], 0, (0, 0, 255), 3)                   # 蓝色旋转框
```

实测：画 30 度、220x70 的长方形 → 普通框 224x170；旋转框 71.1x220.5，角度 -59.9。规律：输出角度 ≈ 倾斜角 - 90；倾斜角 ≈ 90 - abs(angle)。0 度和 90 度都会输出 -90，但宽高顺序会互换（w/h 顺序不固定）。注意：boxPoints 是小数，必须用 np.intp 转整数（np.int0 在新版 NumPy 已删除）。

· 为什么这么写：minAreaRect 用旋转卡壳求最小面积矩形，能给出真实宽高和朝向。

· 干什么用的：物体倾斜、需要中心/角度/真实长宽时使用（机械臂夹取最常要这个）。

· 怎么优化：先面积过滤再算，省算力；boxPoints 输出小数，必须 astype(np.intp)。

· 能省哪一步：不需要朝向时用 boundingRect 就够，更快。

· 和谁配合：吃模板 4/9 的轮廓；输出接分拣决策与夹爪姿态。

· 下一步改进：角度不稳时改用 PCA 主轴方向，或先筛掉细长形状再算角度。

### 模板 17：模板匹配（用一小块图去大图里找相同图案）

什么时候用：目标外观、大小、方向都固定（流水线固定零件、定位标记、标定图案）。不适合会旋转/缩放的目标（实测旋转 15° 后分数从 1.0 掉到 0.674）。

```python
import cv2
import numpy as np
from matplotlib import image as mpimg
from matplotlib import pyplot as plt
scene = mpimg.imread(path)
scene = (scene[:, :, :3] * 255).astype("uint8")
template = scene[y1:y2, x1:x2].copy()        # 剪一小块当模板
th, tw = template.shape[:2]
# TM_CCOEFF_NORMED：分数 -1~1，越接近 1 越像
res = cv2.matchTemplate(scene, template, cv2.TM_CCOEFF_NORMED)
print("最高分:", round(float(res.max()), 3), "位置:", cv2.minMaxLoc(res)[3])
threshold = 0.8                              # 阈值 0.7~0.85 之间调
ys, xs = np.where(res >= threshold)
scores = res[ys, xs]
order = np.argsort(-scores)
chosen = []                                  # 去掉挨得太近的重复匹配点
for i in order:
    pt = (int(xs[i]), int(ys[i]))
    if all(abs(pt[0] - q[0]) > tw * 0.5 or abs(pt[1] - q[1]) > th * 0.5 for q in chosen):
        chosen.append(pt)
print("匹配到", len(chosen), "个:", chosen)
result = scene.copy()
for (x, y) in chosen:
    cv2.rectangle(result, (x, y), (x + tw, y + th), (0, 0, 255), 3)
plt.imshow(result); plt.axis("off"); plt.show()
```

· 为什么这么写：模板匹配是归一化互相关：把模板在图上一格一格滑过去算相似度，分数越接近 1 越像。

· 干什么用的：外观、尺寸、方向都固定的目标定位（流水线固定工位最常见）。

· 怎么优化：阈值取 0.7~0.85；用“距离去重”做非极大值抑制；先限定 ROI 能大幅提速。

· 能省哪一步：目标会旋转时，模板匹配可以直接跳过，改用模板 18——省掉一次全图滑窗。

· 和谁配合：和模板 18 是互补关系：先问“会转吗”，不会转才用这一套。

· 下一步改进：多尺度 + 多角度模板匹配，能覆盖小范围的旋转与缩放。

### 模板 18：ORB 特征匹配（旋转、缩放也能认出）

什么时候用：目标会旋转/缩放/部分遮挡，且表面有纹理、文字、角点。纯色光滑物体（如红色方块）特征点太少，ORB 会失败，改用颜色+轮廓。实测：同一目标旋转 30°+缩小 0.8 倍，模板匹配只有 0.087 分，ORB 有 144 个好匹配、131 个内点。

```python
import cv2
import numpy as np
tpl = (mpimg.imread(tpl_path)[:, :, :3] * 255).astype("uint8")
scene = (mpimg.imread(scene_path)[:, :, :3] * 255).astype("uint8")
# 1) 检测特征点并计算描述子
orb = cv2.ORB_create(nfeatures=500)
k1, d1 = orb.detectAndCompute(tpl, None)
k2, d2 = orb.detectAndCompute(scene, None)
# 2) 匹配描述子（ORB 用汉明距离）
bf = cv2.BFMatcher(cv2.NORM_HAMMING, crossCheck=True)
matches = sorted(bf.match(d1, d2), key=lambda m: m.distance)
good = [m for m in matches if m.distance < 50]   # 50~70 之间调
# 3) RANSAC 求单应矩阵，去掉错配
src = np.float32([k1[m.queryIdx].pt for m in good]).reshape(-1, 1, 2)
dst = np.float32([k2[m.trainIdx].pt for m in good]).reshape(-1, 1, 2)
H, mask = cv2.findHomography(src, dst, cv2.RANSAC, 5.0)
print("好匹配:", len(good), "内点:", int(mask.sum()))
# 4) 把模板四个角投影到场景，画出目标框
h, w = tpl.shape[:2]
pts = np.float32([[0, 0], [w, 0], [w, h], [0, h]]).reshape(-1, 1, 2)
dst_pts = cv2.perspectiveTransform(pts, H)
result = scene.copy()
cv2.polylines(result, [np.intp(dst_pts)], True, (0, 0, 255), 4)
plt.imshow(result); plt.axis("off"); plt.show()
```

· 为什么这么写：ORB = FAST 角点 + BRIEF 描述子，用汉明距离匹配，再用 RANSAC 算单应矩阵剔除错配。

· 干什么用的：目标会旋转、缩放、部分遮挡时仍能定位。

· 怎么优化：nfeatures=500 通常够用；距离阈值 50~70；必须用 findHomography + RANSAC 过滤。

· 能省哪一步：纯色无纹理物体特征点太少，用它必然失败；这种情况该省，直接换颜色法。

· 和谁配合：吃模板 1 读入的两张图；输出目标框交给定位流程。

· 下一步改进：换 SIFT/AKAZE 更稳但更慢；工业上也可换成深度学习特征。

## 第六组 · 机器学习与神经网络（让程序自己学）

### 模板 19：机器学习五步流水线（所有模型都这么写）

什么时候用：任何“给例子让它学”的任务。从手写数字到以后的 YOLO，顺序都一样，只需要换两处：数据集、模型。

```python
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
# ① 数据：X 是特征（像素），y 是标签（正确答案）
# X, y = ...
# ② 切分（70% 学，30% 考；random_state 固定住才能复现）
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42, stratify=y)
print("训练", len(y_train), "张，测试", len(y_test), "张")
# ③ 选模型（换模型就改这一行）
from sklearn.neighbors import KNeighborsClassifier
model = KNeighborsClassifier(n_neighbors=3)
# ④ 训练
model.fit(X_train, y_train)
# ⑤ 预测 + 打分（训练集和测试集都打，看差距）
print("训练集准确率：%.4f" % accuracy_score(y_train, model.predict(X_train)))
print("测试集准确率：%.4f" % accuracy_score(y_test, model.predict(X_test)))
```

口诀：切 → 训 → 测 → 打分。训练集和测试集的差距越大，越可能是“背答案”。

· 为什么这么写：机器学习五步：切分 → 训练 → 预测 → 打分 → 看差距。所有模型（包括 YOLO）都是这个骨架。

· 干什么用的：一切“给例子让它自己学”的任务入口。

· 怎么优化：固定 random_state、用 stratify 保证每类都出现；先跑偷懒基线再谈模型好坏。

· 能省哪一步：只是试用模型时交叉验证可以省；正式比较时不能省。

· 和谁配合：吃模板 1–18 产出的特征；供模板 20/21/22/23/24 继续。

· 下一步改进：把特征提取也写进 Pipeline，避免训练与推理不一致。

### 模板 20：混淆矩阵（看清模型错在哪两个类之间）

什么时候用：准确率只有一个数字，看不出问题；混淆矩阵能告诉你在哪些类别上栽跟头。

```python
import matplotlib.pyplot as plt
import numpy as np
from sklearn.metrics import confusion_matrix
plt.rcParams["font.sans-serif"] = ["Microsoft YaHei", "SimHei"]
plt.rcParams["axes.unicode_minus"] = False
cm = confusion_matrix(y_test, pred)      # 行 = 真实答案，列 = 模型预测
print(cm)
fig, ax = plt.subplots(figsize=(5.8, 5.0))
im = ax.imshow(cm, cmap="Blues")
for i in range(cm.shape[0]):
    for j in range(cm.shape[1]):
        if cm[i, j]:
            ax.text(j, i, cm[i, j], ha="center", va="center", fontsize=8,
                    color="white" if cm[i, j] > cm.max() / 2 else "black")
ax.set_xlabel("模型预测")
ax.set_ylabel("真实答案")
fig.colorbar(im)
fig.tight_layout()
fig.savefig(IMG_DIR + r"\混淆矩阵.png", dpi=130)
plt.show()
# 把错的样本挑出来看原图
wrong = np.where(pred != y_test)[0]
for i in wrong:
    print("第 %d 张：真实 %d，猜成 %d" % (i, y_test[i], pred[i]))
```

对角线 = 猜对；非对角线上的数字，就是“真实是 A 却被认成 B”的次数。

· 为什么这么写：混淆矩阵行是真实、列是预测，对角线是猜对的，非对角线就是错法分布。

· 干什么用的：准确率只有一个数字，它告诉你“错在哪两类之间”。

· 怎么优化：归一化后看比例；结合错图一起看，判断是模型问题还是数据问题。

· 能省哪一步：二分类直接看精确率/召回率更快，矩阵可以省。

· 和谁配合：吃模板 19 的预测结果。

· 下一步改进：再看 classification_report 拿到每类的 P/R/F1，写报告更专业。

### 模板 21：保存和加载模型（训练一次，以后直接用）

什么时候用：训练一次可能要几分钟，不想每次重跑；或者要把模型搬到别的脚本、接到机械臂或小车上用。

```python
import joblib
joblib.dump(model, IMG_DIR + r"\我的模型.joblib")     # 存：一行搞定
loaded = joblib.load(IMG_DIR + r"\我的模型.joblib")   # 读回来
print("读回来的准确率：%.4f" % accuracy_score(y_test, loaded.predict(X_test)))
# 以后直接 loaded.predict(新数据) 就能用，不用再 fit
```

注意：模型文件只认“同样格式的输入”。今天输入是 64 个像素值，以后换成别的数据，必须保证喂进去的格式和训练时一致。

· 为什么这么写：joblib 把训练好的模型序列化到磁盘，下次直接读回，不用重新训练。

· 干什么用的：训练一次、反复使用；也是把模型交给别的程序（机械臂、界面）的方式。

· 怎么优化：文件名带版本或日期；同时记录训练时的输入格式（形状、值域）。

· 能省哪一步：没有冗余，但要知道“读回来的模型必须喂同样格式的数据”。

· 和谁配合：吃模板 19/24 的模型对象，供部署环节。

· 下一步改进：导出 ONNX，可以脱离 Python 环境部署。

### 模板 22：换模型对比（同一份数据，只换模型那一行）

什么时候用：不确定该用哪个模型时，别猜，比一比。“选模型”和“调参数”是两件事：选模型 = 换整套方法（kNN ↔ 逻辑回归 ↔ 神经网络）；调参数 = 同一套方法里拧旋钮（n_neighbors 从 3 改成 5）。顺序是先选模型，再调参数。

```python
import time
import numpy as np
import matplotlib.pyplot as plt
from sklearn.datasets import load_digits
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score
from sklearn.neighbors import KNeighborsClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.ensemble import RandomForestClassifier
from sklearn.neural_network import MLPClassifier
plt.rcParams["font.sans-serif"] = ["Microsoft YaHei", "SimHei"]
plt.rcParams["axes.unicode_minus"] = False
IMG_DIR = r"D:\Codex\cv-learning\images"
# ① 数据（换成自己的数据也只需要改这两行）
digits = load_digits()
X, y = digits.data, digits.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42, stratify=y)
# ② 候选模型：想加一个，就照格式多写一行（③ 换模型就是改这里）
candidates = [
    ("3 最近邻 kNN", KNeighborsClassifier(n_neighbors=3)),
    ("逻辑回归", LogisticRegression(max_iter=2000)),
    ("决策树", DecisionTreeClassifier(random_state=42)),
    ("随机森林", RandomForestClassifier(n_estimators=100, random_state=42)),
    ("小型神经网络 MLP", MLPClassifier(hidden_layer_sizes=(64,), max_iter=600,
                                     random_state=42)),
]
# ③ 同一套流程跑一遍：训练 → 预测 → 打分 → 记时间
names, accs, times = [], [], []
for name, model in candidates:
    t0 = time.perf_counter()
    model.fit(X_train, y_train)
    dt = time.perf_counter() - t0
    acc = accuracy_score(y_test, model.predict(X_test))
    names.append(name)
    accs.append(acc * 100)
    times.append(dt)
    print("%-16s 训练 %5.2f 秒  测试 %.2f%%" % (name, dt, acc * 100))
print("这次最好的是：", names[int(np.argmax(accs))])
# ④ 画成条形图，一眼看出谁强谁弱
fig, ax = plt.subplots(figsize=(7.2, 4.0))
bars = ax.barh(names, accs, color="#4a86c8")
for b, a, t in zip(bars, accs, times):
    ax.text(b.get_width() + 0.2, b.get_y() + b.get_height() / 2,
            "%.2f%%（训练 %.2f 秒）" % (a, t), va="center", fontsize=9)
ax.set_xlim(0, 118)
ax.set_xlabel("测试集准确率（%）")
ax.set_title("同一份数据、同一套流程，只换模型")
fig.tight_layout()
fig.savefig(IMG_DIR + r"\模型对比.png", dpi=130)
plt.show()
```

实测（2026-09-17，手写数字，random_state=42）：

| 模型 | 测试集准确率 | 训练耗时 |
| --- | --- | --- |
| 3 最近邻 kNN | 98.70% | 0.00 秒 |
| 逻辑回归 | 95.74% | 0.06 秒 |
| 决策树 | 85.19% | 0.01 秒 |
| 随机森林 | 96.67% | 0.13 秒 |
| 小型神经网络 MLP | 95.74% | 0.33 秒 |

比出来的结论：① 数据少、图小的时候，kNN 往往最强，别急着上神经网络；② 神经网络只有 95.74%，不是它不行，是这里的数据太少太小，它发挥不出来；③ 准确率差不多时，选训练快的那个（随机森林 0.13 秒 vs 逻辑回归 0.06 秒）。

对比时必须“只动一个变量”：同一份切分（random_state 要一样）、同一个打分方法、同一份测试集。要是连切分都换了，那就不是在比模型，是在比运气。

· 为什么这么写：同一份数据、同一套流程，只换模型那一行，比谁的成绩好、谁更快。

· 干什么用的：不确定用哪个模型时，用实验代替猜测。

· 怎么优化：必须固定切分与打分方式，否则比的是运气；同时记录参数量和训练时间。

· 能省哪一步：明显更差的模型可以少跑几轮（早停），节省时间。

· 和谁配合：吃模板 19 的切分，输出“选哪个模型”的结论。

· 下一步改进：进阶用网格搜索或 AutoML 系统化地搜。

### 模板 23：神经网络（加隐藏层，学法“拐弯”的分界线）

什么时候用：一层模型怎么调都学不会时（比如圆环分布、XOR 这种“分界线不直”的问题）。一个神经元 = 加权求和 + 激活；多层叠起来就能拟合任意弯曲的边界。

```python
from sklearn.neural_network import MLPClassifier
from sklearn.metrics import accuracy_score
# ① 数据：沿用模板 19 的切分
# X_train, X_test, y_train, y_test = ...
# ② 选结构：隐藏层写成一个元组
model = MLPClassifier(
    hidden_layer_sizes=(64,),   # (64,) = 一层 64 个神经元；(128, 64) = 两层
    max_iter=2000,              # 训练轮数，太小会学不完（会出 ConvergenceWarning）
    random_state=42,
)
model.fit(X_train, y_train)
print("测试集准确率：%.4f" % accuracy_score(y_test, model.predict(X_test)))
# ③ 数一数网络里有多少参数
for i, (w, b) in enumerate(zip(model.coefs_, model.intercepts_)):
    print("第 %d 层：权重 %s（%d 个），偏置 %d 个" % (i + 1, w.shape, w.size, b.size))
# ④ 看隐藏层把输入变成了什么（第一层输出 = 像素 @ 权重 + 偏置，再过 sigmoid）
import numpy as np
h = X_test[0:1] @ model.coefs_[0] + model.intercepts_[0]
h = 1 / (1 + np.exp(-h))
print("隐藏层输出：", np.round(h[0], 2).tolist())
```

实测（2026-09-18，手写数字，1257 张训练图）：

| 网络结构 | 测试集准确率 | 参数个数 | 训练耗时 |
| --- | --- | --- | --- |
| 单层（无隐藏层） | 95.74% | 650 | 0.06 秒 |
| 1 层 8 个神经元 | 92.59% | 610 | 0.74 秒 |
| 1 层 64 个神经元 | 95.74% | 4810 | 0.37 秒 |
| 2 层 128 + 64 | 97.59% | 17226 | 0.43 秒 |

“分界线不直”的两个经典例子：XOR（单层 0.5000，加隐藏层 1.0000）；圆环分布（单层 0.4167，双层 16 个神经元 1.0000）。

训练轮数的影响（1 层 64 个神经元）：max_iter=1 → 0.1704，3 → 0.3852，10 → 0.8130，50 → 0.9426，300 → 0.9574（到 300 轮就基本到顶了）。

坑：小网络的“初始化运气”。XOR 那个例子用 4 个神经元时，random_state=0/1/7 都能到 100%，换成 42 就只有 50%；加到 16 个神经元就稳定了。所以实验要固定随机种子，并且不要拿一次结果下结论。另一个坑：极小网络要把正则化关掉（alpha=0.0），否则默认的 alpha 会把它压得学不动。

· 为什么这么写：神经元 = 加权求和 + 激活；多层叠加能拟合弯曲的分界（XOR、圆环都靠隐藏层）。

· 干什么用的：分界线不是直线时使用。

· 怎么优化：隐藏层先试 64；max_iter 给 2000；极小网络要把正则化关掉（alpha=0）才学得动。

· 能省哪一步：数据很少时小 MLP 打不过 kNN（实测 92.59% vs 98.70%），这种情况别硬上神经网络。

· 和谁配合：吃模板 19 的数据；供模板 24/25 继续做深度模型。

· 下一步改进：加早停、批归一化、Dropout 能进一步稳住效果。

### 模板 24：PyTorch 训练循环（前向 → 损失 → 反向 → 更新）

什么时候用：想自己控制训练过程、或者要处理 kNN/sklearn 搞不定的复杂图像时。PyTorch 是工业界的主力，CNN、YOLO 都建在它上面。张量 = 能自动求导的数组。

```python
import torch
import torch.nn as nn
torch.manual_seed(42)          # 固定随机种子
# ① 数据转成张量（分类的标签必须用 long）
X = torch.tensor(X_train, dtype=torch.float32)
y = torch.tensor(y_train, dtype=torch.long)
# ② 三件套：模型、损失函数、优化器
model = nn.Linear(64, 10)                     # 换成 nn.Sequential(...) 就是多层网络
loss_fn = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.01)
# ③ 训练循环：这四步是所有 PyTorch 程序的心脏
for epoch in range(300):
    logits = model(X)                 # 1 前向：算预测
    loss = loss_fn(logits, y)         # 2 损失：差多少
    optimizer.zero_grad()             #   清空上一轮梯度（忘了它会出大问题）
    loss.backward()                   # 3 反向：算梯度
    optimizer.step()                  # 4 更新：调权重
    if epoch % 50 == 0:
        print("第 %3d 轮 损失 %.4f" % (epoch, float(loss)))
# ④ 在测试集上打分
with torch.no_grad():
    pred = model(torch.tensor(X_test, dtype=torch.float32)).argmax(dim=1)
    acc = float((pred == torch.tensor(y_test)).float().mean())
print("测试集准确率：%.4f" % acc)
# ⑤ 自动求导单练：y = x²，在 x=3 处的导数是 6
x = torch.tensor(3.0, requires_grad=True)
(x ** 2).backward()
print("dy/dx =", float(x.grad))
```

实测：

| 例子 | 结果 |
| --- | --- |
| 拟合 y = 2x + 1 | 损失 54.87 → 0.08，学出 w=1.992（真实 2）、b=1.010（真实 1） |
| 手写数字 nn.Linear(64,10) | 300 轮后测试准确率 97.59%，训练 2.14 秒（CPU） |
| 自动求导 | x=3 时 y=x² 的导数 6.0；x=2 时 3x³+2x 的导数 38.0，与手算一致 |

对照：同一个手写数字任务，kNN 是 98.70%（不用训练），一层小网络 97.59%——下一课换成卷积网络（CNN）就能超过它。

· 为什么这么写：训练循环四步：前向 → 损失 → 反向 → 更新，外加 zero_grad 清梯度。

· 干什么用的：需要自己控制训练过程时的标准写法。

· 怎么优化：先小学习率跑通再加；zero_grad 必写，否则梯度累加；评估用 no_grad 省内存。

· 能省哪一步：数据只有几百条时可以全批量训练，省掉 batch 循环。

· 和谁配合：吃模板 19 的数据；供模板 25（CNN）复用同一套循环思想，模板 28 也是它的放大版。

· 下一步改进：加学习率调度、混合精度、GPU 迁移，能把训练时间再压一个数量级。

## 第七组 · 目标检测与项目实战

### 模板 25：PyTorch CNN（卷积 + 池化，看懂“滑动窗口”怎么找特征）

什么时候用：图像任务里最常用的网络。它拿着 3×3 的小窗口在图上滑动，权重被所有位置共用，所以参数不多却能看懂整张图，而且对平移不敏感。

```python
import torch
import torch.nn as nn
class SimpleCNN(nn.Module):
    def __init__(self):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 8, 3, padding=1), nn.ReLU(), nn.MaxPool2d(2),    # 8x8 -> 4x4
            nn.Conv2d(8, 16, 3, padding=1), nn.ReLU(), nn.MaxPool2d(2),   # 4x4 -> 2x2
        )
        self.classifier = nn.Sequential(nn.Flatten(), nn.Linear(16 * 2 * 2, 10))
    def forward(self, x):
        return self.classifier(self.features(x))
# 输入必须是四维：(批次, 通道, 高, 宽)
X = (digits.data / 16.0).astype("float32").reshape(-1, 1, 8, 8)
model = SimpleCNN()
print("参数个数：", sum(p.numel() for p in model.parameters()))
# 训练循环和模板 24 完全一样
for epoch in range(60):
    model.train()
    order = torch.randperm(len(X_train))
    for i in range(0, len(X_train), 64):
        idx = order[i:i + 64]
        loss = criterion(model(X_train[idx]), y_train[idx])
        optimizer.zero_grad(); loss.backward(); optimizer.step()
```

实测（手写数字，9/19）：小 CNN（8/16 通道）1898 个参数、60 轮 → 97.96%，训练 1.9 秒；加大版（16/32 + 64 隐藏）13706 个参数、200 轮 → 98.70%，训练 6.1 秒。对照：一层网络 97.59%，kNN 98.70%。

卷积核可以当成“特征探测器”：中间一列权重大就是竖线检测，中间一行大就是横线检测。训练就是在学这些 3×3 的小数字。

· 为什么这么写：卷积用共享的小窗口提取局部特征（边缘、纹理），池化降尺寸并带来一点平移不变性。

· 干什么用的：图像类任务的默认网络：参数少，又能看整张图。

· 怎么优化：输入必须是 (批次, 通道, 高, 宽) 四维；通道 8→16→32 逐层加；池化别随便删。

· 能省哪一步：小图小数据上 CNN 不一定赢 kNN（97.96% vs 98.70%），先想清楚值不值得。

· 和谁配合：吃模板 24 的训练循环；配合模板 26 做增强；供模板 28 做检测骨干。

· 下一步改进：加残差连接、批归一化，或直接换预训练网络做迁移学习。

### 模板 26：数据增强（让模型不怕旋转、不怕歪）

什么时候用：相机角度会变、物体摆放会歪、或者数据不够时。做法是训练时把每张图随机转几度、挪一两个像素、缩放一点点，相当于凭空多出一批“不太一样但答案相同”的数据。

```python
import cv2
import numpy as np
def make_augmented(imgs, rng, max_angle=8.0, max_shift=0.8, scale_range=(0.95, 1.05)):
    out = []
    for im in imgs:
        img = (im[0] * 16).astype("float32")
        m = cv2.getRotationMatrix2D((3.5, 3.5),
                                    rng.uniform(-max_angle, max_angle),
                                    rng.uniform(*scale_range))
        m[0, 2] += rng.uniform(-max_shift, max_shift)
        m[1, 2] += rng.uniform(-max_shift, max_shift)
        img = cv2.warpAffine(img, m, (8, 8), flags=cv2.INTER_LINEAR,
                             borderMode=cv2.BORDER_REPLICATE)
        out.append(np.clip(img, 0, 16).reshape(1, 8, 8) / 16.0)
    return np.array(out, dtype="float32")
# 训练时：把这一批换成增强后的版本再喂给模型
batch = torch.tensor(make_augmented(X_train[idx], rng))
```

实测（9/19，手写数字，两个模型结构相同、都 300 轮）：

| 考卷 | 不用增强 | 用了增强 |
| --- | --- | --- |
| 原图测试集 | 97.96% | 99.07% |
| 旋转 10° | 95.74% | 98.70% |
| 旋转 20° | 80.37% | 94.63% |
| 旋转 30° | 53.89% | 72.22% |

增强强度是旋钮：温和（±8°）干净数据最好；中性（±12°）旋转时更稳。小图别开太猛——8×8 的图转 30° 已经不是“换个角度”，而是“看不清了”。

· 为什么这么写：训练时随机旋转/平移/缩放，等价于凭空多出一批“不太一样但答案相同”的数据。

· 干什么用的：让模型不怕物体摆歪、位置偏移。

· 怎么优化：增强幅度要匹配真实变化（8×8 小图 ±8° 就够，转 30° 等于把数字糊掉）；只在训练时开。

· 能省哪一步：增强会让训练时间翻倍，可以每 2 个 batch 才增强一次，效果基本不变。

· 和谁配合：挂在模板 25 的训练循环里；验证时用“变难的考卷”看效果。

· 下一步改进：进阶用 Mixup、Cutout、AutoAugment，或直接扩真实数据。

### 模板 27：完整分拣流程（识别 → 定位 → 毫米 → 指令表）

什么时候用：机械臂/分拣项目的视觉端交付物。一条链跑到黑：颜色分割 → 去噪 → 找轮廓 → 形状 → 旋转框 → 像素换算毫米 → 排抓取顺序 → 输出指令表。

```python
import csv
# 1) HSV 分割（比 RGB 抗光照变化）
hsv = cv2.cvtColor(rgb, cv2.COLOR_RGB2HSV)
H, S, V = hsv[:, :, 0].astype(int), hsv[:, :, 1].astype(int), hsv[:, :, 2].astype(int)
red_mask = ((H <= 10) | (H >= 168)) & (S > 90) & (V > 60)
green_mask = (H >= 40) & (H <= 85) & (S > 80) & (V > 60)
# 2) 去噪 + 面积过滤 + 每个物体的中心/宽高/角度
m8 = cv2.morphologyEx((red_mask.astype("uint8")) * 255, cv2.MORPH_OPEN,
                      np.ones((5, 5), np.uint8))
found = cv2.findContours(m8, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
contours = [c for c in (found[0] if len(found) == 2 else found[1])
            if cv2.contourArea(c) > 3000]
(cx, cy), (w, h), angle = cv2.minAreaRect(contours[0])
# 3) 比例尺：约定参考物（红色方形）边长 100 毫米
MM_PER_PX = 100.0 / 110.0            # 110 像素 = 100 毫米
center_mm = (round(cx * MM_PER_PX, 1), round(cy * MM_PER_PX, 1))
# 4) 抓取顺序 + 目标筐 + 指令表
objects.sort(key=lambda o: (o["中心像素"][1], o["中心像素"][0]))   # 从上到下
for i, o in enumerate(objects, 1):
    o["抓取顺序"] = i
    o["目标筐"] = "A 筐（左）" if o["颜色"] == "红" else "B 筐（右）"
with open("项目二_分拣指令.csv", "w", newline="", encoding="utf-8-sig") as fh:
    writer = csv.DictWriter(fh, fieldnames=["抓取顺序", "颜色", "形状",
                                            "中心毫米X", "中心毫米Y", "角度", "目标筐"])
    writer.writeheader()
```

实测（9/19 项目二，960×640 场景）：6 个零件全部识别（红 3 绿 3）、灰尘被过滤；比例尺 0.9091 毫米/像素；平均定位误差 0.36 毫米、最大 0.65 毫米（指标 ≤5 毫米）；整条流程 0.38 秒（指标 ≤15 秒）。

角度那一列有坑：minAreaRect 对正的方形/圆会输出 -90°/-45°，没有固定朝向。要给夹爪用必须再换算，或者只抓方形/圆的零件避开姿态问题。

· 为什么这么写：识别 → 定位 → 毫米换算 → 分拣决策 → 输出指令表，是机械臂视觉端的完整交付链。

· 干什么用的：所有要被机械臂执行的视觉程序都长这样。

· 怎么优化：顺序固定：光照/颜色 → 掩码 → 开运算 → 轮廓 → 面积过滤 → 形状 → 旋转框 → 毫米；比例尺挑最稳的参考物。

· 能省哪一步：只需要坐标时形状与角度可以省；要做分拣就保留。

· 和谁配合：它是模板 11 + 14 + 4 + 9 + 8 + 16 + 21 的组装件。

· 下一步改进：接上手眼标定矩阵后，输出直接变成机械臂坐标。

### 模板 28：YOLO 训练（让模型自己找出物体在哪）

什么时候用：物体位置不固定、有多个目标、颜色法不稳的时候。数据集要先做成 YOLO 格式：images/ 与 labels/ 一一对应，标签每行是“类别 中心x 中心y 宽 高”（都除以图片宽高，0~1）。

```python
from ultralytics import YOLO
# 推荐：官方预训练权重（第一次自动下载约 6.5 MB）
model = YOLO("yolov8n.pt")
# 下载不了时再用从零训练（代码结构一模一样，只是起点不同）
# model = YOLO("yolov8n.yaml")
model.train(
    data=r"D:\Codex\cv-learning\datasets\零件检测\data.yaml",
    epochs=40,        # 训练轮数
    imgsz=160,        # 输入尺寸，CPU 用小一点
    batch=16,
    device="cpu",     # 有显卡写 device=0
    workers=0,        # Windows 受限环境报多进程错误时设 0
    seed=42,
)
# 用最好的权重评估
best = YOLO(r"runs/零件检测_yolov8n/weights/best.pt")
m = best.val(data=r"D:\Codex\cv-learning\datasets\零件检测\data.yaml", imgsz=160)
print("mAP50 =", m.box.map50, " mAP50-95 =", m.box.map)
```

数据集目录结构（图片和标签必须同名）：

```python
datasets/零件检测/
    images/train/train_000.png        图片
    labels/train/train_000.txt        标签（一行一个物体）
    images/val/val_000.png
    labels/val/val_000.txt
    data.yaml                         告诉 YOLO 数据在哪、类别叫什么
```

权重下载卡住时，让 Python 走本机代理（端口按你的代理软件改，Clash Verge 默认 7897）：

```python
$env:HTTPS_PROXY = "http://127.0.0.1:7897"
python -c "from ultralytics import YOLO; YOLO('yolov8n.pt')"   # 下好一次，以后就不用了
```

实测（9/19，自建数据集 140 张训练图 / 487 个框，CPU，160×160，40 轮）：

| 起点 | mAP50 | mAP50-95 | 精确率 P | 召回率 R | 耗时 |
| --- | --- | --- | --- | --- | --- |
| 从零训练（yolov8n.yaml） | 0.9857 | 0.7070 | 0.9165 | 0.9637 | 75.3 秒 |
| 预训练权重（yolov8n.pt） | 0.9950 | 0.9282 | 0.9964 | 1.0000 | 65.4 秒 |

数据少的时候差距会拉得很大（30 张训练图、20 轮）：从零训练 mAP50 = 0.0000，换成预训练权重、同样 30 张图 = 0.7061。所以数据不够时，第一步不是加数据，而是先把起点换成预训练权重。

前 10 轮 mAP 为 0 是正常的——检测模型前期都在学“框该怎么画”。

· 为什么这么写：YOLO 是单阶段检测：一次前向就给出所有框和类别；用预训练权重做迁移学习收敛最快。

· 干什么用的：目标位置不固定、数量不固定的场景（零件散乱摆放）。

· 怎么优化：优先预训练权重（30 张训练图：从零 0.0000 → 预训练 0.7061）；imgsz 影响精度与速度；Windows 受限环境写 workers=0。

· 能省哪一步：数据集很小时只训 20~30 轮就够；mAP 早平了就是白跑。

· 和谁配合：吃模板 10（造图）/15（批量）的准备成果；供模板 29 推理评估。

· 下一步改进：加 mosaic 与色彩增强、把 imgsz 提到 320/640、或换更大的模型，都能继续涨点。

### 模板 29：YOLO 推理与效果检查（预测 + 看图 + 读指标）

什么时候用：训练完想看看它到底行不行。三件事：预测一张新图、看预测框、读指标曲线。

```python
from ultralytics import YOLO
model = YOLO(r"runs/零件检测_yolov8n/weights/best.pt")
# 1) 预测（conf 是置信度阈值，低于它的框不显示）
results = model.predict("新照片.png", imgsz=160, conf=0.3, device="cpu")
for box in results[0].boxes:
    cls = int(box.cls)                 # 0 或 1
    conf = float(box.conf)             # 置信度
    x1, y1, x2, y2 = [float(v) for v in box.xyxy[0]]   # 框的像素坐标
    print("类别 %d 置信度 %.2f 位置 (%.0f, %.0f) - (%.0f, %.0f)" % (cls, conf, x1, y1, x2, y2))
# 2) 把框画到图上保存（直接接第 27 课的毫米换算和指令表）
cv2.imwrite("检测结果.png", results[0].plot()[:, :, ::-1])
# 3) 看整套指标：mAP、PR 曲线、混淆矩阵都在 runs/ 目录里
#    BoxPR_curve.png（越靠右上越好）、confusion_matrix.png（有没有把红绿弄混）
#    val_batch0_pred.jpg（预测框）、val_batch0_labels.jpg（真值框）
```

实测（9/19）：拿只在 320×240 小图上训练过的模型，去检测第 25 课那张 960×640 的分拣场景，6 个零件全部检出、没有误报灰尘，置信度 0.844 / 0.769 / 0.732 / 0.719 / 0.660 / 0.598；单张推理约 2.1 毫秒。

指标怎么看：召回率高 = 漏得少；精确率高 = 报得准；mAP50 是综合分，0.9 以上算这个任务做成了。mAP50 高但 mAP50-95 低，说明“找得到但框不够准”，可以加大 imgsz。

· 为什么这么写：推理是前向输出“框 + 置信度”；val 用 IoU 阈值加平均精度算出 mAP 系列指标。

· 干什么用的：上线前的验证：既看单张预测图，也看整体指标。

· 怎么优化：conf 阈值按场景调（0.25~0.5）；批量推理比逐张快；有 GPU 就上 GPU。

· 能省哪一步：演示给别人看时不必算 mAP，直接预测一张图更快。

· 和谁配合：吃模板 28 的模型；输出框中心可以接回模板 27 的毫米换算与分拣决策。

· 下一步改进：导出 ONNX / TensorRT 能显著提速，适合产线部署。

## 第八组 · 标定与执行（把视觉接到真机上）

### 模板 30：标定与毫米换算（像素 → 机械臂坐标）

什么时候用：相机装好、要把它“看到的位置”变成机械臂坐标的时候。这是视觉和机械臂之间唯一的桥。

```python
import numpy as np
import cv2
```

```python
# ① 采集：让末端走 N 个点，每个点同时记两组数
#    相机看到的标记中心(像素) <-> 机械臂读数(毫米)
pts_px = np.float32([[120, 80], [500, 80], [120, 400], [500, 400], [310, 240]])
pts_mm = np.float32([[100, 60], [400, 60], [100, 300], [400, 300], [250, 180]])
```

```python
# ② 拟合：必须用允许镜像的 estimateAffine2D（图像 y 向下、机械臂 y 向上）
M, inliers = cv2.estimateAffine2D(pts_px, pts_mm, method=cv2.RANSAC)
np.save(r"D:\Codex\cv-learning\images\标定_像素到机械臂.npy", M)
```

```python
# ③ 验证：算残差，超过 2~3 毫米就重标
pred = (M[:, :2] @ pts_px.T).T + M[:, 2]
err = np.hypot(*(pred - pts_mm).T)
print("最大残差 %.2f 毫米" % err.max())
```

```python
# ④ 以后每次就这么用（读回矩阵 -> 换算）
def to_arm(u, v, M):
    return M[0, 0] * u + M[0, 1] * v + M[0, 2], M[1, 0] * u + M[1, 1] * v + M[1, 2]
```

```python
M = np.load(r"D:\Codex\cv-learning\images\标定_像素到机械臂.npy")
print("零件中心像素 (300, 200) -> 机械臂坐标", to_arm(300, 200, M))
```

· 为什么这么写：把“标定 + 毫米换算”独立出来：只要输入像素点对和对应世界坐标，就能拟合出一个矩阵。

· 干什么用的：换相机、换工作台高度、换镜头之后，重新算一次比例关系，其他代码不用动。

· 怎么优化：用 estimateAffine2D（允许镜像）而不是 estimateAffinePartial2D（只允许旋转缩放）；拟合完一定要留几个没参与拟合的点做验证。

· 能省哪一步：只是同一台相机同一高度时，不必每次重标，把矩阵存成文件读回来即可。

· 和谁配合：吃模板 4/16 的中心像素；输出喂模板 27 的分拣决策。

· 下一步改进：改用相机内参标定 + 手眼标定，可以同时纠正镜头畸变和相机倾斜。

实测：4 个标记拟合的残差最大 0.11 毫米；换成不会镜像的 estimateAffinePartial2D 时残差涨到 11 毫米、方向整个翻转。

### 模板 31：仿真抓取与动作序列（没有真机也能练）

什么时候用：没有机械臂、或者想在真机之前把逻辑跑通的时候。同一份动作表，换成真机的 move 指令就能用。

```python
import numpy as np
```

```python
L1 = L2 = 250.0                     # 连杆长度（毫米）
Z_HOVER, Z_PICK, Z_PLACE = 60.0, 5.0, 35.0
```

```python
def ik(x, y):
    """逆运动学：给末端 (x, y)，求两个关节角（肘部朝上）"""
    r2 = x * x + y * y
    c2 = np.clip((r2 - L1 ** 2 - L2 ** 2) / (2 * L1 * L2), -1.0, 1.0)
    q2 = -np.arccos(c2)
    return np.arctan2(y, x) - np.arctan2(L2 * np.sin(q2), L1 + L2 * np.cos(q2)), q2
```

```python
def plan_pick_place(targets, bins):
    """把目标点变成动作序列：悬停 -> 下降 -> 夹 -> 抬起 -> 平移 -> 下降 -> 松 -> 抬起"""
    actions = []
    for t in targets:
        x, y = t["机械臂坐标"]
        bx, by = bins[t["目标筐"]]
        actions += [("hover", t, (x, y, Z_HOVER)), ("down", t, (x, y, Z_PICK)),
                    ("close", t, (x, y, Z_PICK)), ("up", t, (x, y, Z_HOVER)),
                    ("move", t, (bx, by, Z_HOVER)), ("drop", t, (bx, by, Z_PLACE)),
                    ("open", t, (bx, by, Z_PLACE)), ("back", t, (bx, by, Z_HOVER))]
    return actions
```

```python
def run(actions, reach=L1 + L2):
    """仿真执行：能解出逆运动学且不超行程，就算这次抓取成功"""
    ok = 0
    for act, t, (x, y, z) in actions:
        if np.hypot(x, y) > reach * 0.98:
            print("超出行程，跳过：", act, t.get("颜色", ""))
            continue
        ik(x, y)
        if act == "close":
            ok += 1
    return ok
```

· 为什么这么写：把抓取动作写成“动作序列 + 逆运动学”：先由目标点解出关节角，再按顺序执行。

· 干什么用的：没有真机时验证整套抓放逻辑；有真机时同一份动作表可以直接下发给控制器。

· 怎么优化：动作顺序固定为 悬停 → 下降 → 夹 → 抬起 → 平移 → 下降 → 松 → 抬起；三个高度在真机上用示教器记录。

· 能省哪一步：二维仿真里 Z 高度看不出效果，做演示时可以只显示俯视轨迹；要真实感就用三维仿真。

· 和谁配合：吃模板 27/30 输出的目标点和筐位；是模板 27 的下游。

· 下一步改进：加碰撞检查、速度规划、抓取姿态（夹爪旋转），才是完整的运动规划。

实测：6 个零件、两个料筐，逆运动学解算全部成功；其中右筐距底座 550 毫米 > 500 毫米臂展，被程序判定为“超出行程”而跳过。

## 附加小抄：第 19–29 课的函数（不用背，查就行）

| 函数 / 参数 | 作用 | 常改成什么 |
| --- | --- | --- |
| load_digits( ) | 读取自带的手写数字数据集 | 以后换成自己拍的数据 |
| train_test_split( ) | 切分训练集和测试集 | test_size、random_state、stratify |
| KNeighborsClassifier( ) | k 最近邻分类器 | n_neighbors 从 1 试到 50 |
| LogisticRegression( ) | 逻辑回归分类器 | max_iter=2000 防止不收敛 |
| DecisionTreeClassifier( ) | 决策树（一串 if-else 规则） | random_state=42 |
| RandomForestClassifier( ) | 随机森林（很多棵树投票） | n_estimators 树的数量 |
| MLPClassifier( ) | 小型神经网络 | hidden_layer_sizes 每层多少个神经元 |
| model.fit( ) | 训练 | 不用改 |
| model.predict( ) | 预测答案 | 不用改 |
| model.predict_proba( ) | 看每个类别有多大的可能性 | 想解释“为什么猜这个”时用 |
| accuracy_score( ) | 算准确率 | 先 predict 再算 |
| confusion_matrix( ) | 混淆矩阵 | 看错在哪两个类之间 |
| joblib.dump / load | 保存 / 读取模型 | 文件名自己起 |

PyTorch 部分（第 22 课，字典里最像“工业界写法”的一段）：

| 函数 / 参数 | 作用 | 常改成什么 |
| --- | --- | --- |
| torch.tensor( ) | 创建张量（能自动求导的数组） | dtype=torch.float32 / .long() |
| torch.manual_seed(42) | 固定随机种子，结果可复现 | 换数字就换结果 |
| a.reshape / a.T | 改形状 / 转置 | 形状不对先 print(a.shape) |
| a @ b | 矩阵乘（* 是逐元素乘，别搞混） | 神经网络里到处是它 |
| a.sum(dim=1) | 按维度求和（dim=0 是列） | 统计、损失计算常用 |
| requires_grad=True | 声明这个数要算梯度 | 训练参数自动带，不用手写 |
| loss.backward( ) | 反向传播：自动算所有梯度 | 后面必须配 optimizer.step() |
| x.grad | 看算出来的梯度 | 验证手算结果时用 |
| nn.Linear(64, 10) | 一层全连接（64 进 10 出） | 多层用 nn.Sequential 串起来 |
| nn.CrossEntropyLoss( ) | 分类任务的损失函数 | 配 argmax 得到预测类别 |
| torch.optim.Adam( ) | 优化器（负责调权重） | lr 学习率常用 0.001~0.01 |
| optimizer.zero_grad( ) | 清空上一轮梯度 | 每个 batch 都要写，忘了损失不降 |
| optimizer.step( ) | 按梯度更新权重 | 训练循环的最后一步 |
| torch.no_grad( ) | 只做预测不算梯度（省内存） | 测试/打分时套在外面 |
| nn.Conv2d(1, 8, 3) | 卷积层（1 进 8 出，3×3 窗口） | 通道数 8→16→32 往上加 |
| nn.MaxPool2d(2) | 池化层（每 2×2 留最大） | 去掉它会变差，一般保留 |
| (N, 1, 8, 8) | CNN 要的四维输入 | 别用拉平的 (N, 64) |

检测部分（第 26–27 课，用的是 ultralytics 库）：

| 函数 / 参数 | 作用 | 常改成什么 |
| --- | --- | --- |
| YOLO("yolov8n.pt") | 加载官方预训练模型（首次自动下载 6.5 MB） | 推荐起点；下载卡住就设 HTTPS_PROXY |
| YOLO("yolov8n.yaml") | 从零搭建模型结构（不需要联网） | 下载不了时才用；数据少时效果差很多 |
| model.train( ) | 训练检测模型 | epochs、imgsz、batch、device |
| model.val( ) | 在验证集上算指标 | 看 mAP50、mAP50-95 |
| model.predict( ) | 对一张新图做检测 | conf 置信度阈值常用 0.25~0.5 |
| box.cls / box.conf | 读类别 / 置信度 | 用来筛选和判断 |
| box.xyxy[0] | 读框的像素坐标 (x1,y1,x2,y2) | 接毫米换算和指令表 |
| results.plot( ) | 把框画到图上（返回 BGR 图） | 存图前记得 [:, :, ::-1] |
| box.map50 / box.map | mAP50 / mAP50-95 | 综合分，0.9 以上算做成 |
| workers=0 | 关掉多进程数据加载 | Windows 或多进程报错时就写它 |

## 常见小问题（先查这里，再问人）

| 报错 / 现象 | 原因和改法 |
| --- | --- |
| FileNotFoundError | 图片路径写错了，先确认 images 文件夹里有没有这个文件 |
| 路径报错 | Windows 路径前面记得加 r，例如 r"D:\..." |
| IndexError | 数字超出了图片大小，先 print(img.shape) 再看数字 |
| 颜色全黑 / 全白 | 0~1 小数和 0~255 整数的阈值不能混用；颜色淡就把阈值放宽 |
| 透明通道 | 形状是 (高, 宽, 4) 时记得运行 img = img[:, :, :3] |
| 找轮廓数量不对 | 先显示 mask8 黑白图，确认只剩想要的物体 |
| OpenCV 版本不同 | 轮廓写法用 contours = found[0] if len(found) == 2 else found[1] |
| 颜色顺序 | matplotlib 读图是红绿蓝，画红色用 (255,0,0)；cv2.imread 读图是蓝绿红，红色要写 (0,0,255) |
| 图里的中文变成方框 | 漏了 plt.rcParams["font.sans-serif"] 那两行 |
| No module named 'sklearn' | 要用课程自带的 Python 运行时（在同一个 Jupyter 里跑） |
| 每次运行结果都不一样 | 没写 random_state，切分方式每次都变 |
| 训练集 100% 但测试很差 | 数据太少在“背答案”（过拟合），加数据或把 k 调大 |
| 换了模型反而更差 | 很正常：数据少、图小的时候 kNN 常常最强。先看是不是参数没调，再决定换不换 |
| 模型训练很慢 | 数据量、模型复杂度都会影响；准确率差不多时选快的那个 |
| 损失不下降 / 一直震荡 | PyTorch 里八成忘了 optimizer.zero_grad()；学习率太大也会震荡 |
| RuntimeError: expected scalar type Long | 分类的标签张量要写 dtype=torch.long |
| 隐藏层加到 8 个反而不如单层 | 神经元太少学不动；先加到 32 或 64 再试 |
| ConvergenceWarning（sklearn） | max_iter 太小没收敛，调到 2000 左右 |
| CNN 报 shape 不对 | 卷积网络要四维输入 (批次, 通道, 高, 宽)，先把图 reshape 成 (N,1,H,W) |
| 数据增强后准确率反而降 | 增强幅度太大（小图转 30° 就糊了），把角度和平移调小 |
| 学习率调大后准确率掉到 10% | 步子太大直接崩，先降到 0.002 左右 |
| 下载 yolov8n.pt 超时 | $env:HTTPS_PROXY="http://127.0.0.1:7897"（端口换成你代理软件的）后再运行一次 |
| 只有几十张图，mAP 一直是 0 | 别从零训练：实测 30 张图从零是 0，换预训练权重到 0.706 |
| 多进程 / PermissionError（Windows） | train 时写 workers=0 |
| 检测 mAP 一直是 0 | 标签格式问题：文件名要一一对应，坐标必须归一化到 0~1 |
| 检测框偏大偏小 | 标注不统一，抽 10 张把标签画回图上核对 |

## 套用顺序口诀

传统视觉：读图 → 拆颜色（或转 HSV）→ 造黑白掩码 → 去噪 → 找轮廓 → 过滤小物体 → 画框 / 量面积 / 求旋转框 → 像素换算毫米 → 保存。

机器学习：读数据 → 切训练/测试 → 选模型 → fit → predict → 打分 → 看混淆矩阵 → 调参数或加数据 → 存模型。

目标检测：拍图或造图 → 标注成 YOLO 格式 → 划分数据 → 训练 → 看 mAP/PR 曲线 → 推理出框 → 把框中心接进毫米换算和指令表。

定位物体先问三句话：外观固定吗（→ 模板匹配）？会转、有纹理吗（→ ORB）？是纯色、颜色稳定吗（→ 颜色 + 轮廓）？

## 这本字典怎么维护（别让它变成垃圾堆）

一个模板四段式：什么时候用 / 代码 / 实测数字 / 常见坑。缺一段就不算整理完。

新模板从 30 号接着排，不插队改旧编号——课程文档里写的“模板 15”要永远指同一个东西。

抄进来的代码必须是跑通的版本，缩进完整（缩进丢了的代码等于没抄）。

学了新函数就加进“附加小抄”，写了新实验就把实测数字更新掉旧数字。

每次大改之前先复制一份备份，文件名带日期（例如 常用代码字典_备份_2026-09-18.docx）。
