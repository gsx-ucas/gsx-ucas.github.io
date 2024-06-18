---
title: GitHub无法访问、443 Operation timed out
date: 2023-6-13 11:20
tags: 
  - GitHub
  - Troubleshooting
categories:
  - Linux
  - Troubles
---

> 针对以下情况: \
> 1.无法访问github，重启电脑，重置网络都无法正常访问。 \
> 2."科学上网"可以访问到GitHub，但是命令行`git pull、push`等命令无法访问远程库，显示‘Failed to connect to github.com port 443: Operation timed out’。
> 
> 解决方法：修改系统hosts文件 \
> [文章参考](https://juejin.cn/post/6844904193170341896)

# 查询github.com的IP地址
打开该网址[https://www.ipaddress.com/site/github.com](https://www.ipaddress.com/site/github.com)
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/17/172bde579dbb3de8~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)
# 查询github.global.ssl.fastly.net的IP地址
打开该网址[https://www.ipaddress.com/site/github.global.ssl.fastly.net](https://www.ipaddress.com/site/github.global.ssl.fastly.net)
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/17/172bde57a05985dd~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)
# 查询assets-cdn.github.com的IP地址
打开该网址[https://www.ipaddress.com/site/assets-cdn.github.com](https://www.ipaddress.com/site/assets-cdn.github.com)
![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/17/172bde57a0a40370~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)

# 修改hosts文件
Linux和MacOS的hosts文件在/etc/hosts，可以用vim等编辑器打开，需要管理员权限
将刚刚复制的IP地址填写到hosts中，网站名称跟在IP之后，然后保存
```
140.82.113.4 github.com 
199.232.69.194 github.global.ssl.fastly.net
185.199.108.153  assets-cdn.github.com
185.199.109.153  assets-cdn.github.com
185.199.110.153  assets-cdn.github.com
185.199.111.153  assets-cdn.github.com
```

# 刷新DNS
```
sudo killall -HUP mDNSResponder
```

