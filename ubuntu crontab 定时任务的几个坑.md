### ubuntu crontab 定时任务的几个坑

- 1.提示MTA uninstall. 

  > 在crontab -e 首行加入MAILTO="" 即可

- 2.cmd不能加引号

  ```shell
  
  39 15 * * * "/bin/bash /data2/gitback/gitback.sh >> /data2/gitback/gitback.log 2>&1"
  ```

不能执行

改成

```shell
30 2 * * * /bin/bash /data2/gitback/gitback.sh >> /data2/gitback/gitback.log 2>&1

```

这可以正常执行了

