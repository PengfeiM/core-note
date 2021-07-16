# emulator



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

配置服务
```py
    self.service_manager: ConfigServiceManager = ConfigServiceManager()
    config_services_path =             os.path.abspath(os.path.dirname(configservices.__file__))
    self.service_manager.load(config_services_path)
    custom_dir = self.config.get("custom_config_services_dir")
    if custom_dir:
        self.service_manager.load(custom_dir)
```
- 实例化[ConfigServiceManager](#ConfigServiceManager类)
- 加载服务([load函数](#load函数))

### 验证环境函数
```py
def _validate_env(self) _> None:
```

### 加载服务函数
```py
def load_services(self) -> None:
```
加载默认服务（[core.services.load()](#加载函数)）
```py
    self.service_errors = core.services.load()
```
这个过程即为加载所需服务中的所有类。

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
- 调用[add_services()](#add_services函数)函数加载服务中的类
- 记录加载错误的服务。



### shutdown函数

### 创建session

### 删除session