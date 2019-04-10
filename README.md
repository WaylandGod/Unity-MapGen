# Unity MapGen

 基于多边形的随机地图生成。

![](./images/mapGen_01.png)

### Part1 Polygon

随机地图的生成首先需要一张高度图(Elevation)，可以理解为海拔数据代表了每个坐标的海拔值。再用一张降雨图(Moisture)来影响植被和河流。高度图一般的做法是使用噪音来得到，而降雨图是根据高度图计算得来的(海岸和季风的影响)。简单的户外环境使用噪音即可以得到比较自然的地形，但实际游戏需求往往是复杂的。可能需要选择在不同的位置生成不同的植物，动物群落。再或者游戏的资源的投放是需要关联不同地貌的，例如在沙漠地带投放水资源相关掉落。在平原地区生成城市，在热带雨林生成部落等。

我们使用多边形来填充地图，相比传统的基于Tile的地图生成在很大程度上降低了空间上的复杂度。100000个Tile的地图可能只需要1000个多边形就可以完成地图数据的填充。

#### Voronoi Diagram (沃罗诺伊图)

首先我们使用**Voronoi Diagram**来分割地图。沃罗诺伊图(Voronoi Diagram)又叫泰森多边形，该算法实现使用一组特定的点将平面分割成不同区域，而每一区域又仅包含唯一的特定点，并且该区域内任意位置到该特定点的距离最小。分割后得到的Voronoi节点可有多种适用场景，如在地图的平原地区生成一个面积最大的城镇等。为了使用Voronoi来分割地图，要先生成一些随机点，随机点的生成可以是柏林噪音，或其他任何随机，为了能够利用种子(seed)来还原地图，这里最好使用伪随机来生成点。

![](./images/mapGen_02.png)

随机生成了1000个点，然后我们使用**Delaunay三角剖分**来得到三角形。这里使用基于**Fortune's algorithm**的三角剖分。

![](./images/mapGen_05.png)

为了方便观察，我把随机的点数调整为500个。接下来用已有的三角形得到Voronoi多边形。

![](./images/mapGen_03.png)

绿色的多边形，即为Voronoi多边形。我们仔细观察一下单个多边形：

![](./images/mapGen_07.png)

可以看到一个三角形对应一个多边形的顶角，每个多边形又对应三角形的一个顶角。这种双重性将会用在不同的地方，比如三角形可以用来寻路，而多边形则可以用来渲染。后面会有具体的应用场景。现在我们把暂时用不到的三角形的渲染关掉，再来观察这些多边形：

![](./images/mapGen_04.png)

 可以看到多边形的大小都非常不规则，接下来我们使用**Lloyd relaxation**调整网格大小使整体更规整:

![](./images/mapGen_06.png)



> Voronoi Diagram: <https://en.wikipedia.org/wiki/Voronoi_diagram>
>
> Fortune's algorithm：<https://en.wikipedia.org/wiki/Fortune's_algorithm>
>
> Lloyd relaxation: <https://en.wikipedia.org/wiki/Lloyd%27s_algorithm>

#### 区分水和陆地

现在要生成一个被大海包围的岛屿，这里我们使用**Radial**正弦波算法来生成圆形岛屿。当然岛屿可以是任何条件约束下生成的形状，代码里提供了另外两种：Perlin和Square算法来生成岛屿。

![](./images/mapGen_08.png)

#### 区分海洋,海岸线和内陆湖

![](./images/mapGen_09.png)

1. 首先规定地图边缘均为海洋 。
2. 所有与海洋接触的水均为海洋。
3. 所有与海洋接触的陆地为海岸线即渲染为沙滩。
4. 其余的水均为湖泊。 

### Part2 海拔，河流，湿度，生物类群

#### Elevation
接下来给每个网格生成海拔值。海拔值会是很重要的数据，它将会用来影响生物类群的分布，河流的流向等。首先通过Voronoi多边形的角距离海岸线的距离来确定其海拔值，最后将Voronoi多边形所有角海拔的平均值存储到Voronoi多边形对应的**Center**中。
1. `AssignCornerElevations`方法为每个角计算海拔值
2. `AssignPolygonElevations`方法计算海拔平均值

