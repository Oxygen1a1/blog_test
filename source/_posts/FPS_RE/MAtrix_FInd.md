---
title: FPS RE
date: 2023-07-23
---

# chap01 基础知识

## 基础名词

![image-20230519164438762](/img/lanping.jpg)

- **pitch(仰角) 又称X轴 左轴**

​	这是FPS游戏的一个概念,通常是指仰角,再`aim bot`的时候，是需要修改的;一般地,当FPS游戏你把准星放置在正中心,pitch一般是0°,而如果一直到最下面,一般是-90°,反之亦然,总共是180°

- **yaw(水平旋转 Y轴)**

​	一般来说,这个角度是0-360°,但是一旦到360,就会立刻转变成0°

- **roll(前轴 Z轴)**

- **mvp矩阵**

​	矩阵通常是线性代数的一个概念,通常是用来表示向量的,但是在计算机图像学以及FPS的相关概念中,矩阵通常用于游戏内的坐标到屏幕坐标的变化;

而在3D游戏中,矩阵一般涉及下面四种

- **模型矩阵(Model Matrix)**：负责将对象的局部坐标（例如，模型的顶点）转换到世界坐标系统。换句话说，Model矩阵将对象放置在3D世界中的正确位置和方向。

无论如何,模型矩阵是描述模型的,也就是说这个矩阵里面包含了模型的xyz 以及模型的旋转角度,比如下面这个

![image-20230522084053006](/image/1.jpg)

那他的模型矩阵实际上是

![image-20230522084407488](/image/1.png)

可以看到,模型矩阵实际上最后一列就是他的xyz坐标,这便是矩阵运算算法的关键;请务必记住,Model矩阵是描述了物体**(可以把他想象成FPS的敌人)**的旋转姿态和方位

**事实上,我们再进行矩阵运算的时候,伪造的ModelMatrix实际上是忽略了他的旋转的,也就是说,前三列都是0，只有最后一列是xyz,因此再`列主`的矩阵(OpenGl)中,在得pv矩阵的时候,就是直接和伪造的这个矩阵相乘,即可得到其次化坐标;**

- **视图矩阵(View Matrix)**：表示摄像机（或观察者）在世界空间中的位置和方向。View矩阵将世界坐标系转换为相机坐标系，也就是从观察者的视角看世界。

**这个就是自己的矩阵**,它描述了摄像机(也就是自己眼里所看到的)的xyz以及旋转,和`Model`矩阵类似,如果摄像机没有旋转,那么实际上他的最后一列和Model矩阵完全一样,也是表示xyz的;

![image-20230522090252230](/image/image-20230522090252230.png)

![image-20230522090258898](/img/image-20230522090258898.png)

前面我们知道,`Model`矩阵最后一列的xyz并不会受到旋转角度的影响,但是view矩阵会;

==他和Model矩阵相乘,就能得到**归零坐标**,所谓归零坐标,就是说两个相对的物体,让一个物体的坐标变成(0,0,0),而另一个物体相对位置不变,另一个物体的坐标;==

比如下面的计算

在不考虑任何旋转的角度下,摄像机的坐标是(0,0,10),物体坐标是(0,0,0)

![image-20230522090452494](./assets/image-20230522090452494.png)

然后将这两个矩阵进行计算

![image-20230522090612049](./assets/image-20230522090612049.png)

可以发现,最终得到的`ModelView`矩阵,他的最后一列是(0,0,10,1),最后一个表示缩放;这便是把摄像机平移到000归零后,物体相对摄像机的坐标;当然这是不考虑物体旋转,如果考虑旋转,那么则更加复杂;

**得到物体的归零坐标,那么接下来和映射矩阵相乘,就可以得到齐次坐标了**

- **投影矩阵(Projection Matrix)**：负责将3D世界坐标投影到2D屏幕坐标。有两种主要的投影方式：正交投影（Orthographic Projection）和透视投影（Perspective Projection）。

​	![image-20230522091040332](./assets/image-20230522091040332.png)

![image-20230522091047844](./assets/image-20230522091047844.png)

![image-20230522091053253](./assets/image-20230522091053253.png)

这就是投影矩阵的计算,可以发现,投影矩阵之和`视角`有关,也就是人眼能看到的这个上下宽高;

因此,投影矩阵之和`FOV`有关;

然后,上面的矩阵都是计算机图形学的重点,对于学习逆向意义不大,我们只需要知道,上述三个矩阵连乘,会得到一个新的矩阵,叫做**模型-视图-投影矩阵(Model-View-Projection Matrix)，**用来一次性将物体从模型空间转换到屏幕空间。事实上,这个矩阵也叫做`MVP矩阵`

### 矩阵算法

事实上,矩阵算法大概有几种,但是不多,主要区别是Dx和OpenGl;

`基于dx就是[x,y,z,1]*(view*project)这样算`

其中Dx的算法是`model*view*projecg`,可以得到一个4*4矩阵,这个矩阵的最后一行是这种形式

```C++
x,x,x,x;
x,x,x,x;
x,x,x,x;
x',y',z',w'
```

其中x,y,z,w叫做`齐次坐标`,然后同时/w

在最终进行屏幕映射之前，我们会将`(x', y', z')`除以`w'`来得到这是在所谓的裁剪空间或规范化设备坐标（Normalized Device Coordinates，简称NDC。这个过程叫做透视除法（Perspective Division）。z'/w'一般表示深度,一般不用;

然后接下来使用**视口变换（Viewport Transform）**

下面是一个基于Dx的WorldToScreen:

![image-20230522100848094](./assets/image-20230522100848094.png)

XJz就是`w`,此外,Dx的变换中,有一个特殊情况,那就是y z颠倒

![image-20230522100949333](./assets/image-20230522100949333.png)

至于OpenGl,他是列主的矩阵,因此他的算法是反过来的

`project*view*model*v`,这样的话,就是行*列了;

## 基质和偏移的寻找

事实上,所谓的基质就是固定不变的地址,当然,进程开了随机基质,这个所谓的基质是变动的,但是他相对于模块基质是不会变动的,这个所谓的`基质`,其实更贴切来说就是一个全局变量,这个全局变量一般存储这重要的信息;

我们知道,游戏的编写一遍是基于OOP思想的,因此全局变量可能会有`World`,`Game`,`Player`这些;

下面使用CS:GO来进行寻找`player`这个全局变量的基质,有了基质,游戏再次打开的时候,就会定位到想要的东西,事实上,其他的东西,比如人物坐标,矩阵这些东西,都是存在于这些全局变量中的某个`偏移`**,这里所提到的偏移,可以指为了定位基质所加上的offset,也可以指结构体偏移;**

此外,因为一般来说,上述的`World`什么结构,里面的矩阵,人物坐标一般都是指针，是动态申请的,因此想要使用基质定位到矩阵 人物血量是不显示的,所以一般不会使用基质去定位这个的;

因此可以通过两种方式找到`基质`和`偏移`

- 通过CE搜索

通过CE搜索人物血量,找到人物血量的地址,找到地址之后,注意这个地方实际上是Player血量的信息,但是毫无疑问,他肯定不是一个基质,

![image-20230521083836836](./assets/image-20230521083836836.png)‘

找到之后,右键,看那些东西访问了这个地址(这个时候就需要调试了)

![image-20230521083916351](./assets/image-20230521083916351.png)

这个时候可以看到,ecx+fc就是访问了这个血量,如果继续找ecx哪里来的,其实就能发现了,这个ecx就是`Player`对象;除此之外,还可以使用扫描指针的方式

![image-20230521084038011](./assets/image-20230521084038011.png)

如下图搜索即可，因为我们知道0xfc是偏移,所以加上即可;然后就直接能找到基质,Max Level就代表扫描的级别;这里选择一次是因为CSGO的结构简单,一次是基质;

- IDA查找

直接使用IDA查找相关字符串,比如血量就查

![image-20230521085017757](./assets/image-20230521085017757.png)

然后点击交叉引用,可以看到,Health的偏移实际上是0x100,上面的fc是因为旧版本的CSGO,除此之外,我们还可以查找到其他有意思的东西

![image-20230521085052546](./assets/image-20230521085052546.png)

![image-20230521085234037](./assets/image-20230521085234037.png)

比如上图,我们可以找到人物的开火状态,在`Player+0xA380`的位置

出现这个原因的本质是因为某些基质,比如C#的元数据,反射等等,提供了一种查询自己的方式;

## 实验01

写一个代码,用于读取CSGO 当前人物血量+开火状态的代码

# chap02 CSGO-ESP篇

## CSGO的基础命令

- sv-cheats 1

开启作弊模式

- bot_stop 1

暂停机器人

- mp_roundtime_defuse 60

一局时间延长60分钟

- hutme x

伤害自己

## 查找人物链表(数组)的相关信息

根据血量查找到人物也就Player之后,以内存显示该区域,就可以找到相关的人物信息,把他当成一个结构,

然后我们需要上上下下查找相关的信息,比如Player毫无疑问存放了人物的坐标,因此我们最好查看以下周围是否是坐标类似的(以浮点数查看)

然后根据他和`生命值`和`坐标`的相对偏移就是可以找到人物的坐标相对于`Player`的偏移了

这里,找到生命值的偏移是`Player+0x100`,事实上,有一个额外的地址,是`xx+0x230`,这个并不是,虽然他看起来也很像,但是问题在他是`server.dll`在访问,并不是`client.dll`在访问,因此排除掉;

当然,前提是要找对;

找到这些之后,我们最好在进行查找team,当前,查找team的前提是不断地改变阵营,然后判断到底是哪个改变,从而确定阵营;

### 矩阵的找法

前面提到了VP矩阵的概念,事实上,由于其性质,我们可以找到矩阵用很多方法;

- pitch找法

不停的改俯仰角度,然后搜索-1-1的这个角度来定位矩阵;当然,游戏的矩阵有很多个,可能会找到很多矩阵,最终验证来找到一个完美的正确的矩阵;矩阵一般都是3*4或者是4*4的;

如果ESP的时候,发现绘制有问题,基本上就是矩阵找的有问题; 事实上,ESP时候,找到的坐标一般是在Player附近,但是矩阵则不然,矩阵是给我们显示到2D屏幕上的,游戏有一个叫做`camera`的对象,也就是摄像机,正是这个东西决定了矩阵的值;

**事实上,这个找矩阵不太准**

- 变动找法

这个找法比较暴力,思路是晃动鼠标+开关镜搜索变动,来达到初筛矩阵的目的;

值得一提是CS:GO的矩阵是基质..因此是绿色的,除此之外,矩阵的特征是

- 第一行第三列或者第三列第一行是0
- 如果是4*4 矩阵 那么第三行第四行相同

```C++
0.38 -0.64 0.00 -1067.45 
0.37 0.22 1.26 676.65 
0.81 0.48 -0.33 1121.23 
0.81 0.48 -0.33 1127.11
```

如图,上面就是CSGO的矩阵;最终,找到人物数据如下:

```C++
LocalPlayer: "client.dll"+00DE997C
人的血量:0x100
坐标0x138
阵营 0xF4 2匪徒 3警
人的数组 4DFEF0C 一共10个 第一个是Player
投影矩阵:client.dll+4DEFD54
```

这样,就可以做出来最基础的透视了;
