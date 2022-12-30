# 自建图床服务器picx
## 背景
gitee 图床国内虽然快, 但是偶尔302 无法访问 , 所以准备回迁至github中 (还是大厂稳, 万恶的资本主义) , 虽然迁移后即使用了cdn加速, 仍然有时无法访问, 但是好在图片都保存在仓库中, 换几个hosts或自己爬梯子应该还是很稳的.

## 目标
- 迁移gitee上图床到github, 弃用gitee
- 无需下载客户端(picgo) 就能上传图片(私有化部署)
- 安全,简单, 最好web部署

## 业界方案
参考文档: https://blog.csdn.net/wbsu2004/article/details/121154470  
PicX 官方网站：https://picx.xpoet.cn/
### 构建镜像
官方虽然有很详细的使用方法，但没有提供安装方法，只有一个网站，所以想要私有化部署，需要我们自己打包镜像, 如果你不想自己构建，可以跳过，直接阅读下一章节  
Dockerfile 是按照 Vue官方提供的例子实现的，据说这是目前性能最好、久经考验的解决方案之一，采用了多阶段构建的方法，最终的镜像不到 30M  
```python
# build stage
FROM node:lts-alpine as build-stage
LABEL maintainer=laosu<wbsu2003@gmail.com>

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# production stage
FROM nginx:stable-alpine as production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
构建镜像和容器运行的基本命令如下:  
```python
# 下载代码,当前对应的是 1.4.1
git clone https://github.com/XPoet/picx.git
# 或者镜像站点
git clone https://hub.fastgit.org/XPoet/picx.git
# 进入目录
cd picx
# 将 Dockerfile 放进代码目录中
# 构建镜像
docker build -t wbsu2003/picx:v1 .
```
### 运行镜像
```python
#!/bin/bash
docker run -d --net=bridge-to-router --restart=always  \
--name=picx --ip=192.168.0.205 --memory="256000000"  \
wbsu2003/picx:latest

```
然后访问 [http://192.168.0.205](http://192.168.0.205)
> 注意, 用别的端口号console会有跨域报错, 貌似只能用80端口访问才能正常
> 其实问题不大,实在不行, nginx再代理一层


## 个人方案
参考业界方案, 目前是 picgo和 picx都可以用;  
发布文章用docsify;  
文章中图床用picx/picgo 上传至github的小号.  
> 还踩了个坑, 个人导航网站用gitee_page发布,
> 访问mxcall.gitee.io/homepage/可以 , mxcall.gitee.io/homepage就不通(少个斜杠),
> 目测是gitee_page的nginx配置问题

### 访问报错参考修改chrome配置
https://www.cnblogs.com/whm-blog/p/16418314.html
配置chrome选项为disable 
chrome://flags/#block-insecure-private-network-requests
### cdn访问图挂了
尝试用https://cd(去掉此处文字和括号)n.jsdelivr.net/ 换为 https://fast(去掉此处文字和括号)ly.jsdelivr.net/
或者其他
- f(去掉此处文字和括号)astly.jsdelivr.net
- t(去掉此处文字和括号)estingcf.jsdelivr.net  (test ok)
- g(去掉此处文字和括号)core.jsdelivr.net (test ok)
- q(去掉此处文字和括号)uantil.jsdelivr.net
- originfas(去掉此处文字和括号)tly.jsdelivr.net
