#有个项目要求能动态生成属性值,包括ro属性.记录下几个坑
##1. android 系统property机制
- ro开头属性值是只读属性,只能设置一次不能修改,大部分都持久化在build.prop文件中,开机在init进程中去load
- persist开头持久化在/data文件下
##2.动态生成ro属性,取代build.prop中的值
- build.prop的值是系统init去加载的,于是在init中找到load build.prop地方去修改此值即可
	
##3 坑一:
- android O及更早版本,system/core/init是属于boot.img的,但是在P版本上google将init以及init.rc移到system.img了,于是gms测试替换gsi后,init的修改部分就失效了
只能在vendor下面的init.rc去实现

##4 坑二:
- vendor下面init.rc ,自己定义的一个文件init.xx.rc权限没有init.mt6761.rc的权限高,只能在init.mt6761.rc里面去设置vendor的ro属性
	
##5.坑三:
- on property触发时间是在系统property load完之后才会去触发,并不是on property的某个属性值一被设置就触发. 所以build.prop的ro在这个之前,只能在编译时将buildinfo.sh你的属性值注释掉.

