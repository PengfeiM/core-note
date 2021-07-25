# api
[toc]


## grpc

### client.py

#### start_session
通过编译出来的*_pb2.py文件，调用[server.py](#server.py)中的[`StartSession`方法](#startsession方法)

### server.py

#### StartSession方法
**仿真开始的入口找到了！！！！**

1. 获取session：通过CoreEmu对象中保存的Int数字<-->Session对象的字典Dict来获取通过数字获取Session.
2. 运行`session.clear`清理sesion可能的遗留内容，并将session设置到初始状态。
3. 设置session状态为`CONFIGURATION_STATE`
4. 设定位置location，参考坐标的修改详见[GUI-py](./GUI-py.md)
5. 添加hooks？   不懂
6. 创建节点，调用[`grpcutils.create_nodes()`](#grpcutils.create_nodes())。
```py
_, exceptions = grpcutils.create_nodes(session, request.nodes)
if exceptions:
    exceptions = [str(x) for x in exceptions]
    return core_pb2.StartSessionResponse(result=False, exceptions=exceptions)
```
7. 配置EMANE
8. 无线配置
9. 移动性配置
10. 配置服务配置
11. 服务文件配置
12. 创建连接
13. 不对称连接？？？
14. 设置session状态为`EventTypes.INSTANTIATION_STATE`
15. 启动服务。

### grpcutils.py

#### grpcutils.create_nodes()
这个函数遍历节点信息列表，然后把一系列的[session](./emulator.md#session.py)中的[`add_node`](./emulator.md#session.add_node)函数变成命令放到[utils](./utils.md)中的[`threadpool`](./utils.md#threadpool)中自动化执行。
```py
def create_nodes(
    session: Session, node_protos: List[core_pb2.Node]
) -> Tuple[List[NodeBase], List[Exception]]:
    """
    Create nodes using a thread pool and wait for completion.

    :param session: session to create nodes in
    :param node_protos: node proto messages
    :return: results and exceptions for created nodes
    """
    funcs = []
    for node_proto in node_protos:
        _type, _id, options = add_node_data(node_proto)
        _class = session.get_node_class(_type)
        args = (_class, _id, options)
        funcs.append((session.add_node, args, {}))
    #这里start为计时，为了计算完成节点创建花了多长时间。
    start = time.monotonic()
    results, exceptions = utils.threadpool(funcs)
    total = time.monotonic() - start
    logging.debug("grpc created nodes time: %s", total)
    return results, exceptions
```
## tlv

### coreserver.py
初始化一个CORE Server：
* 实例化[CoreEmu](./emulator.md#coreemu.py)
* 读取配置
* 初始化TCP服务。
























