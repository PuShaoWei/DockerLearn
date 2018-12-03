### Docker 基础

1. 搭建全新的环境 [163 镜像](http://mirrors.163.com/.help/centos.html)

首先备份`/etc/yum.repos.d/CentOS-Base.repo`

```
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```
下载对应版本repo文件, 放入/etc/yum.repos.d/(操作前请做好相应备份)

运行以下命令生成缓存
```
yum clean all
yum makecache
```
创建一个个人账号
```
useradd -d /home/pushaowei -m pushaowei
passwd pushaowei

chmod u+w /etc/sudoers
echo 'pushaowei       ALL=(ALL)       ALL' >> /etc/sudoers
```

2. 安装阿里云加速源

a. 安装／升级Docker客户端
```
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3: 更新并安装 Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start
```
b.针对Docker客户端版本大于 1.10.0 的用户

您可以通过修改daemon配置文件`/etc/docker/daemon.json`来使用加速器
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://v161jz10.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
c. 登陆该仓库
`docker login --username=pushaowei520@vip.qq.com registry.cn-hangzhou.aliyuncs.com`


3. 获取应用栈各节点所需镜像

```
docker pull ubuntu 
docker pull django
docker pull haproxy
docker pull redis
```

4. 罗列现有镜像
 
```
docker images
```

