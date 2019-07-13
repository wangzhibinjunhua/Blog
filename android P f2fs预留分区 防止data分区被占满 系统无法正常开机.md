##1 android data分区 之前是mount为ext4,现在是f2fs
- f2fs速度比ext4 要快


##2 ext4 mtk的默认做法是kernel在ext4预留了16MB空间,mtk在mount的时候额外再预留了16MB空间

##3 现在一个android P的项目用的f2fs. 发现并没有预留空间结果在测试的时候发现把data分区填满后adb shell dd if=/dev/zero of=/mnt/sdcard/bigfile 重启系统,发现系统无法开机,最终会走到android自救程序 can't load android system 需要恢复出厂清掉data空间才可以正常开机

