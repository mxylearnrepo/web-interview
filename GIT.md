[fork了别人的项目之后如何保持同步更新-实操一遍](https://blog.csdn.net/wuzhongqiang/article/details/103227170)
[【突发】解决remote: Support for password authentication was removed on August 13, 2021. Please use a perso](https://blog.csdn.net/yjw123456/article/details/119696726)

终端设置走代理: 
```shell
 # 配置http访问的
export https_proxy=http://127.0.0.1:7890
# 配置https访问的
export http_proxy=http://127.0.0.1:7890

# 配置http和https访问 (外部控制设置)(也是盲猜)
export all_proxy=socks5://127.0.0.1:9090

# 取消代理
unset http_proxy
unset https_proxy
```
