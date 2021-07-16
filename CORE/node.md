# node



## NodeBase类
定义了节点的基础类，

### NodeBase-初始化
参数：
- session：session类
- id：节点id
- name：节点名字
- server：服务器参数，用于指定节点位于哪个服务器上。

主要初始化一些参数：
- session：根据输入参数初始化
- id：如果没有，则从session中按顺序获得id。
- name：如果没有，则根据id生成名称
- 服务器：根据参数指定将来被部署在哪个服务器上。
- 类型：无
- 服务：空
- 网口：空
- 网口id：-
- canvas：无
- 图标：无
- 位置：空，实例化一个Position类，但是未指定（x,y,z）参数。
- up：false，不清楚是什么参数暂时
- net_client：实例化一个LinuxNetClient或者OVSNetClient

### NodeBase-startup
抽象函数
### NodeBase-shutdown
抽象函数
### NodeBase-host_cm
除了server参数，其他参数主要与命令内容和执行相关。
根据server参数，决定是在本地服务器还是在远程服务器执行命令，并返回执行结果或者报错信息。

### NodeBase-setposition
参数为：x, y, z坐标。
设置（x, y, z）坐标。
坐标改变返回 `true`，否则返回`false`。

### NodeBase-getposition
返回一个三元Turple，即三维坐标值。



### NodeBase-get_iface
根据`iface_id`返回一个`CoreInterface`。
### NodeBase-get_ifaces
返回一个`CoreInterface`列表

### NodeBase-get_iface_id
根据接口返回id

### NodeBase-next_iface_id
返回一个接口列表中未使用的接口id。

### NodeBase-links
返回`LinkData`列表，
这个方法返回为空，可能后面会被子类重写。




