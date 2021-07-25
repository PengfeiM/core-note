# core.utils

## utils.py
**杂项实用程序功能，一些子流程过程的包装器。**

### load_classes
```py
def load_classes(path: str, clazz: Generic[T]) -> T:
```
动态加载CORE所用到的类。
参数：
- path：加载类的路径（`home/user/core-release-7.5.1/daemon/core/services`）
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

### which
```py
def which(command: str, required: bool) -> str:
    """
    Find location of desired executable within current PATH.

    :param command: command to find location for
    :param required: command is required to be found, false otherwise
    :return: command location or None
    :raises ValueError: when not found and required
    """
    found_path = shutil.which(command)
    if found_path is None and required:
        raise CoreError(f"failed to find required executable({command}) in path")
    return found_path
```

### threadpool
自动化执行输入命令（一堆函数）的函数。并返回结果和异常。
```py
def threadpool(
    funcs: List[Tuple[Callable, Iterable[Any], Dict[Any, Any]]], workers: int = 10
) -> Tuple[List[Any], List[Exception]]:
    """
    Run provided functions, arguments, and keywords within a threadpool
    collecting results and exceptions.

    :param funcs: iterable that provides a func, args, kwargs
    :param workers: number of workers for the threadpool
    :return: results and exceptions from running functions with args and kwargs
    """
    with concurrent.futures.ThreadPoolExecutor(max_workers=workers) as executor:
        futures = []
        for func, args, kwargs in funcs:
            future = executor.submit(func, *args, **kwargs)
            futures.append(future)
        results = []
        exceptions = []
        for future in concurrent.futures.as_completed(futures):
            try:
                result = future.result()
                results.append(result)
            except Exception as e:
                logging.exception("thread pool exception")
                exceptions.append(e)
    return results, exceptions
```