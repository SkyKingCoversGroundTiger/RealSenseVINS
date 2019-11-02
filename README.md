# RealSenseVINS
RealsenVINS running
准备工作
1、Intel Realsense D435i、Ubuntu 18.04
2、已经安装好Realsense驱动，如果没有的话可以参考Intel官方网站教程
3、已经安装好VINS-Mono并且在数据集上正常工作
# realsense 安装和ROS
首先安装SDK
要用最新的版本，不然会有版本冲突。我的版本是２.29
https://github.com/IntelRealSense/realsense-ros

测试运行
如果直接运行可以测试
roslaunch vins_estimator euroc.launch 
roslaunch vins_estimator vins_rviz.launch
rosbag play MH_01_easy.bag 
如果能得到结果说明配置是合理的

# 让realsense在VINS架构上运行有两种办法，分别描述一下
１　使用RTAB_MAP来直接运行
这个运行方法不需要安装rtabmap，在安装完毕realsense之后的发布包里面就含有了
1.1 Launch rtabmap:
roslaunch rtabmap_ros rtabmap.launch rtabmap_args:="--delete_db_on_start" depth_topic:=/camera/aligned_depth_to_color/image_raw rgb_topic:=/camera/color/image_raw camera_info_topic:=/camera/color/camera_info approx_sync:=false odom_topic:=/vins_estimator/odometry
1.2 Launch vins:
roslaunch vins_estimator realsense_color.launch
1.3 Launch realsense d43５i:
roslaunch realsense2_camera rs_camera.launch
1.4 In addition, drifting of imu can be seen in rqt_plot by:
rqt_plot /camera/imu/linear_acceleration/x:y:z /camera/imu/angular_velocity/x:y:z


２　使用VINSMONO提供的launch文件来运行，但是不能直接运行，需要对文件进行修改
2.1 VINSMONO里面提供了realsense的配置文件，但运行以后发现并没有工作（因为不存在imu_topic: "/camera/imu/data_raw"）
另外，主要问题在于realsense d435i在ROS中发布的imu topic是分开来的，同时这两个的时间戳也不太一样：
/camera/gyro/sample	发布角速度
/camera/accel/sample	发布线加速度

2.2 修改realsense包里的rs_camera.launch文件
第一处，修改unite_imu_method如下，这里是让IMU的角速度和加速度作为一个topic输出
 <arg name="unite_imu_method"      default="copy"/>
第二处，修改enable_sync参数为true，这里是开机相机和IMU的同步
  <arg name="enable_sync"           default="true"/>
修改VINS-Mono包里的realsense_color_config.yaml文件
第一处，修改订阅的topic
imu_topic: "/camera/imu"
image_topic: "/camera/color/image_raw"
第二处，修改相机内参，这里先再次打开运行realsesne包，然后可以通过如下命令获取相机内参
rostopic echo /camera/color/camera_info
根据所得到的内参进行修改。但事实上这个内参准不准我也不清楚
第三处，IMU到相机的变换矩阵，这里我根据注释的提示修改成2
# Extrinsic parameter between IMU and Camera.
estimate_extrinsic: 2   # 0  Have an accurate extrinsic parameters. We will trust the following imu^R_cam, imu^T_cam, don't change it.
                        # 1  Have an initial guess about extrinsic parameters. We will optimize around your initial guess.
                        # 2  Don't know anything about extrinsic parameters. You don't need to give R,T. We will try to calibrate it. Do some rotation movement at beginning.                        
#If you choose 0 or 1, you should write down the following matrix.
第四处，IMU参数，这里我全部修改注释给的参数
#imu parameters       The more accurate parameters you provide, the better performance
acc_n: 0.2          # accelerometer measurement noise standard deviation. #0.2
gyr_n: 0.05         # gyroscope measurement noise standard deviation.     #0.05
acc_w: 0.02         # accelerometer bias random work noise standard deviation.  #0.02
gyr_w: 4.0e-5       # gyroscope bias random work noise standard deviation.     #4.0e-5
g_norm: 9.80       # gravity magnitude
第五处，是否需要在线估计同步时差，根据下面的建议这里选择不需要
https://blog.csdn.net/weixin_44580210/article/details/89789416
#unsynchronization parameters
estimate_td: 0                      # online estimate time offset between camera and imu
td: 0.000                           # initial value of time offset. unit: s. readed image clock + td = real image clock (IMU clock)
第六处，相机曝光改成全局曝光
#rolling shutter parameters
rolling_shutter: 0                      # 0: global shutter camera, 1: rolling shutter camera
rolling_shutter_tr: 0               # unit: s. rolling shutter read out time per frame (from data sheet). 

3. 打开摄像头，运行VINS-Mono
roslaunch realsense2_camera rs_camera.launch 
roslaunch vins_estimator realsense_color.launch 
roslaunch vins_estimator vins_rviz.launch
应该可以得到结果了。虽然IMU飘逸还是很明显
一直显示[ WARN] [1572665263.902421381]: imu message in disorder!
另外，很多人反映IMU经常跑飞了
https://blog.csdn.net/qq_41839222/article/details/86552367



