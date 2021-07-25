# node
[toc]

## base.py
### NodeBase类
定义了节点的基础类，
NodeBase类。

#### NodeBase-初始化

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
- up：false，控制**startup**操作。
- net_client：实例化一个LinuxNetClient或者OVSNetClient

#### NodeBase-startup
抽象函数
#### NodeBase-shutdown
抽象函数
#### NodeBase.host_cmd()
除了server参数，其他参数主要与命令内容和执行相关。
根据server参数，决定是在本地服务器还是在远程服务器执行命令，并返回执行结果或者报错信息。
```py
def host_cmd(
    self,
    args: str,
    env: Dict[str, str] = None,
    cwd: str = None,
    wait: bool = True,
    shell: bool = False,
) -> str:
    """
    Runs a command on the host system or distributed server.
     :param args: command to run
    :param env: environment to run command with
    :param cwd: directory to run command in
    :param wait: True to wait for status, False otherwise
    :param shell: True to use shell, False otherwise
    :return: combined stdout and stderr
    :raises CoreCommandError: when a non-zero exit status occurs
    """
    if self.server is None:
        return utils.cmd(args, env, cwd, wait, shell)
    else:
        return self.server.remote_cmd(args, env, cwd, wait)
```
#### NodeBase-setposition
参数为：x, y, z坐标。
设置（x, y, z）坐标。
坐标改变返回 `true`，否则返回`false`。

#### NodeBase.getposition
返回一个三元Turple，即三维坐标值。



#### NodeBase-get_iface
根据`iface_id`返回一个`CoreInterface`。
#### NodeBase-get_ifaces
返回一个`CoreInterface`列表

#### NodeBase-get_iface_id
根据接口返回id

#### NodeBase-next_iface_id
返回一个接口列表中未使用的接口id。

#### NodeBase-links
返回`LinkData`列表，
这个方法返回为空，可能后面会被子类重写。


### CoreNodeBase类
继承了[NodeBase类](#nodebase类)
#### CoreNodeBase.__init__()
```py
def __init__(
    self,
    session: "Session",
    _id: int = None,
    name: str = None,
    server: "DistributedServer" = None,
) -> None:
```
调用[父类的初始化方法](#nodebase-初始化)
然后初始化需要配置的服务列表，
设置可选参数：节点缓存文件目录。
```py
super().__init(session, _id, name, server)__
self.config_services: Dict[str, "ConfigService"] = {}
self.nodedir: Optional[str] = None
self.tmpnodedir: bool = False
```

#### CoreNodeBase.makenodedir()
创建一个节点缓存文件夹（调用父类[NodeBase](#nodebase类)中的[host_cmd方法](#nodebase.host_cmd())）。
```py
def makenodedir(self) -> None:
    """
    Create the node directory.

    :return: nothing
    """
    if self.nodedir is None:
        self.nodedir = os.path.join(self.session.session_dir, self.name + ".conf")
        self.host_cmd(f"mkdir -p {self.nodedir}")
        self.tmpnodedir = True
    else:
        self.tmpnodedir = False
```


### CoreNode类
继承了[CoreNodeBase类](#CoreNodeBase类)

#### CoreNode.__init__()
```py
def __init__(
    self,
    session: "Session",
    _id: int = None,
    name: str = None,
    nodedir: str = None,
    server: "DistributedServer" = None,
) -> None:
```
调用父类的[构建方法（初始化方法）](#corenodeBase.__init__())
```py
super().__init__(session, _id, name, server)
```
根据参数设置节点缓存目录
```py
self.nodedir: Optional[str] = nodedir
```
设置控制频道名称？
```py
self.ctrlchnlname: str = os.path.abspath(
    os.path.join(self.session.session_dir, self.name)
)
```
声明自身要绑定的`VnodeClient`和进程号
```py
self.client: Optional[VnodeClient] = None
        self.pid: Optional[int] = None
```
看不懂
```py
self.lock: RLock = RLock()
self._mounts: List[Tuple[str, str]] = []
```
创建自身需要绑定的`LinuxNetClient`
```py
self.node_net_client: LinuxNetClient = self.create_node_net_client(
    self.session.use_ovs()
)
```

#### CoreNode.startup()
通过使用可以分配新namespace的vnoded process启动一个新的`namespace`节点。设置hostname。Bring up the loopback device？不懂
```py
def startup(self) -> None:
```
为这个节点创建缓存文件夹（调用父类[CoreNodeBase](#CoreNodeBase类)中的）[makenodedir()方法](#CoreNodeBase.makenodedir())
```py
self.makenodedir()
```
检测该节点是否已经启动过，是则报错
```py
if self.up:
    raise ValueError("starting a node that is already up")
```

使用vnoded为该节点创建新的namespace。同时读取其**进程号pid**
vnode命令为：
```py
vnoded = (
    f"{VNODED} -v -c {self.ctrlchnlname} -l {self.ctrlchnlname}.log "
    f"-p {self.ctrlchnlname}.pid"
)
if self.nodedir:
    vnoded += f" -C {self.nodedir}"
env = self.session.get_environment(state=False)
env["NODE_NUMBER"] = str(self.id)
env["NODE_NAME"] = str(self.name)

output = self.host_cmd(vnoded, env=env)
self.pid = int(output)
logging.debug("node(%s) pid: %s", self.name, self.pid)
```
上面代码中，vnoded命令为：
```bash
vnoded -v -c {self.ctrlchnlname} -l {self.ctrlchnlname}.log -p {self.ctrlchnlname}.pid
```
vnoded帮助：
```bash
Usage: vnoded [-h|-V] [-v] [-n] [-C <chdir>] [-l <logfile>] [-p <pidfile>] -c <control channel>

Linux namespace container server daemon runs as PID 1 in the container. 
Normally this process is launched automatically by the CORE daemon.

Options:
  -h, --help  show this help message and exit
  -V, --version  show version number and exit
  -v  enable verbose logging
  -n  do not create and run daemon within a new network namespace (for debug)
  -C  change to the specified <chdir> directory
  -l  log output to the specified <logfile> file
  -p  write process id to the specified <pidfile> file
  -c  establish the specified <control channel> for receiving control commands
```
所以，这个创建容器的命令是：创建namespace，启用log，创建控制channel和对应的日志存储路径。

创建 `vnode client`，（实例化了一个[VnodeClient](#VnodeClient类)，用于来连接节点和主机之间的控制通道（channel））
```py
self.client = VnodeClient(self.name, self.ctrlchnlname)
```

启动loopback interface（调用[deviceup方法](#LinuxNetClient.device_up())）。
```py
logging.debug("bringing up loopback interface")
self.node_net_client.device_up("lo")
```

设置节点的主机名（调用[set_hostname方法](#LinuxNetClient.set_hostname())）
```py
logging.debug("setting hostname: %s", self.name)
self.node_net_client.set_hostname(self.name)
```

标记该节点已经启动：
```py
self.up = True
```

创建私有文件夹（调用[privatedir方法](#CoreNode.privatedir())）
```py
self.privatedir("/var/run")
self.privatedir("/var/log")
```





#### CoreNode.create_node_net_client()
创建节点的`LinuxNetClient`
```py
def create_node_net_client(self, use_ovs: bool) -> LinuxNetClient:
    return get_net_client(use_ovs, self.cmd)
```


#### CoreNode.cmd()
用来给节点的 "NetClient" 执行命令行的参数。
参数：

* args：需要运行的命令
* wait：True：等待状态，False：立即执行
* shell：True to use shell, False otherwise

返回：标准输出或者报错。

```py
def cmd(self, args: str, wait: bool = True, shell: bool = False) -> str:
```
根据`server`参数确定在哪个服务器上执行命令。
```py
if self.server is None:
        return self.client.check_cmd(args, wait=wait, shell=shell)
    else:
        args = self.client.create_cmd(args, shell)
        return self.server.remote_cmd(args, wait=wait)
```

#### CoreNode.privatedir()
```py
def privatedir(self, path: str) -> None:
    """
    Create a private directory.

    :param path: path to create
    :return: nothing
    """
    if path[0] != "/":
        raise ValueError(f"path not fully qualified: {path}")
    hostpath = os.path.join(
        self.nodedir, os.path.normpath(path).strip("/").replace("/", ".")
    )
    self.host_cmd(f"mkdir -p {hostpath}")
    self.mount(hostpath, path)
```

#### CoreNode.mount()
```py
def mount(self, source: str, target: str) -> None:
    """
    Create and mount a directory.

    :param source: source directory to mount
    :param target: target directory to create
    :return: nothing
    :raises CoreCommandError: when a non-zero exit status occurs
    """
    source = os.path.abspath(source)
    logging.debug("node(%s) mounting: %s at %s", self.name, source, target)
    self.cmd(f"mkdir -p {target}")
    self.cmd(f"{MOUNT} -n --bind {source} {target}")
    self._mounts.append((source, target))
```

#### CoreNode.new_iface()
创建一个新的网络接口
```py
def new_iface(
    self,net: "CoreNetworkBase", iface_data: InterfaceDate
) -> CoreInterface:
```

## client.py

### VnodeClient类

#### VnodeClient.__init__()
读取name和控制channel name
```py
def __init__(self, name: str, ctrlchnlname: str) -> None:
    """
    Create a VnodeClient instance.

    :param name: name for client
    :param ctrlchnlname: control channel name
    """
    self.name: str = name
    self.ctrlchnlname: str = ctrlchnlname

```





#### CoreNode.hostfilename()
根据当前节点的`nodedir`返回文件的绝对路径。
```py
def hostfilename(self, filename: str) -> str:
    """
    Return the name of a node"s file on the host filesystem.

    :param filename: host file name
    :return: path to file
    """
    dirname, basename = os.path.split(filename)
    if not basename:
        raise ValueError(f"no basename for filename: {filename}")
    if dirname and dirname[0] == "/":
        dirname = dirname[1:]
    dirname = dirname.replace("/", ".")
    dirname = os.path.join(self.nodedir, dirname)
    return os.path.join(dirname, basename)
```


#### CoreNode.nodefile()
根据给定的模式创建文件。
```py
def nodefile(self, filename: str, contents: str, mode: int = 0o644) -> None:
```
根据文件名获取该文件的绝对路径（[self.hostfilename方法](#corenode.hostfilename())），并拆分为文件夹路径和文件名。
```py
hostfilename = self.hostfilename(filename)
dirname, _basename = os.path.split(hostfilename)
```
在节点所在的服务器上添加文件夹。
```py
if self.server is None:
    if not os.path.isdir(dirname):
         os.makedirs(dirname, mode=0o755)
    with open(hostfilename, "w") as open_file:
        open_file.write(contents)
        os.chmod(open_file.name, mode)
else:
    self.host_cmd(f"mkdir -m {0o755:o} -p {dirname}")
    self.server.remote_put_temp(hostfilename, contents)
    self.host_cmd(f"chmod {mode:o} {hostfilename}")
logging.debug("node(%s) added file: %s; mode: 0%o", self.name, hostfilename, mode)
```




## netclient.py

### get_net_client()
根据`use_ovs`参数，给节点创建`OvsNetClient`或者`LinuxNetClient`。
```py
def get_net_client(use_ovs: bool, run: Callable[..., str]) -> LinuxNetClient:
    """
    Retrieve desired net client for running network commands.

    :param use_ovs: True for OVS bridges, False for Linux bridges
    :param run: function used to run net client commands
    :return: net client class
    """
    if use_ovs:
        return OvsNetClient(run)
    else:
        return LinuxNetClient(run)
```

### LinuxNetClient类

#### LinuxNetClient.__init__()
啊，明白了，这个东西就像是把[CoreNode类](#CoreNode类)中定义的[cmd函数](#CoreNode.cmd())作为参数传递给LinuxNetClient来使用。
```py
"""
Create LinuxNetClient instance.
:param run: function to run commands with
"""
self.run: Callable[..., str] = run
```
#### LinuxNetClient.create_bridge()
```py
def create_bridge(self, name: str) -> None:
    """
    Create a Linux bridge and bring it up.

    :param name: bridge name
    :return: nothing
    """
    self.run(f"{IP} link add name {name} type bridge")
    self.run(f"{IP} link set {name} type bridge stp_state 0")
    self.run(f"{IP} link set {name} type bridge forward_delay 0")
    self.run(f"{IP} link set {name} type bridge mcast_snooping 0")
    self.run(f"{IP} link set {name} type bridge group_fwd_mask 65528")
    self.device_up(name)
```

#### LinuxNetClient.device_up()
运行命令行，启动对应的网络接口。
```py
def device_up(self, device: str) -> None:
    """
    Bring a device up.

    :param device: device to bring up
    :return: nothing
    """
    self.run(f"{IP} link set {device} up")
```

#### LinuxNetClient.set_hostname()
```py
def set_hostname(self, name: str) -> None:
    """
    Set network hostname.

    :param name: name for hostname
    :return: nothing
    """
    self.run(f"hostname {name}")
```



### OvsNetClient
继承了[LinuxNetClient类](#LinuxNetClient类)













