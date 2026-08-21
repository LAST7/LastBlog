---
title: GAMES101 [1] Affine Transformation & Projection
date: 2026-08-22 12:13:24
categories: 笔记
tags:
    - games101
    - linear_algebra
excerpt: GAMES101 图形学中的基础变换和投影
mathjax: true
---

# 仿射变换 Affine Transformation

## 线性变换 Linear Transformation

- 本质上，二维空间中某一个点/向量的旋转和缩放都是对其坐标值进行某种线性变换，因此二维空间图形的线性变换可以通过一个二维方阵来实现。

    $$
    P' = MP
    $$

### 旋转 Rotation

- 对于旋转，可以取两个特殊点来简单推导旋转变换矩阵（类似于用基向量变换的方式来理解），$(1, 0)$ 和 $(0, 1)$，逆时针旋转 $\theta$ ：

<img src="https://files.seeusercontent.com/2026/08/21/m9Tx/rotation.png" alt="rotation.png" width="500" />

- 观察图像可知：

    $$
    (1, 0) \to (\cos{\theta}, \sin{\theta})
    $$

    $$
    (0, 1) \to (-\sin{\theta}, \cos{\theta})
    $$

- 由此可以写出变换矩阵 $M$ ：

    $$
    \begin{pmatrix}
    x' \\\\
    y'
    \end{pmatrix} =
    \begin{pmatrix}
    \cos{\theta} & -\sin{\theta} \\\\
    \sin{\theta} & \cos{\theta}
    \end{pmatrix}
    \begin{pmatrix}
    x \\\\
    y
    \end{pmatrix}
    $$

### 缩放 Scale

- 同理可得缩放矩阵 $M$ ：

    $$
    \begin{pmatrix}
    ax \\\\
    by
    \end{pmatrix} =
    \begin{pmatrix}
    a & 0 \\\\
    0 & b
    \end{pmatrix}
    \begin{pmatrix}
    x \\\\
    y
    \end{pmatrix}
    $$

## 平移变换 & 齐次坐标 Translation & Homogeneous Coordinates

- 由于平移变换并非线性变换，因此无法使用一个与空间维度同维的方阵来表示。解决方法是给变换矩阵添加一个新的维度，也就是引入齐次坐标 Homogeneous coordinates，以此来为坐标值增加平移量。

- 在二维里，我们把点写成三维的形式：

    $$
    \begin{pmatrix}
    x \\\\
    y \\\\
    1
    \end{pmatrix}
    $$

- 如此一来，我们就能使用一个 $3 \times 3$ 的矩阵同时表示线性变换和平移变换。
    - 以向上平移 1 个单位，向右平移 2 个单位的二维变换矩阵为例：

        $$
        \begin{pmatrix}
        x + 2 \\\\
        y + 1 \\\\
        1
        \end{pmatrix} =
        \begin{pmatrix}
        1 & 0 & 2 \\\\
        0 & 1 & 1 \\\\
        0 & 0 & 1
        \end{pmatrix}
        \begin{pmatrix}
        x \\\\
        y \\\\
        1
        \end{pmatrix}
        $$

- 将矩阵的第三列写为 $\begin{pmatrix} 0 \\\\ 0 \\\\ 1 \end{pmatrix}$ 意味着图像在变换中没有发生平移，左上角 $2 \times 2$ 的分块矩阵仍表示二维图形的线性变换。

## 齐次坐标表示法的妙用

- 齐次坐标表示法中，第三个新填入的值在表示“点”时写为 $1$，而表示“向量”时写为 $0$。这是一个非常巧妙的设计，原因如下：
- 我们希望点和向量在进行变换的前后遵循如下规则：
    1. 点 + 向量 = 点
    2. 向量 + 向量 = 向量
    3. 点 - 点 = 向量
    4. 点 + 点 = 中点（**非一般规则**）

- 上述的规则刚好满足这些要求。
    - 例如，点与点相减得到向量（第三项为 $0$）：

        $$
        \begin{pmatrix}
        a \\\\
        b \\\\
        1
        \end{pmatrix} -
        \begin{pmatrix}
        c \\\\
        d \\\\
        1
        \end{pmatrix} =
        \begin{pmatrix}
        a - c \\\\
        b - d \\\\
        0
        \end{pmatrix}
        $$

- 此外还有一点：仅代表方向的向量不受平移变换的影响：

    $$
    \begin{pmatrix}
    1 & 0 & 2 \\\\
    0 & 1 & 1 \\\\
    0 & 0 & 1
    \end{pmatrix}
    \begin{pmatrix}
    a \\\\
    b \\\\
    0
    \end{pmatrix} =
    \begin{pmatrix}
    a \\\\
    b \\\\
    0
    \end{pmatrix}
    $$

- 这也是非常符合直觉和数学定义的。

## 齐次坐标表示法的比例不变性与透视直觉

- 对于一个使用齐次坐标表示法表示的二维平面上的点：

    $$
    \begin{pmatrix}
    X \\\\
    Y \\\\
    W
    \end{pmatrix}, (W \neq 0)
    $$

- 若将其整体乘以一个非零常数，结果仍然表示相同的点：

    $$
    \begin{pmatrix}
    X \\\\
    Y \\\\
    W
    \end{pmatrix}
    \sim
    \begin{pmatrix}
    kX \\\\
    kY \\\\
    kW
    \end{pmatrix}, (k \neq 0)
    $$

- 这是一个非常有趣的性质，因为这在某种程度上意味着 **齐次坐标表示法所表示的点，是一条从视口中心出发的射线**。
    - 当然，这条射线不是这个点所在的二维平面上的射线，而是为了描述射影点而引入的几何解释。换言之，观察一张照片时，大脑想象出的“深度空间”（射影空间）中，由摄像机出发的一条射线。
    - 严格来说，在由 $(X, Y, W)$ 构成的抽象三维空间中，所有形如 $(kX,kY,kW)$ 的点构成了一条穿过坐标原点的射线。选择 $W=1$ 平面作为截面，就能得到普通二维坐标 $(X/W,Y/W)$。

- 我们可以用三维空间中的透视投影来建立一种直觉上的类比，如图：

<img src="https://files.seeusercontent.com/2026/08/21/2csI/affine_projection.png" alt="affine_projection.png" width="500" />

- 我们假设摄像机中心位于坐标原点，观察方向为 $-z$，成像平面位于相机前方的某个固定深度，例如 $z=-d$，观察上图可以发现，对于一个处于 $xy$ 平面坐标原点，面朝 $-z$ 方向的观察者而言，$P(a, b, -c)$ 与 $P'(2a, 2b, -2c)$ 这两个点在其视野中事实上是 **重合** 的，它们在针孔相机中会投影到同一个图像位置。

- 换言之，这两个点在相机视野里是等价的（投影到同一个二维图像位置），它们的大小比例只取决于距离相机的深度（近大远小）。

- 这就是为什么齐次坐标表示法被设计成如此，甚至于它的名字“齐次”本身就意味着“按同一比例变化”。其多出的一个维度不仅解决了平移变换的矩阵乘法表示，其比例不变性更是天然为透视投影与消除距离缩放而生的数学工具。

---

{% callout type="primary" icon="fa-solid fa-lightbulb" title="SomeThoughts" %}

- 或许这样巧妙契合视觉的性质会让人心生感慨，感叹数学的精妙乃至于相信造物主的存在。但事实上这仅仅只是一种“必然的巧合”，即设计好的巧合。究其原因就在于我们人类对于这个世界特定频段波（可见光）的感知方式：视觉，它决定了古代数学家在描述三维空间到二维平面的透视投影时，会创造出什么样的数学工具。

- 事实上，所谓计算机图形学（起码在光栅化渲染领域）也就是模拟人眼/摄像机对于三维世界所发出和反射的光的感知方式。我们“看”到的并不是几何本体，我们的双眼呈现给大脑的信息本身就已经是“投影”的结果。就好比所谓的“洞穴寓言”，我们所看到的一切都只是某种投影所衍生的幻象。

- 设想一下，如果我们可以通过某种神奇的“新型视觉”直接感知距离以及穿透物体，齐次坐标表示法“等比例缩放不变性”还会显得如此自然且必要吗？显然不会。对我们的“新型视觉”来说，$P(a, b, -c)$ 和 $P'(2a, 2b, -2c)$ 将不会被视作“重合/等价”——就如同他们的坐标看上去的那样，它们就是真实三维空间中两个距离迥异的点。齐次坐标表示法的等比例缩放不变性对这种“新视觉”可以说是毫无作用。

- 简而言之，高一维度的齐次坐标所具备的等比例缩放不变性，本质上是因为我们的视觉天然压缩了距离。因此是人类主动选择为齐次坐标增添“等比例缩放不变的性质”以表示投影平面的点坐标，而不是反过来。

{% endcallout %}

## 三维空间中的仿射变换矩阵

- 旋转：
    - 绕 X 轴：

        $$
        \begin{pmatrix}
        1 & 0 & 0 & 0 \\\\
        0 & \cos{\theta} & -\sin{\theta} & 0 \\\\
        0 & \sin{\theta} & \cos{\theta} & 0 \\\\
        0 & 0 & 0 & 1
        \end{pmatrix}
        $$

    - 绕 Y 轴：

        $$
        \begin{pmatrix}
        \cos{\theta} & 0 & \sin{\theta} & 0 \\\\
        0 & 1 & 0 & 0 \\\\
        -\sin{\theta} & 0 & \cos{\theta} & 0 \\\\
        0 & 0 & 0 & 1
        \end{pmatrix}
        $$

    - 绕 Z 轴：

        $$
        \begin{pmatrix}
        \cos{\theta} & -\sin{\theta} & 0 & 0 \\\\
        \sin{\theta} & \cos{\theta} & 0 & 0 \\\\
        0 & 0 & 1 & 0 \\\\
        0 & 0 & 0 & 1
        \end{pmatrix}
        $$

- 缩放和平移与二维同理，此处略过。

# 视图变换

- 一般约定摄像机需位于三维坐标的原点，面朝 $-z$ 方向。视图变换的过程就是将一个场景中任意摆放的摄像机位移到约定的位置。

- 一般使用如下三个参数定义摄像机在世界坐标空间中的位置：
    - $e$: eye position
    - $\hat{g}$: gaze direction
    - $\hat{t}$: top direction

    {% callout type="primary" icon="fa-solid fa-circle-exclamation" title="NOTE" %}
    $\hat{x}$ 上的尖角表示这是一个 **单位向量**
    {% endcallout %}

- 要想写出一个变换矩阵，使得摄像机所在的空间变换到我们先前的约定方式，也就是：

    > 一般约定摄像机需位于三维坐标的原点，面朝 $-z$ 方向

- 这是相当困难的，因为摄像机很可能处于一种非标准的状态，即位置不位于原点，$\hat{g}$ 和 $\hat{t}$ 可能朝向非坐标轴方向。想要简单的写出一个能将他们全部变换到约定形式的矩阵非常困难，因此我们想到可以利用矩阵在空间变换中的 **可逆** 性质来倒推变换矩阵。

- 即 **写出使得位于三维坐标的原点，面朝 $-z$ 方向，顶部朝向 $y$ 方向的摄像机，变换为当前摄像机位置的变换矩阵，而后求该矩阵的逆矩阵**。

    $$
    R_{view}^{-1} =
    \begin{pmatrix}
    x_{\hat{g} \times \hat{t}} & x_{t} & x_{-g} & 0 \\\\
    y_{\hat{g} \times \hat{t}} & y_{t} & y_{-g} & 0 \\\\
    z_{\hat{g} \times \hat{t}} & z_{t} & z_{-g} & 0 \\\\
    0 & 0 & 0 & 1
    \end{pmatrix}
    $$

- 注意到旋转变换矩阵是正交矩阵，因此：

    $$
    R_{view} = (R_{view}^{-1})^{-1} = (R_{view}^{-1})^{T} =
    \begin{pmatrix}
    x_{\hat{g} \times \hat{t}} & y_{\hat{g} \times \hat{t}} & z_{\hat{g} \times \hat{t}} & 0 \\\\
    x_{t} & y_{t} & z_{t} & 0 \\\\
    x_{-g} & y_{-g} & z_{-g} & 0 \\\\
    0 & 0 & 0 & 1
    \end{pmatrix}
    $$

{% callout type="info" icon="fa-solid fa-circle-question" title="TODO" %}
为什么旋转变换矩阵一定是正交矩阵？
{% endcallout %}

- 当然，还有平移变换矩阵：

    $$
    T_{view} =
    \begin{pmatrix}
    1 & 0 & 0 & -e_{x} \\\\
    0 & 1 & 0 & -e_{y} \\\\
    0 & 0 & 1 & -e_{z} \\\\
    0 & 0 & 0 & 1
    \end{pmatrix}
    $$

- 如此一来，视图变换就完成了：

    $$
    M_{view} = R_{view}T_{view}
    $$

# 投影变换

{% callout type="warning" icon="fa-solid fa-triangle-exclamation" title="**注意**" %}

在进行投影变换前，应当先 **应用视图变换，进入以相机为中心的坐标系**。换言之，将摄像机置于坐标原点，摄像机面朝的方向是 $-z$，

{% endcallout %}

## 正交投影 Orthographic Projection

- 对于正交投影，主要的步骤是将摄像机前方的可视区域变换成一个 **标准立方体**，即在三维都在 -1 ~ 1 之间的立方体空间。

- 对于摄像机坐标系，一般情况下其正交可视区域已经天然关于 Z 轴对称。考虑到视角裁剪和部分渲染的情况，仍然需要将可视区域视作一个可能存在于任何位置的长方体。

<img src="https://files.seeusercontent.com/2026/08/22/piU6/otho.png" alt="otho.png" width="600" />

- 因此，我们需要将其平移到坐标原点，然后缩放为标准立方体（canonical cube），然后进行视口变换和光栅化。

- 变换矩阵：

    $$
    M_{ortho} =
    \begin{pmatrix}
    \frac{2}{r - l} & 0 & 0 & 0 \\\\
    0 & \frac{2}{t - b} & 0 & 0 \\\\
    0 & 0 & \frac{2}{n - f} & 0 \\\\
    0 & 0 & 0 & 1
    \end{pmatrix}
    \begin{pmatrix}
    1 & 0 & 0 & -\frac{r + l}{2} \\\\
    0 & 1 & 0 & -\frac{b + t}{2} \\\\
    0 & 0 & 1 & -\frac{n + f}{2} \\\\
    0 & 0 & 0 & 1
    \end{pmatrix}
    $$

{% callout type="primary" icon="fa-solid fa-circle-question" title="NOTE" %}
先平移后缩放
{% endcallout %}

## 透视投影 Perspective Projection

{% callout type="warning" icon="fa-solid fa-triangle-exclamation" title="**注意**" %}

- 近远平面变量 $zNear,zFar$ 分别表示它们在相机空间中的带符号 $z$ 坐标，因此：

    $$
    zFar < zNear < 0
    $$

{% endcallout %}

- 对于透视投影，摄像机面朝的可视区域不再是一个长方体，而是一个 **截头四棱锥**。投影的思路是先将这个截头四棱锥变换成为正交投影中，被转换为标准立方体之前的长方体可视区域，然后再进行正交投影。 因此透视投影本质上就是在 $M_{ortho}$ 右侧乘一个新的变换矩阵 $M_{persp\\_to\\_ortho}$（先将截头四棱锥变换为正交可视区域，也就是那个长方体）。

<img src="https://files.seeusercontent.com/2026/08/22/piU6/otho.png" alt="otho.png" width="600" />

<div style="margin: 10px"></div>

<img src="https://files.seeusercontent.com/2026/08/22/q0pR/persp.png" alt="persp.png" width="600" />

{% callout type="info" icon="fa-solid fa-triangle-exclamation" title="NOTE" %}
上图有些许的不准确，因为透视投影中，截头四棱锥的近截面应该在 $z = zNear$ 上，而不是 $xy$ 平面。
{% endcallout %}

- 观察上图可知：
    1. 截头四棱锥中任意垂直于 $z$ 轴的截面，都将被缩放到 $z = zNear$ 的平面大小
    2. $z = zNear$ 和 $z = zFar$ 处的平面，变换前后的齐次坐标在完成齐次除法之后，其三维坐标中的深度仍然是 $zNear$ 和 $zFar$

- 可以通过一个简单的剖面图理解这种变换的规律：

<img src="https://files.seeusercontent.com/2026/08/22/ySt5/persp_to_otho.png" alt="persp_to_otho.png" width="500" />

- 显然，对于平面 $z_{k}, (z_{k} \in [zFar, zNear])$ 上的点 $P(x, y, z_{k})$ 来说，有：

    $$
    \frac{x'}{x} = \frac{y'}{y} = \frac{zNear}{z_{k}}
    $$

- 对于该平面，缩放变换矩阵为：

    $$
    \begin{pmatrix}
    \frac{zNear}{z_{k}} & 0 & 0 & 0 \\\\
    0 & \frac{zNear}{z_{k}} & 0 & 0 \\\\
    ? & ? & ? & ? \\\\
    0 & 0 & 0 & 1
    \end{pmatrix}
    $$

- 由于我们希望得到的是一个对于整个可视区域通用的变换矩阵，而上面这个矩阵仅作用于平面 $z = z_{k}$。对此我们可以利用齐次坐标表示法的等比例缩放不变性，将变换关系转换为：

    $$
    \begin{pmatrix}
    x \\\\
    y \\\\
    z \\\\
    1
    \end{pmatrix} \to
    \begin{pmatrix}
    zNear \cdot \frac{x}{z} \\\\
    zNear \cdot \frac{y}{z} \\\\
    ? \\\\
    1
    \end{pmatrix} \sim
    \begin{pmatrix}
    zNear \cdot x \\\\
    zNear \cdot y \\\\
    ? \\\\
    z
    \end{pmatrix}
    $$

- 对于这样的变换，其变换矩阵可以很容易的写出：

    $$
    \begin{pmatrix}
    zNear & 0 & 0 & 0 \\\\
    0 & zNear & 0 & 0 \\\\
    ? & ? & ? & ? \\\\
    0 & 0 & 1 & 0
    \end{pmatrix}
    $$

- 对于该矩阵第三行的推导，我们同样可以使用先前推导旋转变换矩阵那样，选择两个特殊平面：

    $$
    \begin{pmatrix}
    x \\\\
    y \\\\
    zNear \\\\
    1
    \end{pmatrix} \to
    \begin{pmatrix}
    x \\\\
    y \\\\
    zNear \\\\
    1
    \end{pmatrix} \sim
    \begin{pmatrix}
    x \cdot zNear \\\\
    y \cdot zNear\\\\
    zNear^{2} \\\\
    zNear
    \end{pmatrix}
    $$

    和

    $$
    \begin{pmatrix}
    x \\\\
    y \\\\
    zFar \\\\
    1
    \end{pmatrix} \to
    \begin{pmatrix}
    x \cdot \frac{zNear}{zFar} \\\\
    y \cdot \frac{zNear}{zFar} \\\\
    zFar \\\\
    1
    \end{pmatrix} \sim
    \begin{pmatrix}
    x \cdot zNear \\\\
    y \cdot zNear \\\\
    zFar^{2} \\\\
    zFar
    \end{pmatrix}
    $$

- 注意到变换后结果的第三分量与 $x, y$ 无关，由此可以判断变换矩阵的第三行前两项均为 $0$。设剩余两项分别为 $A, B$，有：

    $$
    \begin{cases}
    \begin{pmatrix}
    0 & 0 & A & B \\\\
    0 & 0 & 1 & 0
    \end{pmatrix}
    \begin{pmatrix}
    x \\\\
    y \\\\
    zNear \\\\
    1
    \end{pmatrix} =
    \begin{pmatrix}
    zNear^{2} \\\\
    zNear
    \end{pmatrix} \\\\
    \begin{pmatrix}
    0 & 0 & A & B \\\\
    0 & 0 & 1 & 0
    \end{pmatrix}
    \begin{pmatrix}
    x \\\\
    y \\\\
    zFar \\\\
    1
    \end{pmatrix} =
    \begin{pmatrix}
    zFar^{2} \\\\
    zFar
    \end{pmatrix}
    \end{cases}
    $$

- 得到方程组及其唯一解：

    $$
    \begin{cases}
    A \cdot zNear + B = zNear^{2} \\\\
    A \cdot zFar + B = zFar^{2}
    \end{cases} \implies
    \begin{cases}
    A = zNear + zFar \\\\
    B = -zNear \cdot zFar
    \end{cases}
    $$

- 至此，我们就得到了 $M_{persp\\_to\\_ortho}$：

    $$
    M_{persp\\_to\\_ortho} =
    \begin{pmatrix}
    zNear & 0 & 0 & 0 \\\\
    0 & zNear & 0 & 0 \\\\
    0 & 0 & zNear + zFar & -zNear \cdot zFar \\\\
    0 & 0 & 1 & 0
    \end{pmatrix}, (zFar < zNear < 0)
    $$

- 也就有：

    $$
    M_{persp} = M_{ortho}M_{persp\\_to\\_ortho}
    $$

### Z 轴深度非线性挤压

- 注意到应用 $M_{persp\\_to\\_ortho}$ 前后，摄像机可视区域内点集的深度值变化并不均匀。

- 具体来说，设可视区域内某一点

    $$
    P_{mid} =
    \begin{pmatrix}
    x_{P} \\\\
    y_{P} \\\\
    z_{mid} \\\\
    1
    \end{pmatrix} , z_{mid} = \frac{zFar + zNear}{2}
    $$

- 经由 $M_{persp\\_to\\_ortho}$ 变换后：

    $$
    P_{mid}' = M_{persp\\\_to\\\_ortho} \cdot P_{mid} =
    \begin{pmatrix}
    x_{P} \cdot zNear \\\\
    y_{P} \cdot zNear \\\\
    z_{mid}(zNear + zFar) - zNear \cdot zFar \\\\
    z_{mid}
    \end{pmatrix}
    $$

- 齐次除法：

    $$
    \begin{pmatrix}
    x_{P} \cdot \frac{zNear}{z_{mid}} \\\\
    y_{P} \cdot \frac{zNear}{z_{mid}} \\\\
    zNear + zFar - \frac{zNear \cdot zFar}{z_{mid}} \\\\
    1
    \end{pmatrix}
    $$

- 得到新的深度 $z_{mid}' = zNear + zFar - \frac{zNear \cdot zFar}{z_{mid}}$，与 $z_{mid}$ 相减：

    $$
    z_{P}' - z_{mid} = \frac{(zNear - zFar)^{2}}{2(zNear + zFar)} < 0
    $$

- 可知：$P_{mid}$ 在变换后，其相距坐标原点的距离增大，即摄像机可视区域中的点集在变换后，被整体推向远平面挤压。

# End
