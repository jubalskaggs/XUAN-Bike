# 调试记录
#### 2021-5-29  

> **PID参数：**
>
> * 互补滤波器参数 [0.999]  
> * ax -p 250000 -i 0 -d 30000 -s 3.8  
> * sx -p 0 -i 0

**主要调整**

1. **互补滤波器参数由 0.9->0.95->0.99**。在0.99处大幅减轻了系统震动，经分析互补滤波的作用是在**短时（实时）使用陀螺仪的积分数值作为角度，长时使得滤波后的值收敛至加速度计角度值**，而由于加速度计对震动噪声极为敏感（这是传感器的特性无法消除，在强震动下加速度计数据不滤波的话基本无法使用），所以应该适当加大陀螺仪的权重，最终效果是滤波后抖动明显减弱。

2. **飞轮固定件重新设计打印，使得飞轮相对于旋转轴尽量中心对称**。同样也是为了减轻震动。

目前已经可以初步稳定（但持续不了多久），调试过程发现用手按住控制板减轻震动后可以有效提升稳定性，因此之前**用橡胶球连接板子的方式是不合理的**，后续尝试强化控制板和部件的连接刚度减小震动。

可见不同于以往的小车调试，在大车的功率和惯量下系统震动影响被大幅度放大，传感器数据必须经过妥善滤波处理才能使系统稳定，这点在后续改进。

PID参数调试思路大概是先调P导致左右晃动，~~再单独调D（此时很不稳定会导致高频振荡），加上PD（D的值由0逐渐加，大概就在单独D的震荡点附近，再酌情增大，此时反而不会震荡）。~~

---

#### 2021-5-30

> **PID参数：**
>
> * 互补滤波器参数 [0.999]  
> * ax -p 500000 -i 0 -d 8000000 -s 2.4
> * sx -p 0.000003  -i 0.0000001
>
> **2阶ButterWorth滤波器参数：**
>
> * GYRO_LPF_CUTOFF_FREQ    150
> *  ACCEL_LPF_CUTOFF_FREQ    150

**主要调整**

1. 为了解决之前说的震动问题，**引入了`二阶巴特沃斯滤波器`**，即**先对陀螺仪和加速度计的数据进行滤波，然后再输入互补滤波器进行数据融合**。经分析由于互补滤波的输出陀螺仪权重远大于加速度计，所以震荡的原因陀螺仪应该贡献更大，在代码中将互补滤波的陀螺仪输入改为`二阶巴特沃斯滤波器`滤波之后的数据，效果非常好。

注意此时加速度计依然是使用原始数据，因为加速度计是作为收敛回归用，`fr=0.999`的情况下本身收敛就很慢，如果再滤波引入相位滞后会影响系统稳定性。

2阶ButterWorth滤波器参数的**更新率就是我们调用滤波函数的频率**，代码中`filterFreq`为200（PID线程的频率）；cutoff频率是系统的截止频率，**一般设置得比震荡频率低一点**这样可以有效滤除高频分量。

经调整后已经可以稳定直立，但是还是会有低频的左右摆动，不像小车那样稳定且安静。


---

#### 2021-5-31

> **PID参数：**
>
> * 互补滤波器参数 [0.999]  
> * ax -p 1500000 -i 0 -d 80000000 -s 1
> * sx -p 0.000002  -i 0.00000003 
>
> **2阶ButterWorth滤波器参数：**
>
> * GYRO_LPF_CUTOFF_FREQ    100
> *  ACCEL_LPF_CUTOFF_FREQ    50
>
> **Ctrl-FOC驱动器参数：**
>
> * vel_gain = 0.0003 
> * vel_integrator_gain = 0.006 

**主要调整**

1. 修改了FOC驱动器的参数，`vel_gain`由`0.0005->0.0003`，`vel_integrator_gain`适当增大。
2.  2阶ButterWorth滤波器参数`GYRO_LPF_CUTOFF_FREQ`由`150->100`，`ACCEL_LPF_CUTOFF_FREQ`由`150->50`。

降低FOC驱动器`vel_gain`值后驱动器震荡减弱，但响应速度指令的速度也会变慢一些，所以这个值大概的合理位置就是在设置为0速度时拨动飞轮能感觉到`弹性松弛但有阻尼`的效果。

2阶ButterWorth滤波器参数进一步减低陀螺仪截止频率，经测试稳定效果更好了，但是需要重新调节角度环PID参数，大幅加大D值，适当加大P值，使得系统更快收敛。同时给加速度计也进行了滤波，在互补滤波器中输入的两个数据都经过了滤波，效果有改善。


---

#### 2021-6-1

> **PID参数：**
>
> * 互补滤波器参数 [0.999]  
> * ax -p 1500000 -i 0 -d 80000000 -s 1
> * sx -p 0.000003  -i 0.00000003 
>
> **2阶ButterWorth滤波器参数：**
>
> * GYRO_LPF_CUTOFF_FREQ    100
> * ACCEL_LPF_CUTOFF_FREQ    50
>
> **Ctrl-FOC驱动器参数：**
>
> * vel_gain = 0.0003 
> * vel_integrator_gain = 0.006 

**主要调整**

* `sx -p`由`0.000002->0.000003`。
* 删去了MPU6050低通滤波寄存器中 `setDLPFMode(MPU6050_DLPF_BW_98)`代码，经测试这个几乎没有影响（可能因为后面软件有低通滤波了）。

**本次未作太多参数调整，但稳定效果有质的提升！**

首先是将速度环P适当增大了，系统调节角度速度加快可以抗衡更大的推力（但不可太大否则震荡）。

然后重点来了！！！

**经测试意外发现，如果给控制器插着USB线进行上电则系统可以进入到一个超稳状态，而直接用电池上电就会稳定且左右轻微震荡。**

经分析可能是上电瞬间树莓派和电机初始化带来电源不稳定，造成了控制器某些不稳定，所以**在使用电池时，先按住主控Reset按键，让系统上电几秒钟后再松开按键启动主控，此时车子就依然可以进入到超稳状态。**

> 超稳状态下如果不固定龙头的话可能会出现高频小幅震荡的情况（注意区别于以前的低频震荡），此时用手稳住龙头几秒就可完美平衡且安静了。
>
> 龙头与舵机固定后就不会有这种问题了。

**还有一个很奇怪的问题：**

代码中尝试加入舵机PWM输出代码，但一旦添加就不能进入超稳状态了，更奇怪的是，又不能删掉PWM的初始化代码，否则也进入不了超稳？？类似的Servo线程的FreeRTOS初始化代码也不能删除，否则依然无法超稳？？？ 

> 初步分析可能是两个原因：
>
> * 硬件问题，是否舵机走线带来电磁干扰影响了MPU6050的数据采集？**后续通过直接将该引脚设置为初始化低电平输出替代初始化PWM看看是否可以解决，是的话说明的确是硬件问题。**
> * 软件问题，由于某种奇妙的耦合导致PID线程的周期不稳，**产生了某种竞争？这个后续通过示波器查看PID的GPIO翻转，或者打印micro()来判断。**

---

> **PID参数：**
>
> * 互补滤波器参数 [0.999]  
> * ax -p 1000000 -i 0 -d 30000000 -s 4
> * sx -p 0.000005  -i 0.00000002 
>
> **2阶ButterWorth滤波器参数：**
>
> * GYRO_LPF_CUTOFF_FREQ    100
> * ACCEL_LPF_CUTOFF_FREQ    50
>
> **Ctrl-FOC驱动器参数：**
>
> * vel_gain = 0.0003 
> * vel_integrator_gain = 0.006 
> * current_lim = 30

**主要调整**

* PID output滑动滤波参数N已经去掉，经测试无影响。
* 修复了一个BUG，速度环PID的积分饱和修改不完全。
* 增大了Ctrl-FOC驱动器的`current_lim` ，由`20->30`，恢复里明显增大，稳定效果更好，但对应的PID参数需要重新调节，所以`ax -p 1500000 -i 0 -d 80000000`->`ax -p 1000000 -i 0 -d 30000000`，注意这里的d降低了这么多是因为之前PID计算公式中PID周期表达错误（应该是0.01即10ms而不是0.005的5ms），所以对应的D参数减小一半，对应于原参数的60000000。

本次主要是增大了驱动器电流增大了惯量恢复力，效果很明显。

但是遇到一个问题是最开始出现无法再稳定进入超稳态，而是变成随机状态，~~分析原因可能是由于IMU的初始化不稳定。具体归结为上电状态的温度、电压等导致角度融合输出的差异，同时由于上面提到的BUG，PID积分参数错误放大了不稳定态（这一点也可以验证偶尔出现的车子处于非超稳态但过一阵子突然进入超稳态，也即退出了积分饱和状态）。~~应该是任务优先级问题，提升ControlTask任务的优先级后问题似乎解决了。

---

> **PID参数：**
>
> * 互补滤波器参数 [0.999]  
> * ax -p 1000000 -i 0 -d 25000000 -s 4
> * sx -p 0.000005  -i 0.00000002 
>
> **2阶ButterWorth滤波器参数：**
>
> * GYRO_LPF_CUTOFF_FREQ    100
> * ACCEL_LPF_CUTOFF_FREQ    50
>
> **Ctrl-FOC驱动器参数：**
>
> * vel_gain = 0.0003 
> * vel_integrator_gain = 0.006 
> * current_lim = 30

**主要调整**



尝试将驱动器电流加大至50A

angle和setpoint有一个稳定偏差

疑似跟温度有关

