# git仓库备份

- 用作容灾备份,防止硬盘损坏导致数据丢失风险

   

  ​	

  
### 备份的机器

- 备份服务器用的是ubuntu系统，地址172.28.1.132，开了ssh服务。

-  在用于备份的用户目录下（假设用户为back，密码为123456），创建一个用于备份的目录，如gitback。

-  在备份目录gitback下创建一个脚本gitback.sh：

- ``` shell
  #!/bin/sh
  giturl="http://172.28.12.215/chenzewei/"
  reslist="besopensource.gitbes2000.git bes2000otaboot.git screenrecorddemo.git StudentVR.gitStudentVR    -.git  launcherscence.git testdir.gitrk3399-kernel.git gvr-android-sdk.git"
  gitbackdir=$PWD
  for resin ${reslist};
  do
       cd ${res}
       git fetch
       cd $gitbackdir
       git clone --mirror ${giturl}${res}
  done
  
  ```



### git仓库服务器

- 增加一个定时任务 执行命令crontab –e

- 在出现的vi编辑界面最后加入一行：

-  ```shell
  0 4 * * * sshpass -p 123456 ssh back@172.28.1.132 "cdgitback && sh gitback.sh"
   ```

- 保存，这个任务会在每天4点执行。

  