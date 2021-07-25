# configservice

## manager.py

### ConfigServiceManager类
服务配置管理器

#### add函数
把服务添加到manager中。
看起来流程和[servics](./services.md)中的一样
```py
    def add(self, service: Type[ConfigService]) -> None:
        """
        Add service to manager, checking service requirements have been met.

        :param service: service to add to manager
        :return: nothing
        :raises CoreError: when service is a duplicate or has unmet executables
        """
        name = service.name
        logging.debug(
            "loading service: class(%s) name(%s)", service.__class__.__name__, name
        )

        # avoid duplicate services
        if name in self.services:
            raise CoreError(f"duplicate service being added: {name}")

        # validate dependent executables are present
        for executable in service.executables:
            try:
                utils.which(executable, required=True)
            except CoreError as e:
                raise CoreError(f"config service({service.name}): {e}")

            # make service available
        self.services[name] = service
```

#### load函数
搜索该目录下的服务，并将它们添加到待管理的清单中。
```py
"""
    :param path: path to search configurable services
    :return: list errors when loading and adding services
"""
    path = pathlib.Path(path)
    subdirs = [x for x in path.iterdir() if x.is_dir()]
    subdirs.append(path)
    service_errors = []
    for subdir in subdirs:
        logging.debug("loading config services from: %s", subdir)
        services = utils.load_classes(str(subdir), ConfigService)
        for service in services:
            try:
                self.add(service)
            except CoreError as e:
                service_errors.append(service.name)
                logging.debug("not loading service(%s): %s", service.name, e)
        rn service_errors
```