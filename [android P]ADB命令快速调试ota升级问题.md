# [android P]ADB命令快速调试ota升级问题
- 1.制作ota差分包,跟以前版本差不多
- 2.将制作好的差分包update.zip push到系统中 放到/cache下面
- 3.adb shell "echo \"--update_package=/cache/update.zip\" > /cache/recovery/command"
adb reboot recovery 进入升级
- 关键:在P版本上,update.zip 放在/data 和/sdcard 下面均不行.  只能放在/cache下用此命令可以升级
- 4.recovery log 在/cache/recovery 可pull出来分析错误
