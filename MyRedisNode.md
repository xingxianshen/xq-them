#网络 NETWORK
```bash
bind 127.0.0.1 #绑定的ip
protected-mode yes #保护模式
port 6379 # 默认端口
```
#通用 GENERAL
```bash
daemonize yes # 以守护进程的方式运行，需要自己开启为yes
pidfile /var/run/redis.pid  #如果以后台方式运行，我们就需要指定一个pid文件
# 日志
# Specify the server verbosity level.
# This can be one of:
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably)
# warning (only very important / critical messages are logged)
loglevel notice

```
afefsef发
