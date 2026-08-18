

# 蒙特卡洛路径追踪软渲染器开发

## 构建

### 前置条件

1. **VS 2022 或 Build Tools**（免费）：[下载](https://visualstudio.microsoft.com/zh-hans/downloads/)，安装时勾选"使用 C++ 的桌面开发"
2. **CMake**（>= 3.21）：装 VS 时自带，或单独 [下载](https://cmake.org/download/)
3. **OpenCV 运行时 DLL**：从 OpenCV 4.10.0 Windows 版 `x64/vc16/bin/` 中复制到仓库 `lib/` 目录：
   - `opencv_world4100.dll`（Release，必须）
   - `opencv_world4100d.dll`（Debug，可选）
4. **HDRI 环境贴图**：`model/HDRI/room.jpg`（~28MB，不在仓库中，需单独获取）

### 构建 & 运行

```bash
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

（仓库中的 `CMakePresets.json` 提供了 `msvc` 配置预设，可用 `cmake --preset msvc` 代替上面的 configure 命令。）

exe 在 `build/Release/PathTracing.exe`。**必须在 `PathTracing/` 目录下运行**（源码中的模型路径以此为准）：

```bash
cd PathTracing
../build/Release/PathTracing.exe
```

输出图片在 `SHOW.assets/`。

### IDE 打开

- **VSCode**：装 CMake Tools + C/C++ 扩展，打开仓库根目录即可
- **VS 2022**：文件 → 打开 → CMake → 选 `CMakeLists.txt`，顶部下拉选 Release

---

## CUDA 加速

项目支持 GPU 加速（cuda 分支），RTX 5060 Laptop 实测：

| 场景 | CPU | GPU | 加速比 |
|------|-----|-----|--------|
| 墙体灯光（~30 tris） | 41.7s | 0.75s | 55× |
| 球体 | 78.3s | 1.2s | 65× |
| 复杂 OBJ（5874 tris） | 136.7s | 5.4s | 25× |

核心优化：
1. **场景级→三角级 BVH**：叶子从 ~300 三角降至 ≤4，消除 warp 线程发散
2. **NEE 预检查消除**：以状态标记替代冗余遍历，每 bounce 减少 1/3 的 BVH 遍历次数
3. **SAH 空间划分**：以表面积启发式替代中点切分，节点空间分布更均匀
4. **三角形 SOA 布局**：冷热数据分离，求交时仅加载必需字段

累计 **38× 加速**（206s → 5.4s）。切换：`main.cpp` 中 `useGPU = 1`（GPU）/ `0`（CPU）。

---

此渲染器是学习性质，使用纯 C++ 开发，CPU 计算渲染（纹理读取使用 `openCV`）。

演示视频：

https://www.bilibili.com/video/BV1D5NNeiE5G/?spm_id_from=333.1387.homepage.video_card.click



### 10

- 实现基础路径追踪渲染管线，暂未实现MVP变换部分，目前是移动相机和静态场景
- 1.Diffuse材质，均匀将光线反射至上半球面；适配平面，球体，和OBJ模型以及OBJ场景
- 2.微表面材质 ，根据材质的折射率和粗糙度决定BRDF的值；适配平面，球体，和OBJ模型以及OBJ场景
- 3.全镜面反射材质，在一个平面下根据法线计算反射方向，实现渲染镜面材质；适配平面，球体，和OBJ模型以及OBJ场景
- 4.折射透射材质，适配平面，球体，和OBJ模型
- 5.Diffuse_Specular材质
- 实现球体，平面 ，面光源，并且支持以上材质，

### 11

- 基于BVH加速树结构和AABB包围盒，将三角形和网格划分成树形，查找时减少求交点遍历三角形或物体`Mesh`的个数

### 12

- 实现基于`thread`和`mutex`的多线程加速，进一步加速`cpu`渲染

release模式：

1.    BVH+多线程   200*1spp ： 110ms
2.    BVH+单线程   200*1spp ： 450ms
3.    遍历+多线程   200*1spp ： 100ms **（可能是BVH叶子结点放的三角形太少，导致性能没有遍历优，一个坑以后再来补，或者有大佬pr 也是可以的owo）**
4.    遍历+单线程   200*1spp ： 350ms



- 实现三角形`Triangle`类



#### 1.13

- 实现`MeshTriangle` 结合`Triangle`和 `OBJ_Loader`实现导入任意OBJ格式模型
- 适配了OBJ格式模型的diffuse ，反射 ，微表面材质

#### 1.14

- 实现**平滑着色** ，通过对三角顶点的法线做线性插值，实现了任意**平滑OBJ物体**的平滑着色和垂直着色的切换

| ![image-20250114150456749](SHOW.assets/image-20250114150456749.png) | ![image-20250114150506243](SHOW.assets/image-20250114150506243.png) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 法线差值 - diffuse材质-spp5                                  | 非法线差值 - diffuse材质-spp5                                |
| ![image-20250114151637742](SHOW.assets/image-20250114151637742.png) | ![image-20250114151622657](SHOW.assets/image-20250114151622657.png) |
| 非法线差值-镜面材质-spp5                                     | 法线差值-镜面材质-spp5                                       |
| ![image-20250118131019277](SHOW.assets/image-20250118131019277.png) | ![image-20250118130940291](SHOW.assets/image-20250118130940291.png) |
| 非法线差值的兔子                                             | 法线差值的兔子                                               |





#### 1.15

- 修正`Boll 和 Mesh 和 Plane`的双面法线问题，实现Mesh的透射和平滑透射

| <img src="SHOW.assets/image-20250115195434249.png" alt="image-20250115195434249" style="zoom:94%;" /> | ![image-20250115195523858](SHOW.assets/image-20250115195523858.png) | ![image-20250118134514369](SHOW.assets/image-20250118134737673.png) |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| OBJ棱球，垂直着色透射 ior = 1.9                              | OBJ棱球，平滑着色透射  ior = 1.9                             | 平滑着色透射的兔子  ior = 1.3                                |

#### 1.17

**材质加载方式新增**

更新了Mesh模型的材质加载的两种方式

1. 初始化Mesh时直接指定材质种类
2. 初始化Mesh时不指定材质种类，而是根据MTL文件的参数决定着色方式



**实现读取MTL材质**

实现原理化BSDF到MTL材质参数的初步确定，着色方式如下：

- 基础色：漫反射颜色，Kd的值
- 糙度变大：高光强度不变 ，是光的集中程度变小
- 金属度变小：高光集中程度不变，但是高光变淡
- 折射率：结果就是Ni的值
- illum ：如果这个是illum 2为漫反射，illum 3为 diffuse-specular 材质， 他们之间的比例由Ns决定

#### 1.18

**实现使用diffuse-specular材质近似MTL材质**

重构了`MeshTriangle   Material     `和 `Scene::PathTracing`使用diffuse和reflect材质结合实现diffuse-specular材质来近似OBJ模型自带的材质

| ![image-20250118115302009](SHOW.assets/image-20250118115302009.png) | ![image-20250118114825072](SHOW.assets/image-20250118114825072.png) | ![image-20250118115606602](SHOW.assets/image-20250118115606602.png) |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| diffuse-specular光滑度0.1                                    | diffuse-specular光滑度0.5                                    | diffuse-specular光滑度0.8                                    |

**实现随机采样抗锯齿**

每一个像素多重采样时，相机射出去的光线在方向上都给一个小的偏移，最后所有spp采样的结果取平均估值实现反走样的效果

| ![image-20250118152304183](SHOW.assets/image-20250118152304183.png) | >    | ![image-20250118152145446](SHOW.assets/image-20250118152145446.png) |
| ------------------------------------------------------------ | ---- | ------------------------------------------------------------ |
| 无SAA锯齿                                                    | >    | SAA反走样                                                    |

**实现UV映射纹理**

重构了Scene ， Material ， Object

根据三角形的顶点的UV坐标差值得出击中点的`uv`坐标，通过`openCV`读取图片对应像素颜色，实现纹理映射

| <img src="SHOW.assets/image-20250120142751765.png" alt="image-20250120142751765"  /> | ![image-20250120143106948](SHOW.assets/image-20250120143106948.png) |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| UV纹理下的光滑奶牛 - `spp` -200                              | UV纹理下的Box -` spp` - 60                                   |

#### 2.9 

对`Object,MeshTraingle,BVHStruct , main ` 进行修改，修复了内存泄露问题
