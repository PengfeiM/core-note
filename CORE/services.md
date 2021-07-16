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
### 服务管理类(ServiceManager)
管理CORE节点可用的服务。

#### add_service函数
```py
def add_services(cls, path: str) -> List[str]:
```
查询给定路径下的所有服务。
参数：路径
返回：加载失败的服务列表

初始化加载失败服务列表，**读取要加载的服务。**（[util.load_classes](#utils.py)）
```py
    service_errors = []
    services = utils.load_classes(path, CoreService)
```


## utils.py
**杂项实用程序功能，一些子流程过程的包装器。**

### load_classes
```py
def load_classes(path: str, clazz: Generic[T]) -> T:
```
动态加载CORE所用到的类。
参数：
- path：加载类的路径
- clazz：预期为加载而继承的类类型
返回：加载的类列表

验证路径的合法性：
```py
    logging.debug("attempting to load modules from path: %s", path)
    if not os.path.isdir(path):
        logging.warning("invalid custom module directory specified" ": %s", path)
```
检查是否为sys.path，不是的话就在sys.path中加入路径
```py
    parent_path = os.path.dirname(path)
    if parent_path not in sys.path:
        logging.debug("adding parent path to allow imports: %s", parent_path)
        sys.path.append(parent_path)
```
查询潜在的服务模块，并过滤掉不可用的模块。
```py
    base_module = os.path.basename(path)
    module_names = os.listdir(path)
    module_names = filter(lambda x: _valid_module(path, x), module_names)
    module_names = map(lambda x: x[:-3], module_names)
```
`module_names`中存储所有的模块名称。
引入并添加路径中的所有服务
```py
    classes = []
    for module_name in module_names:
        import_statement = f"{base_module}.{module_name}"
        logging.debug("importing custom module: %s", import_statement)
        try:
            module = importlib.import_module(import_statement)
            members = inspect.getmembers(module, lambda x: _is_class(module, x, clazz))
            for member in members:
                valid_class = member[1]
                classes.append(valid_class)
        except Exception:
            logging.exception(
                "unexpected error during import, skipping: %s", import_statement
            )
```
遍历所有模块名称，如果出错则报错，将错误信息加入log中；
如果没有错误，则将该模块中的所有类加入`classes`中。
最终返回加载的类列表
```py
    return classes
```
