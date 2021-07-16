# api

## grpc

### client.py

#### start_session
通过编译出来的*_pb2.py文件，调用[server.py](#server.py)中的`StartSession`

### server.py

#### StartSession方法。
**仿真开始的入口找到了！！！！**

1. 获取session
2. 设置session状态为`CONFIGURATION_STATE`
3. 设定位置location
4. 添加hooks？   不懂
5. 创建节点
6. 配置EMANE
7. 无线配置
8. 移动性配置
9. 配置服务配置
10. 服务文件配置
11. 创建连接
12. 不对称连接？？？
13. 设置session状态为`EventTypes.INSTANTIATION_STATE`
14. 启动服务。


