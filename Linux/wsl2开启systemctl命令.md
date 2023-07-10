默认情况时wsl2使用systemctl命令会报以下错误：

```shell
 ✘ ⚡ root@hayami ~ $ systemctl
System has not been booted with systemd as init system (PID 1). Can't operate.
```

解决办法:

1. 安装daemonize

   ```
   sudo apt-get install daemonize
   ```

2. 执行以下两句命令开启

   ```shell
   sudo daemonize /usr/bin/unshare --fork --pid --mount-proc /lib/systemd/systemd --system-unit=basic.target
   
   exec sudo nsenter -t $(pidof systemd) -a su - $LOGNAME
   ```

