#  展讯8521 android4.4平台gsensor hal层兼容问题



### 一.hal代码位置

`8521e_dev_kk\vendor\sprd\modules\sensors`



`android.mk `

`#ACCELERATION
ifeq ($(BOARD_ACC_COMPATIBLE),true)
LOCAL_SRC_FILES += acc/AccSensor.cpp
LOCAL_CFLAGS += -DACC_INSTALL_$(BOARD_ACC_INSTALL)
else
ifneq ($(BOARD_HAVE_ACC),NULL)
LOCAL_SRC_FILES += acc/Acc_$(BOARD_HAVE_ACC).cpp
LOCAL_CFLAGS += -DACC_INSTALL_$(BOARD_ACC_INSTALL)
else
LOCAL_CFLAGS += -DACC_NULL
endif #BOARD_HAVE_ACC
endif #BOARD_ACC_COMPATIBLE`

- 1.android.mk内容看出BOARD_ACC_COMPATIBLE 这个宏表面走兼容模式,编译AccSensor.cpp文件

  而BOARD_HAVE_ACC 定义为ic型号,表面只用这一个ic

  下面以mc3xxxx gsensor ic型号为例说明



### 二.分析

- 1.在没有集成hal层代码时,android上层通过android标准api是获取不到gsensor数据

  `mSensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)` 获取为null	 

   					

- 2.发现hal层的sensor库文件并没有编译进去

  `include $(CLEAR_VARS)
  LOCAL_MODULE := sensors.$(TARGET_BOARD_PLATFORM)
  LOCAL_MODULE_PATH := $(TARGET_OUT_SHARED_LIBRARIES)/hw
  LOCAL_MODULE_TAGS := optional`
  
  需要在对应的编译工程mk文件中加入
  
  PRODUCT_PACKAGES +=sensors.$(TARGET_BOARD_PLATFORM) 这句代码,才会编译到这个模块.
  
  生成的文件是sensors.sl8521.so文件

- 3.hal库文件编译进去后,发现还是无法获取到gsensor type

  查看log发现错误`sensor  : couldn't find 'accelerometer' input devicei fd=-1`

  找到对应的代码

  ```
  int SensorBase::openInput(const char *inputName) {
    int fd = -1;
    const char *dirname = "/dev/input";
    char devname[PATH_MAX];
    char *filename;
    DIR *dir;
    struct dirent *de;
    dir = opendir(dirname);
    int path_len = 0;
    if (dir == NULL) return -1;
  
    snprintf(devname, sizeof(devname), "%s", dirname);  // luciddle
    path_len = strlen(devname);
    filename = devname + path_len;
    *filename++ = '/';
    path_len++;
    while ((de = readdir(dir))) {
      if (de->d_name[0] == '.' &&
          (de->d_name[1] == '\0' ||
           (de->d_name[1] == '.' && de->d_name[2] == '\0')))
        continue;
  
      snprintf(filename, PATH_MAX - path_len, "%s", de->d_name);  // luciddle
        ALOGE("[wzb] devname: %s,filename: %s ", devname,filename);
      fd = open(devname, O_RDONLY);
      if (fd >= 0) {
        char name[80];
        if (ioctl(fd, EVIOCGNAME(sizeof(name) - 1), &name) < 1) {
          name[0] = '\0';
        }
         ALOGE("[wzb] name: %s,inputName: %s ", name,inputName);
        if (strlen(name) > 0 && !strcmp(name, inputName)) {
          snprintf(input_name, PATH_MAX, "%s", filename);  // luciddle
          break;
        } else {
          close(fd);
          fd = -1;
        }
      } else
      ALOGE("open %s fail , fd=%d", devname, fd);
    }
    closedir(dir);
    ALOGE_IF(fd < 0, "couldn't find '%s' input devicei fd=%d", inputName , fd);
    return fd;
    
  }
  ```

  发现是 

  ```
   if (strlen(name) > 0 && !strcmp(name, inputName)) {
  ```

  name和inputName不匹配.

  inputName是`accelerometer`. 而name是从ic驱动中获取的名称.发现是 `sensor  : [wzb] name: mc3xxx_accelerometer,inputName: accelerometer` 

  `mc3xxx_accelerometer`是驱动里面定义的名称

  将驱动代码改为 `\#define MC3XXX_INPUT_NAME   "accelerometer"`

  

- 4.重新编译后发现还是无法找到匹配的gsensor.继续分析log. 上面3的错误没有了.

  有个新的错误

  ```
  01-01 08:08:53.099   609   609 D sensor  : AccSensor: sys_chip_info=/sys/class/input/input0/chip_info, input_sysfs_path=/sys/class/input/input0
  01-01 08:08:53.099   609   609 E sensor  : AccSensor: read sysfs file /sys/class/input/input0/chip_info error, No such file or directory
  01-01 08:08:53.099   609   609 D sensor  : AccSensor: size=0
  01-01 08:08:53.099   609   609 E sensor  : AccSensor: getAccChipInfo:chip_info failed!
  01-01 08:08:53.099   609   609 E sensor  : AccSensor: There is no acc sensor match !
  01-01 08:08:53.099   609   609 E sensor  : , please check your acc_sensors array
  ```

  

​      get chip_info failed. 找到对应的代码

```
static void getAccChipInfo(char *input_sys_path) {
  int		size = 0;
  char sys_chip_info[PATH_MAX];

  sprintf(sys_chip_info, "%s/%s",
          input_sys_path, SYSFS_NODE_CHIP_INFO);
  ALOGD("AccSensor: sys_chip_info=%s, input_sysfs_path=%s\n", sys_chip_info,
        input_sys_path);
  memset(chip_info, 0, sizeof(chip_info));
  size = read_sysfs(sys_chip_info, chip_info, sizeof(chip_info));
  ALOGD("AccSensor: size=%d\n", size);

  if (0 == size) {
    ALOGE("AccSensor: %s:chip_info failed!\n", __FUNCTION__);
  } else {
    /*  add string termination symbol  */
    chip_info[size] = '\0';
    ALOGD("AccSensor: %s ,%s\n", __FUNCTION__, chip_info);
  }
}
```

发现是`/sys/class/input/input0/chip_info`读取失败.

此节点是ic驱动中实现的,检查驱动代码发现没有添加此节点

增加代码

```
static DEVICE_ATTR(chip_info   , 	S_IRUGO, 		 mc3xxx_chipinfo_show   , NULL);
static ssize_t mc3xxx_chipinfo_show(struct device *dev,
		struct device_attribute *attr, char *buf)
{

    return sprintf(buf, "%s", "MC34XX");
}
```

此处的chip_info名称需要与AccSensor.cpp中定义的名称一致

```
static const char *mc34xx = "MC34XX";
struct SensorInfo acc_sensors[] = {
  {bmi160, sSensorList_BMI160},
  {bma253, sSensorList_BMA253},
  {kionix_kxtj3, sSensorList_KIONIXKXTJ3},
  {mc34xx, sSensorList_MC34XX},
  {lis2dh, sSensorList_LIS2DH},
  {"", sSensorList_NULL},
};

```



重新编译下载开机.发现能正常匹配到gsensor驱动了

log显示

```
01-01 08:00:23.179   607   607 D sensor  : AccSensor: sys_chip_info=/sys/class/input/input0/chip_info, input_sysfs_path=/sys/class/input/input0
01-01 08:00:23.179   607   607 D sensor  : AccSensor: size=6
01-01 08:00:23.179   607   607 D sensor  : AccSensor: getAccChipInfo ,MC34XX
01-01 08:00:23.179   607   607 D sensor  : numSensors=1, AccSensor::numSensors=1
01-01 08:00:23.179   607   607 E sensor  : wakeIndex=1, flushIndex=2 
01-01 08:00:23.179   607   607 D sensor  : activate handle=0; drv=0
01-01 08:00:23.179   607   607 D sensor  : AccSensor: handle=0,enabled=0
01-01 08:00:23.179   607   607 D sensor  : AccSensor: mEnabled = 0
01-01 08:00:23.179   607   607 I SensorService: ST MC34XX 3-axis Accelerometer
01-01 08:00:23.179   607   607 D SensorService: Max socket buffer size 163840
```



- 5.开机初始化能找到sensor.但是app调用时还是无法获取gsensor数据

  查看main log

  ```
  D/sensor  (  607): int poll__batch(sensors_poll_device_1*, int, int, int64_t, int64_t) handle=0 sampling_period_ns=200ms, max_report_latency_ns=0ms
  E/sensor  (  607): AccSensor: mDelay(-1) delay_ns(200000000)
  E/sensor  (  607): AccSensor: ms(200)
  E/sensor  (  607): SensorBase: write_attr failed to open /sys/class/input/input0/acc_delay (No such file or directory)
  E/sensor  (  607): AccSensor:setDelay fail (No such file or directory)
  D/sensor  (  607): int poll__batch(sensors_poll_device_1*, int, int, int64_t, int64_t) handle=0 sampling_period_ns=200ms, max_report_latency_ns=0ms
  E/sensor  (  607): AccSensor: mDelay(-1) delay_ns(200000000)
  E/sensor  (  607): AccSensor: ms(200)
  E/sensor  (  607): SensorBase: write_attr failed to open /sys/class/input/input0/acc_delay (No such file or directory)
  E/sensor  (  607): AccSensor:setDelay fail (No such file or directory)
  D/sensor  (  607): int poll__flush(sensors_poll_device_1*, int) handle=0
  E/sensor  (  607): reading flush event from pipe
  D/sensor  (  607): activate handle=0; drv=0
  D/sensor  (  607): AccSensor: handle=0,enabled=1
  D/sensor  (  607): AccSensor: set EVIOCSCLOCKID = 7
  E/sensor  (  607): SensorBase: write_attr failed to open /sys/class/input/input0/acc_enable (No such file or directory)
  ```

  发现两处错误

  `/sys/class/input/input0/acc_delay (No such file or directory)` 

  此节点也是由驱动实现,hal层调用时设置gsensor数据上报频率

  `/sys/class/input/input0/acc_delay (No such file or directory)`

  此节点也是由驱动实现,hal层调用时使能/关闭 gsensor

  驱动代码中添加此两个节点

  ```
  static DEVICE_ATTR(acc_enable   , 	S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP,		 mc3xxx_enable_show   , mc3xxx_enable_store);
  static DEVICE_ATTR(acc_delay   , 	S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP, 		 mc3xxx_delay_show   , mc3xxx_delay_store);
  
  
  ```

  

```
static struct attribute *mc3xxx_attributes[] = {
	&dev_attr_delay.attr,
	&dev_attr_enable.attr,
	&dev_attr_range.attr,
	&dev_attr_bandwidth.attr,
	&dev_attr_position.attr,
	&dev_attr_mode.attr,
	&dev_attr_value.attr,
	&dev_attr_reg.attr,
	&dev_attr_product_code.attr,
	&dev_attr_version.attr,
	&dev_attr_step.attr,	
	&dev_attr_shake.attr,
	&dev_attr_chip_info.attr,
	&dev_attr_acc_delay.attr,
	&dev_attr_acc_enable.attr,
	NULL
};
```

```
static ssize_t mc3xxx_enable_show(struct device *dev,
		struct device_attribute *attr, char *buf)
{
	struct i2c_client *client = to_i2c_client(dev);
	struct mc3xxx_data *mc3xxx = i2c_get_clientdata(client);

#ifdef MCUBE_FUNC_DEBUG
	printk( "%s called\n",__func__);
#endif

	return sprintf(buf, "%d\n", atomic_read(&mc3xxx->enable));
}

static ssize_t mc3xxx_enable_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf, size_t count)
{
	unsigned int enable = simple_strtoul(buf, NULL, 10);

#ifdef MCUBE_FUNC_DEBUG
	printk( "%s called %d\n", __func__, enable);
#endif

	if (enable)
		mc3xxx_set_enable(dev, 1);
	else
		mc3xxx_set_enable(dev, 0);

	return count;
}
```

```
static ssize_t mc3xxx_delay_show(struct device *dev,
		struct device_attribute *attr, char *buf)
{
	struct i2c_client *client = to_i2c_client(dev);
	struct mc3xxx_data *mc3xxx = i2c_get_clientdata(client);

#ifdef MCUBE_FUNC_DEBUG
	printk( "%s called\n", __func__);
#endif

	return sprintf(buf, "%d\n", atomic_read(&mc3xxx->delay));
}

static ssize_t mc3xxx_delay_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf, size_t count)
{
	struct i2c_client *client = to_i2c_client(dev);
	struct mc3xxx_data *mc3xxx = i2c_get_clientdata(client);
	unsigned long delay = simple_strtoul(buf, NULL, 10);

#ifdef MCUBE_FUNC_DEBUG
	printk( "%s called\n", __func__);
#endif

	if (delay > MC3XXX_MAX_DELAY)
		delay = MC3XXX_MAX_DELAY;
	//atomic_set(&mc3xxx->delay, delay);
	mc3xxx_set_delay(&(mc3xxx->client->dev),delay);

	return count;
}
```



至此hal代码全部完毕. 使用android标准api可以获取到gsensor数据

```
private void initSensor() {
		mSensorManager=(SensorManager)getSystemService(Context.SENSOR_SERVICE);
		List<Sensor> sensors=mSensorManager.getSensorList(Sensor.TYPE_ALL);
		int sensorsNum=sensors.size();
		Log.e("wzb","sensorNum="+sensorsNum);
		for(int i=0;i<sensorsNum;i++) {
			Log.e("wzb","sensor name="+sensors.get(i).getName());
			Log.e("wzb","sensor type="+sensors.get(i).getType());
		}
		mSensorManager.registerListener(new SensorEventListener() {
			
			@Override
			public void onSensorChanged(SensorEvent event) {
				// TODO Auto-generated method stub
				if(Sensor.TYPE_ACCELEROMETER == event.sensor.getType()) {
					 float[] values = event.values;
		                float ax = values[0];
		                float ay = values[1];
		                float az = values[2];
		                tv.setText("x="+ax+"\n"+"y="+ay+"\n"+"z="+az+"\n");		
				}
			}
			
			@Override
			public void onAccuracyChanged(Sensor sensor, int accuracy) {
				// TODO Auto-generated method stub
				
			}
		}, mSensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER),SensorManager.SENSOR_DELAY_NORMAL);
	}
```















​		









  















