对于CORE的调试，暂时没啥思路。
虚拟机pycharm直接卡死了。
目前我把CORE的LOG配置改了一下，[修改后的配置文件](./logging.conf)。
把这个文件放在`/etc/core/`目录下就行了（对于ubuntu来说）。
这样的话CORE在运行过程中会将DEBUG级别一下的log全部输出到控制台以及`/home/username/.coredaemon/core-daemon.log`中。

另外，如果想要core-pygui的详细log的话，可以在运行core-pygui如下所示：
```bash
core-pygui -l DEBUG
```
DEBUG是级别，自己可以调，默认是INFO级别，DEBUG>INFO。
