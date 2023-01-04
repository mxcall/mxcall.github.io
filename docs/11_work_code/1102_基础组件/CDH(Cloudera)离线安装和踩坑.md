


## 背景
黑群晖配置太低, cpu/内存都不够, 跑虚拟机ubuntu/centos很卡;
在用maclan解决docker的网络问题后(docker使用家用路由器分配的ip, 避免ip间无法访问), 使得把Nas当虚拟机成为可能;
docker 当成虚拟机的过程中, 一个比较常见的场景是虚拟机中的有些配置好的服务需要开机自启动, 避免每次重启后, 还得手动敲命令启动;
物理机自启动的方案是用systemd, 但是docker无法正常运行systemd, 所以只能另辟蹊径!!!
后期可以尝试docker in docker各种方案, 把docker当成真正的虚拟机使用!!!

## 定义及目标
将docker容器当成虚拟机使用, 在Nas重启后, 自动启动docker容器, 并且自动运行docker内自定义服务

## 业界方案(表格对比)
参考链接: 
https://www.cnblogs.com/ZXdeveloper/p/16021422.html
https://blog.51cto.com/u_11529070/3608203


## 我们的方案
方案一: 启动docker时,以特权方式启动, 然后里面就可以用systemd了
```
# ubuntu
docker run --net=bridge-to-router --ip=192.168.0.251 --name ubuntu18_1 -d  --restart always  --privileged=true  ubuntu:18 /sbin/init
# centos
docker run  --net=bridge-to-router --ip=192.168.0.250 --name centos7_1 -d  --restart always  --privileged=true  centos:7 /usr/sbin/init
```
注意, 特权模式(privileged+/sbin/init)会造成宿主机cpu 负载占用高(agetty进程冲突导致), 参考https://blog.csdn.net/bobpen/article/details/78559263
解决办法: 关闭宿主机和容器的gettty
```python
systemctl stop getty@tty1.service
systemctl mask getty@tty1.service
# 然后再次top查看, 也可跟在 docker run 后面, 如
docker run --net=bridge-to-router --ip=192.168.0.251 --name ubuntu18_1 -d  --restart always  --privileged=true  ubuntu:18 /sbin/init && systemctl stop getty@tty1.service && systemctl mask getty@tty1.service

```

方案二: 写自定义脚本, 存入/root/.bashrc中
ps: cat > filename << EOF  和 cat << EOF > filename  效果相同, 
<<EOF是输入, > 是输出, 写在一起就连接输入和输出!
```
cat >> /root/.bashrc <<EOF
if [ -f /root/startup_run.sh ]; then
  /root/startup_run.sh
fi
EOF

cat >> /root/startup_run.sh <<EOF
service mysql start >>/root/startup_run.log
service ssh start >>/root/startup_run.log
EOF
```

方案三: 同样使用脚本, docker run xxxx  某个脚本, 这个脚本有如下要求:
1. 脚本结尾必须hang住
2. 脚本中前面服务启动, 不能hang住, 否则不会执行后面的相关服务
3. 脚本若在容器内部, 新建新容器后会全部失效, 不方便复制; 因此可考虑挂载进容器内部

## 方案总结
综上所述, 可以根据需求选择: 
若当做测试服务器, 没有长期运行及迁移要求, 则直接方案一
若当做生产服务器, 有长期运行的服务, 为了方便重创, 销毁, 迁移, 则建议方案三
建议 先在 方案一上跑, 然后看情况是不是刚需, 若是刚需服务, 再折腾迁移至方案三上!
