# emulator
[toc]


## coreemu.py

定义了**CoreEmu**类，该类用于定义创建和配置CORE session和node的逻辑。

### \_\_init__函数
```py
def __init__(self, config: Dict[str, str] = None) -> None:
```
给予程序较大的读写权限创建了一个CoreEmu对象，参数为配置的选项。
```py
    os.umask(0)
```
读取配置
```py
    if config is None:
            config = {}
        self.config: Dict[str, str] = config
```
session 管理
```py
    self.sessions: Dict[int, Session] = {}
```
加载服务（函数：[load_service](#加载服务函数)）并记录加载错误的类。
```py
    self.service_errors: List[str] = []
    self.load_services() #包括自定义服务和默认服务。
```

配置服务，这里配置服务的流程和上面加载服务的流程是一样的，不知道为啥。而且服务内容大致相同。
```py
    self.service_manager: ConfigServiceManager = ConfigServiceManager()
    ##这里调用了load函数，这里的__file__是指configservices的绝对路径。
    config_services_path = os.path.abspath(os.path.dirname(configservices.__file__))
    self.service_manager.load(config_services_path)
    custom_dir = self.config.get("custom_config_services_dir")
    if custom_dir:
        self.service_manager.load(custom_dir)
```
- 实例化[ConfigServiceManager](./configservice.md#ConfigServiceManager类)
- 加载服务([load函数](./configservice.md#load函数))

上面两套服务加载内容大致一样，**core-pygui**和**core-gui**启用的是加载服务中的`core.services`
对比一下上面的加载服务和配置服务中涉及到的服务。

|       services        | configservice |
| :-------------------: | ------------- |
|      OvsService       |               |
|      ryuService       |               |
|         ucarp         |               |
|   emane-transported   |               |
|         Babel         |               |
|          BGP          |               |
|        OSPFv2         |               |
|        OSPFv3         |               |
|       OSPFv3MDR       |               |
|          RIP          |               |
|        RIPING         |               |
|         Xpimd         |               |
|         zebra         |               |
|          atd          |               |
| DefaultMulticastRoute |               |
|     DefaultRoute      |               |
|      DHCPClient       |               |
|         DHCP          |               |
|          FTP          |               |
|         HTTP          |               |
|       IPForward       |               |
|         radvd         |               |
|      StaticRoute      |               |
|      UsrDefined       |               |
|       FRRBabel        |               |
|                       |               |




### 验证环境函数
```py
def _validate_env(self) _> None:
```

### 加载服务函数
```py
def load_services(self) -> None:
```
加载默认服务（[core.services.load()](./services.md#加载函数)）
```py
    self.service_errors = core.services.load()
```
这个过程即为加载所需服务中的所有类。并验证这些类所以来的软件包是否安装。

加载自定义服务
```py

    service_paths = self.config.get("custom_services_dir")
    logging.debug("custom service paths: %s", service_paths)
    if service_paths:
        for service_path in service_paths.split(","):
            service_path = service_path.strip()
            custom_service_errors = ServiceManager.add_services(service_path)
            self.service_errors.extend(custom_service_errors)
```
- 设置自定义服务路径
- 分别读取路径
- 调用[add_services()](./services.md#add_services函数)函数加载服务中的类
- 记录加载错误的服务。



### shutdown函数

### 创建session

### 删除session



## session.py

### session.add_node
将一个节点添加到session中。
```py
def add_node(
        self, _class: Type[NT], _id: int = None, options: NodeOptions = None
    ) -> NT:
        """
        Add a node to the session, based on the provided node data.

        :param _class: node class to create
        :param _id: id for node, defaults to None for generated id
        :param options: data to create node with
        :return: created node
        :raises core.CoreError: when an invalid node type is given
        """
```
检查是否应该“start”节点，以及rj45的状态。
```py
    start = self.state.should_start()
    enable_rj45 = self.options.get_config("enablerj45") == "1"
    if _class == Rj45Node and not enable_rj45:
        tart = False
```
决定节点ID
```py
# 如果节点ID不存在，则为节点生成下一个可用ID。
if not _id:
    _id = self.next_node_id();
```
检测节点是否有名字，如果没有则生成一个。
```py
if not options:
    options = NodeOptions()
    options.set_position(0, 0)
name  = options.name
if not name:
    name = f"{_class.__name__}{_id}"
```
验证分布式服务器，如果分配的服务器不可用则报错。
```py
server = self.distributed.servers.get(options.server)
if options.server is not None and server is None:
    raise CoreError(f"invalid distributed server: {options.server}")
```
创建节点（[create_node函数](./emulator.md#session.create_node)）：
```py
#输出日志到控制台
logging.info(
    "creating node(%s) id(%s) name(%s) start(%s)",
    _class.__name__,
    _id,
    name,
    start,
)
#节点参数：id,名称，服务器
kwargs = dict(_id = id, name = name, server = server)
#如果节点是容器节点，则配置镜像参数
if _class in CONTAINER_NODES:
    kwargs["image"] = options.image
# 调用创建节点函数
node = self.create_node(_class, start, **kwargs)


```
设置节点的图标和所在的画布
```py
node.icon = options.icon
node.canvas = options.canvas
```

设置节点的位置：
```py
self.set_node_position(node, options)
```

如果是CoreNode或者PhysicalNode为需要服务的节点添加服务
* 判断节点是否是CoreNode或者PhysicalNode。
* 设置node类型（此处可以看出CORE中所有节点都是由一个类实例化出来的，只是根据gui传来的信息设置了具体type，根据type设置服务和图标）。
* [设置服务](./services.md#coreservices.add_services())
* [设置config服务]()//还没看，感觉和上面的重复了。
```py
if isinstance(node, (CoreNode, PhysicalNode)):#判断节点是否是CoreNode或者PhysicalNode。
    node.type = options.model # 获取node类型。
    logging.debug("set node type: %s", node.type)
    # 配置服务
    self.services.add_services(node, node.type, options.services)

    # add config services添加另一种服务，不是很懂区别在哪里。
    logging.info("setting node config services: %s", options.config_services)
    for name in options.config_services:
        service_class = self.service_manager.get_service(name)
        node.add_config_service(service_class)
```

如果是EmaneNet节点并且启用了EMANE。
保证EMANE的默认配置存在。
```py
if isinstance(node, Emanenet) and options.emane:
    model = self.emane.models.get(options.emane) # 获取配置的模型
    if not model:
        raise CoreError(f"node({node.name}) emane model ({options.emane}) does not exist")
    node.model = model(self, node.id)
    if self.state == EventTypes.RUNTIME_STATE:
        self.emane.add_node(node)#如果到了运行阶段，则添加EMANE节点（通过Core中的EMANE模块添加）。
```

需要的话设置默认WLAN配置。~~看起来是移动性配置（并不是）~~ ，看了一下，大概是设置了一个根据范围确定WLAN通断的无限模型。
```py
if isinstance(node, WlanNode):
    self.mobility.set_model_config(_id, BasicRangeModel.name)
```

到了运行阶段（RUNTIME_STATE），启动节点。

* 检查节点是否是CoreNode或者PhysicalNode
* 检查是否是RUNTIME_STATE
* 调用[self.write_nodes()](#session.write_nodes())将节点的信息写入一个”session“文件夹中的”node“文件中
* 调用[self.add_remove_control_iface()](#session.add_remove_control_iface)为节点添加控制端口
* 调用[self.services.boot_services()](services.md#coreservices.boot_services())启动节点的所有服务。
```py
is_boot_node = isinstance(node, (CoreNode, PhysicalNode))
if self.state == RUNTIME_STATE and is_boot_node:
    self.write_nodes()
    self.add_remove_control_iface(node, remove = False)
    self.services.boot_services(node)
```

实例化SDT节点。
```py
self.sdt.add_node(node)
```

返回节点。
```py
return node
```






### session.create_node
创建节点，如果节点的ID与其他节点冲突则报错。
```py
def create_node(
    self, _class: Type[NT], start: bool, *args: Any, **kwargs: Any
) -> NT:
"""
        Create an emulation node.

        :param _class: node class to create
        :param start: True to start node, False otherwise
        :param args: list of arguments for the class to create
        :param kwargs: dictionary of arguments for the class to create
        :return: the created node instance
        :raises core.CoreError: when id of the node to create already exists
        """
```

生成节点，并检查该节点的ID是否被占用，如果是则报错，如果不是则将该节点添加到session的节点列表中，同时根据`start`参数选择是否[启动节点](./node.md)（具体使用哪个函数应该取决于该方法的在该node所属class中的具体实现）。
```py
with self.nodes_lock:
    node = _class(self, *args, **kwargs)
    if node.id in self.nodes:
        node.shutdown()
        raise CoreError(f"duplicate node id {node.id} for {node.name}")
    self.nodes[node.id] = node
if start:
    node.startup()
return node
```

```py
        logging.info(
            "controlnet(%s) prefix(%s) updown(%s) serverintf(%s)",
            _id,
            prefix,
            updown_script,
            server_iface,
        )
        control_net = self.create_node(
            CtrlNet,
            start=False,
            prefix=prefix,
            _id=_id,
            updown_script=updown_script,
            serverintf=server_iface,
        )
        control_net.brname = f"ctrl{net_index}.{self.short_session_id()}"
        control_net.startup()
        return control_net
```



### session.write_nodes()
将节点的信息写入一个”session“文件夹中的”node“文件中
写入的信息有：序号(number/id)，名称(name)，api-type，class-type
```py
def write_nodes(self) -> None:
    file_path = os.path.join(self.session_dir, "nodes")
    try:
        with self.nodes_lock:
            with open(file_path, "w") as f:
                for _id, node in self.nodes.items():
                    f.write(f"{_id} {node.name} {node.apitype} {type(node)}\n")
    except IOError:
        logging.exception("error writing nodes file")
```

### session.delete_node()


### session.get_control_net_prefixes()
这个没啥好分析的，就是读取了配置文件中的`controlnet`信息（从options变量中，如果没配置，则使用代码中默认的）。
```py
def get_control_net_prefixes(self) -> List[str]:
        """
        Retrieve control net prefixes.

        :return: control net prefix list
        """
        p = self.options.get_config("controlnet")
        p0 = self.options.get_config("controlnet0")
        p1 = self.options.get_config("controlnet1")
        p2 = self.options.get_config("controlnet2")
        p3 = self.options.get_config("controlnet3")
        if not p0 and p:
            p0 = p
        return [p0, p1, p2, p3]
```

### session.get_control_net_server_ifaces()
返回一个控制网络服务端的端口列表。应给也是从配置文件中读取（如果没配置，则使用默认的）。
```py
def get_control_net_server_ifaces(self) -> List[str]:
    """
    Retrieve control net server interfaces.

    :return: list of control net server interfaces
    """
    d0 = self.options.get_config("controlnetif0")
    if d0:
        logging.error("controlnet0 cannot be assigned with a host interface")
    d1 = self.options.get_config("controlnetif1")
    d2 = self.options.get_config("controlnetif2")
    d3 = self.options.get_config("controlnetif3")
    return [None, d1, d2, d3]
```

### session.add_remove_control_net()
`remove`参数控制添加还是删除网桥。`conf_required`参数控制是否必须添加网桥。
```py
def add_remove_control_net(
        self, net_index: int, remove: bool = False, conf_required: bool = True
    ) -> Optional[CtrlNet]:
    logging.debug(
        "add/remove control net: index(%s) remove(%s) conf_required(%s)",
        net_index,
        remove,
        conf_required,
    )
```
获取控制网络前缀（[self.get_control_net_prefixes](#session.get_control_net_prefixes())）
```py
    prefix_spec_list = self.get_control_net_prefixes()
    prefix_spec = prefix_spec_list[net_index]
    if not prefix_spec:
        if conf_required:
            # no controlnet needed
            return None
        else:
            prefix_spec = CtrlNet.DEFAULT_PREFIX_LIST[net_index]
    logging.debug("prefix spec: %s", prefix_spec)
```

获取控制网络服务端端口（[self.get_control_net_server_ifaces](#session.get_control_net_server_ifaces())）
```py
server_iface = self.get_control_net_server_ifaces()[net_index]
```

尝试返回现有控制网桥，如果`remove`参数为真，则删除节点（[self.delete_node](#session.delete_node())）。
```py
        try:
            control_net = self.get_control_net(net_index)
            if remove:
                self.delete_node(control_net.id)
                return None
            return control_net
        except CoreError:
            if remove:
                return None
```

上面没能返回，则这里生成新的控制网桥。
```py
 _id = CTRL_NET_ID + net_index
```

仅对`control net 0`使用 updown 脚本
```py
updown_script = None

if net_index == 0:
    updown_script = self.options.get_config("controlnet_updown_script")
    if not updown_script:
        logging.debug("controlnet updown script not configured")
```

获取控制网络前缀
```py
prefixes = prefix_spec.split()
if len(prefixes) > 1:
    # a list of per-host prefixes is provided
    try:
        # split first (master) entry into server and prefix
        prefix = prefixes[0].split(":", 1)[1]
    except IndexError:
        # no server name. possibly only one server
        prefix = prefixes[0]
else:
    prefix = prefixes[0]
```
创建`CtrlNet`类型的节点并返回。
```py
control_net = self.create_node(
    CtrlNet,
    start=False,
    prefix=prefix,
    _id=_id,
    updown_script=updown_script,
    serverintf=server_iface,
)
control_net.brname = f"ctrl{net_index}.{self.short_session_id()}"
control_net.startup()
return control_net
```







### session.add_remove_control_iface()
如果有”controlnet“参数在配置文件（/etc/core/core.conf）或者”session options“中被配置，则添加控制端口。使用`addremovectrlnet()`来建立或者删除控制网桥。如果`conf_reqd`参数为`False`，则当用户没有配置时仍然创建控制网络（比如：EMANE）
```py
def add_remove_control_iface(self, node: Union[CoreNode, PhysicalNode], net_index: int = 0, remove: bool = False, conf_required: bool = True)
# remove参数控制添加还是删除。
```
获取控制网络节点（调用[self.add_remove_control_net](#session.add_remove_control_net())）
```py
control_net = self.add_remove_control_net(net_index, remove, conf_required)
```
如果参数中的node或者`CtrlNet`node不符合要求或者已经有了控制网络则直接返回。
```py
if not control_net:
    return
if not node:
    return
# ctrl# already exists
if node.ifaces.get(control_net.CTRLIF_IDX_BASE + net_index):
    return
```
在节点上添加一个控制端口，然后构建控制网络（具体细节还不清楚，我暂时用不上就不看了。其中的[`node.new_iface`方法](./node.md#corenode.new_iface())）
```py
try:
    ip4 = control_net.prefix[node.id]
    ip4_mask = control_net.prefix.prefixlen
    iface_data = InterfaceData(
        id=control_net.CTRLIF_IDX_BASE + net_index,
        name=f"ctrl{net_index}",
        mac=utils.random_mac(),
        ip4=ip4,
        ip4_mask=ip4_mask,
    )
    iface = node.new_iface(control_net, iface_data)
    iface.control = True
except ValueError:
    msg = f"Control interface not added to node {node.id}. "
    msg += f"Invalid control network prefix ({control_net.prefix}). "
    msg += "A longer prefix length may be required for this many nodes."
    logging.exception(msg)
```
