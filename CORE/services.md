# services

## __init__.py
节点可用的服务可以放在services目录下，所有再__all__中列出的内容将被CORE的主模块自动加载
从[CORE服务库](#coreservices.py)中导入了 ServiceManager。

### 加载函数
```py
def load():
```
加载[core.services](#services)下的所有服务（[add_services()](#add_services函数)）
返回：加载失败的服务列表
```py
    return ServiceManager.add_services(_PATH)
```

## coreservices.py

### ServiceDependencies类
用于产生服务的启动路径（应该是启动顺序的意思，每次可以并行启动那些服务，哪些服务需要前面的作为依赖启动）。验证服务所需的依赖全部存在。
#### ServiceDependencise.__init__()
```py
def __init__(self, services: List["CoreServiceType"]) -> None:
    self.visited: Set[str] = set()
    self.services: Dict[str, "CoreServiceType"] = {}
    self.paths: Dict[str, List["CoreServiceType"]] = {}
    self.boot_paths: List[List["CoreServiceType"]] = []
    roots = set([x.name for x in services])
    for service in services:
        self.services[service.name] = service
        roots -= set(service.dependencies)
    self.roots: List["CoreServiceType"] = [x for x in services if x.name in roots]
    if services and not self.roots:
        raise ValueError("circular dependency is present")
```

#### ServiceDependencise._search()
```py
def _search(
    self,
    service: "CoreServiceType",
    visiting: Set[str] = None,
    path: List[str] = None,
) -> List["CoreServiceType"]:
    if service.name in self.visited:
        return self.paths[service.name]
    self.visited.add(service.name)
    if visiting is None:
        visiting = set()
    visiting.add(service.name)
    if path is None:
        for dependency in service.dependencies:
            path = self.paths.get(dependency)
            if path is not None:
                break
    for dependency in service.dependencies:
        service_dependency = self.services.get(dependency)
        if not service_dependency:
            raise ValueError(f"required dependency was not provided: {dependency}")
        if dependency in visiting:
            raise ValueError(f"circular dependency, already visited: {dependency}")
        else:
            path = self._search(service_dependency, visiting, path)
    visiting.remove(service.name)
    if path is None:
        path = []
        self.boot_paths.append(path)
    path.append(service)
    self.paths[service.name] = path
    return path
```

#### ServiceDependencise.boot_order()

```py
def boot_order(self) -> List[List["CoreServiceType"]]:
    for service in self.roots:
        self._search(service)
    return self.boot_paths
```


### 服务管理类(ServiceManager)
管理CORE节点可用的服务。
这一句初始化了manager，这个字典用于存储可用的service。
`services: Dict[str, Type["CoreService"]] = {}`

#### ServiceManageradd函数
```py
def add(cls, service: Type["CoreService"]) -> None:
    """
    将服务添加到ServiceManager。
    :param service: service to add
    :return: nothing
    :raises ValueError: when service cannot be loaded
    """
```
说明正在添加的服务。
```py
    name = service.name
    logging.debug("loading service: class(%s) name(%s)", service.__name__, name)
```

检测是否存在冲突
```py
    #检测冲突
    if name in cls.services:
        raise ValueError("duplicate service being added: %s" % name)
```

调用[utils.which](./utils.md#which)验证这个服务所依赖的可执行程序是否存在。
```py
    # validate dependent executables are present
    for executable in service.executables:
        try:
            utils.which(executable, required=True)
        except CoreError as e:
            raise CoreError(f"service({name}): {e}")

```

验证服务是否成功**import**
```py
    # validate service on load succeeds
    try:
        service.on_load()
    except Exception as e:
        logging.exception("error during service(%s) on load", service.name)
        raise ValueError(e)

    # make service available
    cls.services[name] = service

```

#### ServiceManager.get()
查询manager中的服务，如果有则返回服务，没有则报错。
```py
def get(cls, name: str)->Type["CoreService"]:
    service = cls.services.get(name)
    if service is None:
        raise CoreServiceError(f"service({name}) does not exist")
    return service
```

#### add_service函数
```py
def add_services(cls, path: str) -> List[str]:
```
查询给定路径下的所有服务。
参数：路径
返回：加载失败的服务列表

初始化加载失败服务列表，
**读取要加载的服务。**（[util.load_classes](./utils.md#load_classes)）

```py
    service_errors = []
    services = utils.load_classes(path, CoreService)
```

对于加载的服务进行[添加操作](#ServiceManageradd函数)。
同时收集添加失败的服务。
```py
for service in services:
        if not service.name:
            continue
        try:
            cls.add(service)
        except (CoreError, ValueError) as e:
            service_errors.append(service.name)
            //这个LOG的错误信息来自于add函数的异常。
            logging.debug("not loading service(%s): %s", service.name, e)
    return service_errors
```



### CoreServices类
用于沟通节点和服务，为节点添加服务。通常用于将`CoreService`转化为配置API信息。
这个类存活于一个session中，并保存了一些节点的默认服务。
#### CoreServices.__init__()
实例化一个`CoreService`类。
```py
def __init__(self, session: "Session") -> None:
    self.session: "Seesion" = session
    #存储不同节点的默认服务的dict。
    self.default_services: Dict[str, List[str]] = {
        "mdr": ["zebra", "OSPFv3MDR", "IPForward"],
        "PC": ["DefaultRoute"],
        "prouter": [],
        "router": ["zebra", "OSPFv2", "OSPFv3", "IPForward"],
        "host": ["DefaultRoute", "SSH"],
    }
    #用于存储节点id和节点自定义服务的字典。
    self.custom_services: Dict[int Dict[str, "CoreService"]] = {}
```

#### CoreServices.get_default_services()
针对给定的节点类型（node_type）补充默认的服务。
```py
def get_default_services(self, node_type: str) -> List[Type["CoreService"]]
    logging.debug("getting default services for type: %s", node_type)
    results = []
        defaults = self.default_services.get(node_type, [])
        for name in defaults:
            logging.debug("checking for service with service manager: %s", name)
            service = ServiceManager.get(name)
            if not service:
                logging.warning("default service %s is unknown", name)
            else:
                results.append(service)
        return results
```

#### CoreServices.get_service()
根据服务名字获取匹配的服务。如果找到了就返回，找不到返回特定服务
```py
def get_service(self, node_id: int, service_name: str, default_service: bool=False)->"CoreService":
```
检查是否存在`node_id`这个key。如果不存在则初始化一个。
```py
node_services = self.custom_services.setdefault(node_id, {})
```

[查询服务](#servicemanager.get())。
```py
if default_service:
    default = ServiceManager.get(service_name)
```

返回一个服务名称对应的服务类。
```py
return node_services.get(service_name, default)
```


#### CoreServices.add_services()
为一个节点添加服务。
```py
def add_services(self, node: CoreNode, node_type: str, services: List[str] = None) -> None:
"""
参数：
node：节点
node_type：节点类型（pc,router等）
services：节点需要添加的服务名称
"""
```
检测服务是否为空：如果为空，则[补充默认服务](#coreservices.get_default_services())。
```py
if not services:
    logging.info(
        "using default services for node(%s) type(%s)", node.name, node_type
    )
    services = self.default_services.get(node_type, [])
```
放出配置服务的日志，遍历服务列表：
* 调用[self.get_service方法](#coreservices.get_service())获取服务
* 检测服务是否是已知的服务，如果不是则跳过
* 为节点添加获取到的服务（以某个具体服务类的形式）。
```py
logging.info("setting services for node(%s): %s", node.name, services)
for service_name in services:
    service = self.get_service(node.id, service_name, default_service = True)
    if not service:
        logging.warning("unkown service(%s) for node(%s)", service_name, node.name)
        continue;
    node.services.append(service)
```

#### CoreServices.boot_services()
启动节点的所有服务。
参数：需要启动服务的节点。
```py
def boot_services(self, node: Corenode) -> None:
```
获取服务的启动顺序（调用[服务依赖类](#servicedependencies类)中的[boot_order方法](#servicedependencise.boot_order())）。
```py
boot_paths = ServiceDependencies(node.services).boot_order()
```
按顺序将节点需要启动的服务以[批量启动服务函数](#coreservices._boot_service_path())的形式添加到线程池中启动。
```py
funcs = []
for boot_path in boot_paths:
    args = (node, boot_path)
    funcs.append((self._boot_service_path, args, {}))
result, exceptions = utils.threadpool(funcs)
if exceptions:
    raise CoreServiceBootError(*exceptions)
```

#### CoreServices._boot_service_path()
```py
logging.info(
    "booting node(%s) services: %s",
    node.name,
    " -> ".join([x.name for x in boot_path]),
)
```
读取当前步骤需要启动的服务并启动（调用[启动服务函数](#coreservices.boot_service())）。
```py
for service in boot_path:
    service = self.get_service(node.id, service.name, default_service=True)
    try:
        self.boot_service(node, service)
    except Exception as e:
        logging.exception("exception booting service: %s", service.name)
        raise CoreServiceBootError(e)

```

#### CoreServices.boot_service()
为一个节点启动某一项服务。创建私有的文件夹，生成配置文件，执行服务的`startup`命令。
参数为：节点和服务。
```py
def boot_service(self, node: CoreNode, service: "CoreServiceType") -> None:
    logging.info(
        "starting node(%s) service(%s) validation(%s)",
        node.name,
        service.name,
        service.validation_mode.name,
    )
```
创建服务文件夹（如果有需要的话）：
```py
for directory in service.dirs:
    try:
        node.privatedir(directory)
    except (CoreCommandError, ValueError) as e:
        logging.warning(
            "error mounting private dir '%s' for service '%s': %s",
            directory,
            service.name,
            e,
        )
```
[创建服务文件](#coreservices.create_service_files())
```py
self.create_service_files(node, service)
```
启动服务（[服务启动方法](#coreservices.startup_service())）
```py
# 服务启动是否需要等待参数。
wait = service.validation_mode == ServiceMode.BLOCKING
# 读取服务启动状态
status = self.startup_service(node, service, wait)
if status:#服务启动失败，报错
    raise CoreServiceBootError("node(%s) service(%s) error during startup" % (node.name, service.name))
```
如果是以阻塞方式启动的服务则返回，否则执行下面的验证方案。
```py
# blocking mode is finished
if wait:
    return

# timer mode, sleep and return
if service.validation_mode == ServiceMode.TIMER:
    time.sleep(service.validation_timer)
# non-blocking, attempt to validate periodically, up to validation_timer time
elif service.validation_mode == ServiceMode.NON_BLOCKING:
    start = time.monotonic()
    while True:
        status = self.validate_service(node, service)
        if not status:
            break

        if time.monotonic() - start > service.validation_timer:
            break

        time.sleep(service.validation_period)

    if status:
        raise CoreServiceBootError(
            "node(%s) service(%s) failed validation" % (node.name, service.name)
        )
```



#### CoreServices.startup_service()
启动一项节点的服务。
```py
def startup_service(
    self, node: CoreNode, service: "CoreServiceType", wait: bool = False
) -> int:
    """
    Startup a node service.

    :param node: node to reconfigure service for
    :param service: service to reconfigure
    :param wait: determines if we should wait to validate startup
    :return: status of startup
    """
    cmds = service.startup
    if not service.custom:
        cmds = service.get_startup(node)

    status = 0
    for cmd in cmds:
        try:
            node.cmd(cmd, wait)
        except CoreCommandError:
            logging.exception("error starting command")
            status = -1
    return status
```

#### CoreServices.create_service_files()
创建节点服务配置文件
```py
def create_service_files(self, node: CoreNode, service: "CoreServiceType") -> None:
```
获取服务配置文件名以及它是否需要自定义服务。
```py
config_files = service.configs
if not service.custom:
    config_files = servcies.get_config(node)
```
获取配置文件名，并写入配置信息（调用[node](./node.md)中的[nodefile方法](./node.md#corenode.nodefile())）
```py
for file_name in config_files:
    logging.debug(
        "generating service config custom(%s): %s", service.custom, file_name
    )
    #生成配置文件内容
    if service.custom:
        cfg = service.config_data.get(file_name)
        if cfg is None:
            cfg = service.generate_config(node, file_name)

        # cfg may have a file:/// url for copying from a file
        try:
            if self.copy_service_file(node, file_name, cfg):
                continue
        except IOError:
            logging.exception("error copying service file: %s", file_name)
            continue
    else:
        cfg = service.generate_config(node, file_name)
    #将配置未见内容写入文件
    node.nodefile(file_name, cfg)
```




### CoreService类
这个类是所有服务的父类。
定义了一些变量和方法（未必实现）
变量有：

* name：名字（不包含空格）
* executables：可执行文件
* dependencies：服务的依赖服务
* group：服务所属的组
* dirs：每个节点私有的对应服务所需文件夹
* configs：这个服务需要生成的配置文件内容，主要存储文件名，文件内容由[generate_config方法](#coreservice.generate_config())写入。
* config_data：配置文件数据
* startup: 启动命令的List
* shutdown：关闭服务的命令List
* validation：验证服务命令List
* validation_mode：用于确定启动是否成功
* validation_timer：验证服务启动是否成功的超时秒数。
* validation_period：尝试验证服务启动状态间隔
* meta：服务相关的metadata
* custom: bool = False,    custom_needed: bool = False ：自定义配置文本。

#### CoreService.generate_config()
生成配置文件内容的方法。把配置写进文件或者返回给GUI使用。
```py
#抽象方法，需要子类实现。即每个服务子类写出自己的配置方案。
def generate_config(cls, node: CoreNode, filename: str) -> None:
    raise NotImplementedError

```

