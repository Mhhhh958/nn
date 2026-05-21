# carla\_auto\_vision\_navigator 项目

各位老师、同学，大家好！今天我带来的项目是 `carla\_auto\_vision\_navigator`——基于多模态融合的Carla非结构化场景异常检测自动驾驶项目。

本项目基于原有`carla\_auto\_vision\_navigator`框架，补充了多模态感知、非结构化场景适配、异常检测及智能决策功能，实现了自动驾驶车辆在复杂非结构化场景下的安全行驶，填补了原有项目在“多模态\+异常检测”领域的空白，兼具实用性与创新性。

接下来，我将从项目背景、核心架构、功能实现、运行演示及总结展望五个部分，为大家详细介绍这个项目。

# 一、项目背景与意义

## 1\.1 现有痛点

当前自动驾驶仿真项目多聚焦于**结构化城市道路**，仅支持单模态感知（多为图像）和常规目标检测，无法应对非结构化场景（如乡村道路、施工路段）中的突发异常，存在明显的功能局限：

- 场景局限：仅覆盖城市结构化道路，无乡村、城郊等非结构化场景适配；

- 感知单一：依赖单模态图像数据，抗干扰能力弱（如雨天、大雾天易失效）；

- 无异常检测：仅能识别常规车辆、行人，无法检测道路坑洼、落石、施工围挡等突发异常。

## 1\.2 项目意义

本项目针对上述痛点，拓展了非结构化场景适配能力和多模态感知能力，新增异常检测与智能决策模块，实现三大核心价值：

1. 填补技术空白：完善原有项目功能，实现“多模态融合\+非结构化场景\+异常检测”的全流程适配；

2. 贴近实际需求：模拟真实复杂路况，为自动驾驶算法落地提供更全面的仿真测试环境；

3. 复用现有资源：基于原有`carla\_auto\_vision\_navigator`框架开发，降低开发成本，便于后续拓展。

# 二、项目核心架构

项目整体采用模块化设计，基于Carla仿真平台，融合深度学习、多模态感知、PID控制等技术，核心分为5大模块，各模块独立运行、协同工作，架构清晰且易于维护。

核心架构图（演讲时可配合PPT展示）：

Carla客户端 → 传感器管理 → 多模态异常检测 → 智能决策 → PID控制 → 车辆执行

## 2\.1 模块分工说明

|模块名称|核心功能|核心作用|
|---|---|---|
|Carla客户端（carla\_client\.py）|连接Carla服务器、加载非结构化场景、生成车辆及异常Actor|搭建仿真环境，提供基础运行载体|
|传感器管理（sensor\_manager\.py）|初始化RGB相机、LiDAR、雷达、IMU，采集并预处理多模态数据|为异常检测提供高质量数据输入|
|多模态异常检测（object\_detector\.py）|融合图像\+点云数据，检测场景中的异常目标及类型|项目核心，实现异常识别能力|
|智能决策（decision\_maker\.py）|根据异常检测结果，生成减速、绕行、紧急停车等决策|保障车辆行驶安全，应对复杂异常场景|
|PID控制（pid\_controller\.py）|将决策转化为车辆控制指令，实现速度、转向精准控制|执行决策，确保车辆按指令行驶|

# 三、核心功能实现（重点）

接下来，我将重点介绍项目的三大核心功能实现，这也是本项目的创新点和核心价值所在。

## 3\.1 非结构化场景适配

针对原有项目仅支持结构化道路的问题，我们新增了非结构化场景加载和异常Actor生成功能：

- 场景支持：适配Carla的Town07（乡村道路）、Town10（城郊道路），可设置雨天、大雾等复杂天气；
- 异常Actor：可自动生成落石、道路坑洼、施工围挡、违规行人等6类异常目标，模拟真实复杂路况；
- 核心代码：通过`load\_unstructured\_scene\(\)`加载场景，`spawn\_anomaly\_actors\(\)`生成异常目标，可灵活拓展场景和异常类型。核心代码片段如下：
              `\# 加载非结构化场景核心代码
          def load\_unstructured\_scene\(self, town\_name=\&\#34;town07\&\#34;\):
    if town\_name not in UNSTRUCTURED\_SCENES:
        logger\.error\(f\&\#34;不支持的非结构化场景：\{town\_name\}\&\#34;\)
        return False
    \# 加载地图并设置天气
    self\.world = self\.client\.load\_world\(town\_name\)
    self\.map = self\.world\.get\_map\(\)
    if UNSTRUCTURED\_SCENES\[town\_name\]\[\&\#34;weather\&\#34;\] == \&\#34;RAINY\&\#34;:
        self\.world\.set\_weather\(carla\.WeatherParameters\.RainyNight\)
    logger\.info\(f\&\#34;加载非结构化场景：\{town\_name\}\&\#34;\)
          ``    return True`

## 3\.2 多模态感知融合

突破原有单模态感知局限，融合RGB相机、LiDAR、雷达、IMU四种传感器数据，提升感知稳定性：

- 传感器配置：RGB相机（采集图像）、LiDAR（采集点云，感知距离）、雷达（检测动态目标速度）、IMU（感知车辆姿态）；
- 数据预处理：对图像进行尺寸调整、归一化，对点云进行格式转换，确保数据统一性；
- 融合方式：支持加权融合、拼接融合、注意力融合三种方式，可根据场景灵活切换，提升异常检测准确率。核心代码片段如下：
              `\# 多模态融合核心代码（加权融合示例）
          def forward\(self, rgb\_data, lidar\_data\):
    \# 单模态特征提取
    rgb\_feat = self\.rgb\_model\(self\.preprocess\_rgb\(rgb\_data\)\)
    lidar\_feat = self\.lidar\_model\(self\.preprocess\_lidar\(lidar\_data\)\)
    \# 加权融合
    fusion\_feat = self\.rgb\_weight \* rgb\_feat \+ self\.lidar\_weight \* lidar\_feat
    \# 异常分类
    logits = self\.classifier\(fusion\_feat\)
    probs = torch\.softmax\(logits, dim=1\)
          ``    return probs`

## 3\.3 异常检测与智能决策

这是项目的核心创新模块，实现“检测\-决策\-控制”的闭环：

1. 异常检测：基于YOLOv7和自定义LiDAR特征提取模型，融合多模态数据，检测6类异常，置信度阈值可调节（默认0\.7）；
2. 智能决策：根据异常类型和置信度，划分3级风险，生成对应决策（低风险：减速；中风险：绕行；高风险：紧急停车）；
3. PID控制：将决策转化为车辆控制指令，精准控制车速和转向，确保车辆平稳执行决策，避免急刹、侧滑等问题。核心代码片段如下：
              `\# PID控制核心代码
          def control\_vehicle\(self, vehicle, action\):
    \# 获取当前车速（km/h）
    current\_speed = 3\.6 \* np\.linalg\.norm\(\[vehicle\.get\_velocity\(\)\.x, 
                                          vehicle\.get\_velocity\(\)\.y\]\)
    \# PID计算输出
    speed\_output = self\.compute\(action\[\&\#34;speed\&\#34;\], current\_speed\)
    \# 构造控制指令
    control = carla\.VehicleControl\(\)
    control\.throttle = max\(min\(speed\_output / 100, 1\.0\), 0\.0\) if speed\_output \&gt; 0 else 0\.0
    control\.brake = max\(min\(\-speed\_output / 100, 1\.0\), 0\.0\) if speed\_output \&lt; 0 else action\[\&\#34;brake\&\#34;\]
    control\.steer = action\[\&\#34;steer\&\#34;\]
          ``    vehicle\.apply\_control\(control\)`

## 3\.4 依赖与环境配置

项目依赖简洁，可快速部署，核心依赖包括：

`carla==0\.9\.15、torch==2\.0\.1、opencv\-python、open3d` 等，已整理为`requirements\.txt`，执行`pip install \-r requirements\.txt`即可完成安装。

# 四、运行演示说明

项目可直接运行，步骤简单，演讲时可现场演示（提前准备Carla服务器）：

效果图

![camera_radar_fusion_demo](C:\Users\Administrator\Desktop\camera_radar_fusion_demo.png)

![c3caafd6a1f9734c4d7899a0dd8a3ab6](C:\Users\Administrator\Desktop\c3caafd6a1f9734c4d7899a0dd8a3ab6.gif)

## 4\.1 运行前提

- 启动Carla 0\.9\.15服务器，命令：`CarlaUE4\.exe /Game/Carla/Maps/Town07 \-carla\-port=2000`；

- 安装项目所有依赖，确保Python版本为3\.7\-3\.12；

- 无需额外配置，直接运行根目录`run\.py`即可启动项目。

## 4\.2 演示流程

1. 运行`run\.py`，自动连接Carla服务器，加载Town07乡村场景；

2. 自动生成红色特斯拉车辆和各类异常目标（落石、坑洼等）；

3. 车辆启动，传感器实时采集数据，异常检测器实时识别；

4. 当检测到异常时，自动生成决策（如遇到落石时绕行，遇到违规行人时紧急停车）；

5. 演示过程中，可通过控制台查看异常检测结果、决策指令和车辆状态。

## 4\.3 关键说明

演示时可重点展示：多模态数据可视化（RGB图像、LiDAR点云）、异常检测准确率、决策的合理性和车辆控制的平稳性，直观体现项目的核心功能。补充演示效果图说明：
项目完整运行效果图，左侧为Carla仿真场景（车辆行驶、异常目标），右侧为控制台输出（异常检测结果、决策指令、车速）：不同异常场景的决策执行对比图，清晰展示“减速、绕行、紧急停车”三种决策的实际效果，体现项目的实用性。

# 五、项目总结与展望

## 5\.1 项目总结

本项目基于`carla\_auto\_vision\_navigator`原有框架，完成了全功能补全，实现了三大突破：

- 场景突破：从结构化道路拓展到非结构化场景，适配更复杂的真实路况；

- 感知突破：从单模态升级为多模态融合，提升感知稳定性和抗干扰能力；

- 功能突破：新增异常检测与智能决策，实现“感知\-决策\-控制”闭环，提升自动驾驶安全性。

项目所有代码可直接运行，结构规范、易于拓展，可作为自动驾驶仿真测试的重要工具。

## 5\.2 未来展望

后续我们将从三个方面优化项目，提升其实用性和扩展性：

1. 拓展场景与异常类型：新增更多非结构化场景（如山区道路）和异常目标（如动物闯入、道路积水）；

2. 优化模型性能：替换更精准的预训练异常检测模型，提升检测速度和准确率，适配实时场景；

3. 对接ROS系统：实现与真实车辆的对接，将仿真算法落地到实际自动驾驶场景中。

# 六、致谢

感谢各位老师、同学的聆听！以上就是`carla\_auto\_vision\_navigator`项目的全部介绍，如有疑问，欢迎大家提问。
