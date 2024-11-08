============================
Cartographer导航算法部署
============================

1. Cartographer导航算法介绍
============================

Cartographer项目地址： `Cartographer_github <https://github.com/JYC924693/cartographer_navigation/tree/main>`_

Cartographer是Google推出的一套基于图优化的激光SLAM算法，它同时支持2D和3D激光SLAM，可以跨平台使用，支持Lidar、IMU、Odemetry、GPS、Landmark等多种传感器配置。是目前落地应用最广泛的激光SLAM算法之一。

Cartographer代码最重要的贡献不仅仅是算法，而是工程实现实在是太优秀了！它不依赖PCL，g2o, iSAM, sophus, OpenCV等第三方库，所有轮子都是自己造的，2D/3D的SLAM的核心部分仅仅依赖于Boost、Eigen（线性代数库）、Ceres（Google开源的非线性优化库）等几个底层的库。

Cartographer采用基于google自家开发的ceres非线性优化的方法，基于submap子图构建全局地图的思想，能有效的避免建图过程中环境中移动物体的干扰。并能天然的输出协方差矩阵，后端优化的输入项。

2. Cartographer导航算法部署
============================

Cartographer的部署大致分为两步，第一步：完成建图部分；第二步：完成导航部分。

之前不清楚Cartographer的实际功能，它可以 **实现地图构建**、 **定位** 等功能，但是设置完目标点后它是不能控制机器人移动的，机器人移动还是需要依靠move_base来实现，在进行第二步时，机器人可以移动，但是移动过程不平滑，速度忽快忽慢，考虑和局部路径规划的配置有关系，检查move_base的局部路径规划配置文件，首先需要了解其参数含义。

::

    move_base/config
        ├── amcl.launch
        ├── costmap_common_params.yaml
        ├── dwa_local_planner_params.yaml
        ├── global_costmap_params.yaml
        ├── global_planner_params.yaml
        ├── local_costmap_params.yaml
        ├── move_base_params.yaml
        └── navfn_global_planner_params.yaml


move_base配置文件参数详解
----------------------------


**move_base_params.yaml**

.. code-block:: yaml
    :linenos:

    shutdown_costmaps: false # 表示是否在导航停止时关闭代价地图（costmap）。设置为 False 表示不关闭。

    controller_frequency: 5.0 # 控制器的更新频率（以Hz为单位）。
    controller_patience: 3.0 # 控制器在没有找到路径时的耐心时间（以秒为单位）。


    planner_frequency: 1.0 # 全局规划器的更新频率（以Hz为单位）。
    planner_patience: 5.0 # 全局规划器在没有成功找到路径时的最大等待时间（以秒为单位）。

    oscillation_timeout: 10.0 # 振荡超时时间（以秒为单位）。
    oscillation_distance: 0.2 # 机器人被认为是振荡的最小距离（以米为单位）。

    base_local_planner: "dwa_local_planner/DWAPlannerROS" # 使用的局部规划器类型，这里是 DWAPlannerROS，一种常用的动态窗口规划器。
    base_global_planner: "navfn/NavfnROS" # 使用的全局规划器类型，这里是 NavfnROS，基于A*算法的路径规划器。

    recovery_behavior_enabled: true # 表示是否启用恢复行为。当导航遇到问题时，机器人会执行一系列恢复动作，设置为 True 表示启用。


在调试机器人时，测试添加原始github工程的amcl以及yikun的amcl文件，如果不加载amcl，在RVIZ中map和RobotModel到处闪烁，可能原因是amcl中也有相关的坐标变换，导致其与原来的发生冲突，如何实现cartographer与amcl的双重定位？


3. Cartographer+amcl双重定位
============================

amcl.launch配置文件参数详解
-----------------------------


.. code-block:: xml
    :linenos:

    <?xml version="1.0"?>
    <launch>
    <node pkg="amcl" type="amcl" name="amcl" output="screen">
    

    
        <!-- 里程计模型参数 -->
        <!-- 如果模型为corrected，后面的几个参数值都要减小，如下 -->
        <!--
        <param name="odom_model_type" value="diff_corrected">
        <param name="odom_alpha1" value="0.005"/>
        <param name="odom_alpha2" value="0.005"/>
        <param name="odom_alpha3" value="0.010"/>
        <param name="odom_alpha4" value="0.005"/>
        <param name="odom_alpha5" value="0.003"/>
        -->
        <param name="odom_model_type" value="omni"/>  <!-- 里程计模式为差分 -->
        <param name="odom_alpha1" value="0.2"/>  <!-- 由机器人运动部分的旋转分量估计的里程计旋转的期望噪声，默认0.2 -->
        <param name="odom_alpha2" value="0.2"/>  <!-- 由机器人运动部分的平移分量估计的里程计旋转的期望噪声，默认0.2 -->
        <param name="odom_alpha3" value="0.8"/>  <!-- 由机器人运动部分的平移分量估计的里程计平移的期望噪声，默认0.2 -->
        <param name="odom_alpha4" value="0.2"/>  <!-- 由机器人运动部分的旋转分量估计的里程计平移的期望噪声，默认0.2 -->
        <param name="odom_alpha5" value="0.2"/>  <!-- 平移相关的噪声参数（仅用于"omni"模型），默认0.2 -->
        

        
        <!-- 激光模型参数 -->
        <param name="laser_min_range" value="-1.0"/>  <!-- 被考虑的最小扫描范围；参数设置为-1.0时，将会使用激光上报的最小扫描范围 -->
        <param name="laser_max_range" value="-1.0"/>  <!-- 被考虑的最大扫描范围；参数设置为-1.0时，将会使用激光上报的最大扫描范围 -->
        <param name="laser_max_beams" value="30"/>  <!-- default:30,更新滤波器时，每次扫描中多少个等间距的光束被使用 -->
        <!-- 这4个laser_z参数，在动态环境下的定位时用于异常值去除技术 -->
        <param name="laser_z_hit" value="0.5"/>  <!-- 模型的z_hit部分的混合权值，默认0.95 -->
        <param name="laser_z_short" value="0.05"/>  <!-- 模型的z_short部分的混合权值，默认0.1 -->
        <param name="laser_z_max" value="0.05"/>  <!-- 模型的z_max部分的混合权值，默认0.05 -->
        <param name="laser_z_rand" value="0.5"/>  <!-- 模型的z_rand部分的混合权值，默认0.05 -->
        <param name="laser_sigma_hit" value="0.2"/>  <!-- 被用在模型的z_hit部分的高斯模型的标准差，默认0.2m -->
        <param name="laser_lambda_short" value="0.1"/>  <!-- 模型z_short部分的指数衰减参数，默认0.1，λ越大随距离增大意外对象概率衰减越快,默认:0.1 -->
        <param name="laser_model_type" value="likelihood_field"/>  <!-- 模型使用，默认:likelihood_field -->
        <param name="laser_likelihood_max_dist" value="2.0"/>  <!-- 地图上做障碍物膨胀的最大距离，用作likelihood_field模型,默认:2.0 -->
        
        

        <!-- 滤波器参数 -->
        <param name="min_particles" value="2000"/>  <!-- default:100，允许的粒子数量的最小值 -->
        <param name="max_particles" value="5000"/>  <!-- default:5000，允许的粒子数量的最大值 -->
        <param name="kld_err" value="0.05"/>  <!-- default:0.01，真实概率分布与估计概率分布间的误差 -->
        <param name="kld_z" value="0.99"/>  <!-- default:0.99，标准正态分位数（1 - P），其中P是在估计分布的误差，要小于kld_err -->
        <param name="update_min_d" value="0.1"/>  <!-- default:0.2，向前运动多少就更新粒子阈值，建议不大于车半径 -->
        <param name="update_min_a" value="0.2"/>  <!-- default:PI/6，同样的，旋转多少弧度就更新粒子阈值 -->
        <param name="resample_interval" value="1"/>  <!-- default:2，对粒子样本的重采样间隔，设置2~5即可 -->
        <param name="transform_tolerance" value="0.1"/>  <!-- defaule:0.1,tf变换发布推迟的时间 -->
        <param name="recovery_alpha_slow" value="0.001"/>  <!-- 两个失效恢复参数，默认二者都是0 -->
        <param name="recovery_alpha_fast" value="0.1"/>
        <param name="gui_publish_rate" value="10"/>  <!-- default:-1,scan和path发布到可视化软件的最大频率，-1的话就是不发布 -->
        <param name="save_pose_rate" value="0.5"/>  <!-- default:0.5 -->
        <!-- 下面两个参数是navigation 1.4.2以后新加入的参数 -->
        <param name="use_map_topic" value="false"/>  <!-- 为true时，AMCL将会订阅map话题，而不是调用服务返回地图  true-->
        <param name="first_map_only" value="false"/>  <!-- 为true时，AMCL将仅使用订阅的第一个地图，而并非每次都更新接收的新地图   true -->
        
        

        <!-- 坐标系参数 -->
        <!-- 这里设置三个坐标系的名称，默认分别三odom，base_link，map，改成你自己的 -->
        <param name="odom_frame_id" value="odom"/>  <!-- 里程计坐标系 -->
        <param name="base_frame_id" value="base_link"/>  <!-- 添加机器人基坐标系 base_footprint-->
        <param name="global_frame_id" value="map"/>  <!-- 添加地图坐标系 -->
        <param name="tf_broadcast" value="true"/>  <!-- default:true,设置为false可阻止amcl发布全局坐标系和里程计坐标系之间的tf变换 -->
        

        
        <!-- 机器人初始化数据设置 -->
        <!-- 下面几个initial_pose_参数决定撒出去的初始位姿粒子集范围中心 -->
        <param name="initial_pose_x" value="11.9"/>  <!-- 初始位姿均值（x）-->
        <param name="initial_pose_y" value="17.7"/>  <!-- 初始位姿均值（y） -->
        <param name="initial_pose_a" value="0.0"/>  <!-- 初始位姿均值（yaw） -->
        <!-- initial_cov_参数决定初始粒子集的范围 -->
        <param name="initial_cov_xx" value="1"/>  <!-- 初始位姿协方差（x*x） -->
        <param name="initial_cov_yy" value="1"/>  <!-- 初始位姿协方差（y*y） -->
        <param name="initial_cov_aa" value="(π/6)*(π/6)"/>  <!-- 初始位姿协方差（yaw*yaw） -->
        <remap from="scan" to="scan"/>  <!--  /rslidar/scan /r2000_driver_node/scan -->
        

    </node>


    </launch>

目前的定位还是依赖于AMCL，cartographer关于定位的部分网上大部分搜出来的是纯定位，所以关于这部分继续用Cartographer+AMCL+move_base实现机器人的移动。



4. Move_base机器人移动存在问题与解决方法
=========================================

1. 机器人移动过程中抖动
-------------------------

.. code-block:: shell
    :linenos:

    [ERROR] [1727264276.043005216]: Global Frame: odom Plan Frame size 44: map

    [ WARN] [1727264276.043034837]: Could not transform the global plan to the frame of the controller
    [ERROR] [1727264276.043056281]: Could not get local plan
    [ INFO] [1727264276.242865595]: Got new plan
    [ INFO] [1727264277.242824666]: Got new plan
    [ERROR] [1727264277.242928682]: Extrapolation Error: Lookup would require extrapolation 0.000251906s into the future.  Requested time 1727264277.240116358 but the latest data is at time 1727264277.239864588, when looking up transform from frame [odom] to frame [map]

    [ERROR] [1727264277.242954904]: Global Frame: odom Plan Frame size 26: map

    [ WARN] [1727264277.242979373]: Could not transform the global plan to the frame of the controller
    [ERROR] [1727264277.242996166]: Could not get local plan
    [ INFO] [1727264277.442827636]: Got new plan
    [ERROR] [1727264277.442935681]: Extrapolation Error: Lookup would require extrapolation 0.002063299s into the future.  Requested time 1727264277.431960106 but the latest data is at time 1727264277.429896832, when looking up transform from frame [odom] to frame [map]

    [ERROR] [1727264277.442960075]: Global Frame: odom Plan Frame size 21: map

    [ WARN] [1727264277.442981119]: Could not transform the global plan to the frame of the controller
    [ERROR] [1727264277.442999029]: Could not get local plan
    [ INFO] [1727264277.642870484]: Got new plan
    [ INFO] [1727264278.642865193]: Got new plan
    [ INFO] [1727264279.242904882]: Goal reached


**解决方法：**

.. image::  ../image/Q1A1.png

.. image::  ../image/Q1A2.png

.. image::  ../image/Q1A3.png

2. 机器人移动到目标点附近抖动
------------------------------

.. image::  ../image/Q2.png

**解决方法：**

TODO

3. 机器人经过门口时旋转，重新规划路径
-------------------------------------
**解决方法：**

TODO