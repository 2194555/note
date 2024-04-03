# 调车流程

## 1.连车

### wifi

首先要知道车的IP地址：浏览器输入192.168.1.1，账号及密码均为admin，在已链接设备中找到对应的车，记住其IP地址

连接队内WIFI，然后在终端中输入 `ssh dynamicx@*IP*` (密码：dynamicx)

ssh免密：`ssh-copy-id dynamicx@*IP*`之后连接本车不需输入密码

### 有线

在终端中输入 `ssh -o StrictHostKeyChecking=no -l "dynamicx" "IP"`

或直接输入 `wired`

## 2.关自启

``` bash
stop_master     （sudo systemctl stop start_master.service）
stop_ecat_start （sudo systemctl stop rm_ecat_start.service）
```

## 3.加载控制器

1.先开`roscore`

2.然后`bringup`(全开)

或

```bash
mon launch rm_config rm_ecat_hw.launch（或rm_can_hw.launch）（硬件）
mon launch rm_config load_controllers.launch （加载控制器）
```

## 4.调试工具

一般来说，调试工具应在本地终端打开

* rqt

```bash
rqt
```

* plotjugger

```bash
rosrun plotjuggler plotjuggler
```

* rviz

```bash
rviz
```

---

---

# 测发射

开/controllers/shooter_controller

rqt在/controllers/shooter_controller/command/发命令
或

```bash
rostopic pub /controllers/shooter_controller/command （Tab补全）
```

---

---

# 测电机
灵活运用，根据需求在已有的配置文件中修改，本质为用待测电机替换原joint所用电机

## 首先创建测试配置

### 1.本地已有可用控制器

修改电机参数，如在终端输入`candump can0`可获取can0上信息，找到对应电机及其参数

接着转到对应硬件下的配置文件（can或ecat分别对应在rm_config内的rm_hw和rm_ecat_hw），找到想要使用的控制器的joint所对应的电机，修改can、电机id、电机类型

### 2.新建测试用控制器

若本地无合适控制器，则需新建测试用控制器

转到rm_config下rm_controllers

如下例，则可写入一个包含trigger_joint（比如其中的standard4.yaml）的配置文件中

```yaml
  test_controller:
    type: effort_joint_controller/JointPositionController
    joint: trigger_joint
    pid: { p: 3.0, i: 0.0, d: 0.15, i_clamp_max: 0.0, i_clamp_min: 0.0, antiwindup: true, publish_state: true }
```
则可借用trigger_joint生成一个名为test_controller，类型为effort_joint_controller/JointPositionController的控制器

然后获取并修改**相对应**的trigger_joint的电机参数（同1）

最后在load_controllers.launch中加入该控制器

tips：

* 编写yaml文件时注意对应层级的缩进
* 这里相对应指的是，如使用standard4的trigger_joint新建了控制器，在修改电机参数时应去到standard4对应的hw的配置文件中修改
* pid参数复制一个就好

## 开始测试

在本地电脑上开roscore、跑硬件、加载控制器
然后用rqt打开刚刚配置好的控制器，并发布命令，再用plotjugger查看所需数据

---

---

# imu零飘补偿

1.将rm_config/rm_hw/${robot_type}.yaml中的imu：gimbal_imu的angular_vel_offset全部设成0并部署至车上

2.启动摩擦轮等有可能影响imu的组件，然后可将其他控制器全关，只留joint_state和shooter_controller（控摩擦轮）

3.plotjugger拉出gimbal_imu/angular_x/y/z三个图像，然后右键点击图像，选中apply_filter_to_data，选中moving_average，将simple_count拉满，缓存区先给5，几秒后给200

4.轻压车头后，将车保持静止一段时间，等到图像较为规律且小幅波动时，暂停记录三个图像的大概的值，一般取画面内（最高值+最低值）/2

5.将4中得到的值取相反数，填到rm_config/rm_hw/${robot_type}.yaml中的imu：gimbal_imu的angular_vel_offset的x、y、z中

---

---

# 工程相关

## 编写动作组

如下即为一个完整动作的一部分，包含数个步骤（step）

```yaml
  TWO_STONE_SMALL_ISLAND:           #整个动作的名字
    - step: "GIMBAL_READY"          #每一步的名字
      gimbal:                       #该步所操控的组件（同下的arm、gripper等），注意每一个step中只能使用一个组件
        <<: *SIDE_POS               #之后便是该组件的具体参数，如位置、动作速度、误差范围等
    - step: "ORE_ROTATOR_READY"
      ore_rotator:
        target: *READY_POS
    - step: "ORE_LIFTER_READY"
      ore_lifter:
        target: *BIN_MID_POS
    - step: "SMALL_ARM_READY"
      arm:
        joints: [ *JOINT1_SMALL_READY, *JOINT2_BACK_POSITION, *JOINT3_SMALL_READY, *JOINT4_R_POSITION, *JOINT5_MID_POSITION, *JOINT6_MID_POSITION, *JOINT7_MID_POSITION ]
        common:
          <<: *NORMALLY
        tolerance:
          <<: *NORMAL_TOLERANCE
    - step: "OPEN_GRIPPER"
      gripper:
        <<: *OPEN_GRIPPER
```

其中，形如`<<: *SIDE_POS`表示将`SIDE_POS`中的所有参数复制到当前位置，`*SIDE_POS`表示引用`SIDE_POS`中的所有参数，类似于宏定义或是bashrc里的alias

例如以下是`SIDE_POS`和`JOINT1_SMALL_READY`的定义

```yaml
  gimbal:
    side_pos: &SIDE_POS
      frame: gimbal_lifter
      position: [ 0.001 , -3, 0.0 ]

  joint1:
    mechanical:
      #...
      small_ready: &JOINT1_SMALL_READY
                     0.15
```
