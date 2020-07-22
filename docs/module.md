# 组件
FMT提供了功能强大的系统组件，各种组件提供了系统的大部分功能实现。由于硬件虚拟层（HAL）的引入，各个组件模块可以做到跟硬件平台无关，支持在各个不用的硬件平台上使用。用户只需要根据当前的硬件平台资源和驱动，来决定要使用哪些组件模块。
这里，我们将介绍FMT中常用的一些组件。

## Console控制台
控制台组件供了控制台的输入/输出功能，主要用于调试信息的打印以及命令行的交互功能。console可以被映射到不同的设备上，如串口，USB等。

### 设置控制台设备
调用如下函数来将console设置到`dev_name`对应的设备上。
```c
fmt_err console_set_device(char* dev_name);
```
console默认使用`serial1`设备，当用户连接QGC，并在`Mavlink Console`输入任意字符，console将被自动切换到`mav_console`设备上。

当前console支持的设备名称如下：
- `serialx`： 标准串口设备，默认使用`serial1`
- `mav_console`: 基于mavlink协议的虚拟控制台设备。它将console的输入/输出数据封装为`MAVLINK_MSG_ID_SERIAL_CONTROL`的包，再通过串口或者USB进行发送，以支持QGC的`Mavlink Console`功能，如图所示：

![mav_console](img/mav_console.png)

### 绑定Shell设备
调用如下函数将设备绑定到shell。绑定后，对应设备的输入将送给shell进行命令解析。若传入的`dev=NULL`，则默认将当前`console`的设备绑定到shell。
```c
fmt_err console_mount_shell(rt_device_t dev);
```

### 控制台打印
控制台提供了如下两个函数来打印信息到控制台设备。
```c
uint32_t console_printf(const char* fmt, ...);
uint32_t console_write(const char* content, uint32_t len);
```
常用的`console_printf()`用法类似`printf()`，支持各种格式化打印，且支持中断函数中打印，如：
```c
console_printf("%f %d %x\n", 0.2, 33, 0xFF);
```
`console_write()`函数则提供了控制台数据直接写入的功能，如:
```c
console_write(buffer, sizeof(buffer));
```

## uMCN消息订阅/发布模块
uMCN (Micro Multi-Communication Node) 提供基于订阅/发布模式的安全跨进程通信方式，为FMT各个Task，模块之间进行信息交换的主要方式。

### 定义消息
定义一条新的uMCN非常简单。比如要定义一条新的消息`my_mcn_topic`，其消息（topic）内容为：
```c
typedef struct {
	uint32_t a;
	float b;
	int8_t c[4];
} test_data;
```
首先需要定义消息。在任意地方（通常为发布该topic的源文件的头部）添加如下定义：
```c
MCN_DEFINE(my_mcn_topic, sizeof(test_data));
```
其中`my_mcn_topic`为该topic的名字，`sizeof(test_data)`为消息内容的长度。所以理论上可以用uMCN传递任意消息类型。**注意：**同一个消息只能被定义一次。

然后调用函数`mcn_advertise(McnHub* hub, int (*echo)(void* parameter))`注册该条消息：
```c
mcn_advertise(MCN_ID(my_mcn_topic), _my_mcn_topic_echo);
```
其中`_my_mcn_topic_echo`为该条消息添加*echo*打印函数。当前控制台输入指令`mcn echo my_mcn_topic`，将调用该条消息的*echo*函数来打印消息内容。打印的格式可以根据用户需要自行定义，比如`my_mcn_topic`的*echo*函数可以这样定义：
```c
static int _my_mcn_topic_echo(void* param)
{
	test_data data;
	if(mcn_copy_from_hub((McnHub*)param, &data) == FMT_EOK){
		console_printf("a:%d b:%f c:%c %c %c %c\n", data.a, data.b,
						data.c[0], data.c[1], data.c[2], data.c[3]);
	}
	return 0;
}
```

### 消息发布
发布消息使用函数`mcn_publish(McnHub* hub, const void* data)`。但在发布之前，需要确认是否需要申明该条消息。如果是在定义消息的文件，则无需申明，否则，需要在当前文件的头部添加消息的申明：
```c
MCN_DECLARE(my_mcn_topic);
```
然后调用消息发布函数：
```c
mcn_publish(MCN_ID(my_mcn_topic), &my_data);
```
这里的`my_data`为`test_data`数据类型，跟MCN_DEFINE()消息定义时的数据类型相同。

### 消息订阅
uMCN支持同步/异步消息订阅。同样，在订阅消息的文件中，可能需要添加消息的申明：
```c
MCN_DECLARE(my_mcn_topic);
```
然后调用消息订阅函数`mcn_subscribe(McnHub* hub, MCN_EVENT_HANDLE event_t, void (*cb)(void* parameter))`。其中`cb`为消息发布的回调函数，在每次发布消息时，回调函数将被调用。 **注意：**回调函数将在发布消息的线程中被调用。

`event_t`为用于消息同步的事件句柄，当需要进行同步消息订阅时：
```c
rt_sem_t event = rt_sem_create("my_event", 0, RT_IPC_FLAG_FIFO);
McnNode_t my_nod = mcn_subscribe(MCN_ID(my_mcn_topic), event, NULL);
```
如果不需要进行同步消息订阅：
```c
McnNode_t my_nod = mcn_subscribe(MCN_ID(my_mcn_topic), NULL, NULL);
```

### 消息读取
如果是采用同步订阅的方式，则采用如下方式读取消息。`mcn_poll_sync()`函数将挂起当前线程，直到收到新的消息或者等待超时。
```c
test_data read_data;
if(mcn_poll_sync(my_nod, RT_WAIT_FOREVER)){
	mcn_copy(MCN_ID(my_mcn_topic), my_nod, &read_data);
}
```
如果是采用一部订阅方式，使用如下的方式读取消息。`mcn_poll()`会立马返回当前是否有新的消息发布。
```c
test_data read_data;
if(mcn_poll(my_nod){
	mcn_copy(MCN_ID(my_mcn_topic), my_nod, &read_data);
}
```

### uMCN指令
FMT内置uMCN的指令，指令用法如下：
```shell
msh />mcn
usage: mcn [OPTION] ACTION [ARGS]

Action:
 list  List all uMCN topics.
 echo  Echo a uMCN topic.

Option:
 -c, --cnt     Set topic echo count, e.g, -c=10 will echo 10 times.
 -p, --period  Set topic echo period(ms), -p=0 inherits topic period. The default period is 500ms.
```
输入`mcn list`将显示当前系统所有的topic
```shell
msh />mcn list
      Topic       #SUB   Freq(Hz)   Echo
---------------- ------ ---------- ------
usb_status          1      21.7     true
sensor_imu          1      500.0    true
sensor_mag          1      100.0    true
sensor_baro         1      100.0    true
sensor_gps          1      10.0     true
ins_output          3      500.0    true
fms_output          2      125.0    false
control_output      2      250.0    false
pilot_cmd           1       0.0     true
```
其中**Topic**为消息名称，**#SUB**为消息订阅者数量，**Freq**为消息发布频率，**Echo**表示该条消息是否提供了*echo*打印函数。

对于提供了*echo*函数的消息，可以输入`mcn echo [topic_name]`来打印消息。比如输入
```shell
echo sensor_imu
```
将打印IMU的数据
```shell
msh />mcn echo sensor_imu
gyr:-0.000870 -0.002555 -0.003110 acc:0.011333 0.083641 -9.862786
gyr:-0.001118 -0.001791 0.001414 acc:-0.067505 -0.048334 -9.854862
gyr:0.002485 0.005287 0.005572 acc:-0.040842 -0.008587 -9.773908
gyr:-0.000103 -0.001102 0.003385 acc:-0.092554 -0.013108 -9.854832
gyr:-0.001078 0.002592 -0.003151 acc:-0.040336 0.019121 -9.748902
......
```
`mcn echo`指令支持`-c`和`-p`Option来设置消息打印的**条数**和**频率**，默认打印频率500ms。 比如如下指令将打印10条消息，打印频率为100ms。`-p=0`表示按照消息发布的频率进行打印。

```shell
msh />mcn echo -c=10 -p=100 sensor_mag
mag:0.274922 -0.024422 -0.159042
mag:0.276737 -0.022779 -0.159433
mag:0.274406 -0.024486 -0.161853
mag:0.276287 -0.023486 -0.159781
mag:0.275939 -0.022636 -0.159832
mag:0.274942 -0.024098 -0.160137
mag:0.275403 -0.023519 -0.159008
mag:0.278422 -0.024728 -0.158227
mag:0.275549 -0.023282 -0.160785
mag:0.274039 -0.025209 -0.159165
```

## param参数模块
param参数模块提供了参数的读取，存储和修改功能。参数将以xml文件的形式存储在存储设备，如SD卡上。系统上电会，会读取参数文件，并加载文件中的参数。若参数文件不存在，则使用默认的参数值。参数也会被记录到`blog`中，供Simulink模型使用。目前参数支持的数据类型包括
```c
enum param_type_t {
	PARAM_TYPE_INT8 = 0,
	PARAM_TYPE_UINT8,
	PARAM_TYPE_INT16,
	PARAM_TYPE_UINT16,
	PARAM_TYPE_INT32,
	PARAM_TYPE_UINT32,
	PARAM_TYPE_FLOAT,
	PARAM_TYPE_DOUBLE,
	PARAM_TYPE_UNKNOWN = 0xFF
};
```

### 参数定义
参数以Group为单位，其中每个Group可以包含一到多个参数。例如，定义新的*float*类型的参数`my_param1`和*uint32*类型的参数`my_param2`，它们的Group名称为`my_group`，那么可以按如下步骤添加新的参数：
1. 申明Group: 在`param.h`中申明新的组`my_group`
```c
typedef struct {
	......
	param_group_t	PARAM_GROUP(my_group);
} param_list_t;
```
2. 定义Group: 在`param.c`中定义新的组`my_group`
```c
param_list_t param_list = { 
    ......
    PARAM_DEFINE_GROUP(my_group),
};
```
3. 申明参数: 在`param.h`中添加新的参数申明
```c
typedef struct {
	PARAM_DECLARE(my_param1);
	PARAM_DECLARE(my_param2);
} PARAM_GROUP(my_group);
```
4. 定义参数：在`param.c`中定义新的参数
```c
PARAM_GROUP(my_group) PARAM_DECLARE_GROUP(my_group) = \
{
	PARAM_DEFINE_FLOAT(my_param1, 0.5),
	PARAM_DEFINE_UINT32(my_param2, 1),
};
```
以上就定义了一个两个新的参数`my_param1`（默认值0.5）和`my_param2`（默认值1）。如果是在已有Group中添加新的参数，那么则可以省去步骤1和步骤2.

### 参数读取
参数模块提供了如下宏定义来快速获取参数的值,需要选择对应参数类型的宏并传入组名称和参数名称. 
```c
#define PARAM_GET_INT8(_group, _name)
#define PARAM_GET_UINT8(_group, _name)
#define PARAM_GET_INT16(_group, _name)
#define PARAM_GET_UINT16(_group, _name)
#define PARAM_GET_INT32(_group, _name)
#define PARAM_GET_UINT32(_group, _name)
#define PARAM_GET_FLOAT(_group, _name)
#define PARAM_GET_DOUBLE(_group, _name)
```

除此之外,还可以通过如下函数来获取参数:
```c
param_t* param_get(char* group_name, char* param_name);
param_t* param_get_by_name(const char* param_name);
param_t* param_get_by_full_name(const char* group_name, const char* param_name);
```
这些函数使用轮询的方式查找与`group_name`和`param_name`匹配的参数. 当参数较多时,查找速度较慢,不建议在程序中使用,仅供系统指令(syscmd)模块使用.

### 参数设置
类似参数读取,参数设置也提供了对应的宏来快速设置参数的值:
```c
#define PARAM_SET_INT8(_group, _name, _val)
#define PARAM_SET_UINT8(_group, _name, _val)
#define PARAM_SET_INT16(_group, _name, _val)
#define PARAM_SET_UINT16(_group, _name, _val)
#define PARAM_SET_INT32(_group, _name, _val)
#define PARAM_SET_UINT32(_group, _name, _val)
#define PARAM_SET_FLOAT(_group, _name, _val)
#define PARAM_SET_DOUBLE(_group, _name, _val)
```
同样,也可以使用如下函数来设置与`group_name`和`param_name`匹配的参数.
```c
fmt_err param_set_val(param_t* param, void* val);
fmt_err param_set_val_by_name(char* param_name, void* val);
fmt_err param_set_val_by_full_name(char* group_name, char* param_name, void* val);
fmt_err param_set_string_val(param_t* param, char* val);
fmt_err param_set_string_val_by_name(char* param_name, char* val);
fmt_err param_set_string_val_by_full_name(char* group_name, char* param_name, char* val);
```
参数设置完后,默认不会保存到参数文件.如需将当前参数保存,需要调用`param_save(char* path)`函数,其中`path`为参数文件的路径名称,默认路径为`PARAM_FILE_NAME`.

### param指令
FMT提供了`param`指令来对参数进行操作,其基本用法如下所示:
```shell
msh />param
usage: param [OPTION] ACTION [ARGS]

Action:
 list   List group(s) parameters.
 group  List all parameter groups.
 set    Set parameter.
 get    Get parameter.
 save   Save parameters to file.
 load   Load parameters from file.

Option:
 -s, --save  Save parameter value.
```

- `param list`: 显示所有的Group和Group中的Parameter:
```shell
msh />param list
SYSTEM:
           BLOG_MODE: 0
CALIB:
          GYRO0_XOFF: 0.000000
          GYRO0_YOFF: 0.000000
          GYRO0_ZOFF: 0.000000
		  ......
```
如果在list后加上Group的名称,则只显示对应Group的参数:
```shell
msh />param list SYSTEM
SYSTEM:
           BLOG_MODE: 0
```

- `param group`: 显示所有的Group:
```shell
msh />param group
  Parameter Groups
--------------------
SYSTEM
CALIB
```

- `param set`: 设置参数的值,其用法如下.其中方括号表示可选项,尖括号为必填项.
```shell
param set [group] <parameter> <value>
```
比如要将`BLOG_MODE`设置为2,可以使用如下指令:
```shell
param set SYSTEM BLOG_MODE 2
```
也可以简写为如下指令.但是注意,由于这里没有指定Group,如果存在两个相同名字但是在不同Group的参数,则会写入第一个找到的参数.
```
param set BLOG_MODE 2
```