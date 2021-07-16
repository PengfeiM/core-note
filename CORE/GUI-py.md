# core.gui

**配置文件位置：**`home/user/.coregui/config.yaml`


## start_session
1. 读取配置
2. 尝试连接服务器，
3. 尝试开始session.(调用[api.grpc.client.py](./api.md#client.py)中的[start_session](./api.md#start_sessioin))

## appconfig.py
这个文件用于`core-pygui`的配置。
比如调整log文件的命名。
**注**：我把log文件的明明里面加入了时间戳
