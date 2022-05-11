# Lab1 MapReduce











# 附录：Linux中安装go环境

1、下载安装包

```
 wget -c https://storage.googleapis.com/golang/go1.13.1.linux-amd64.tar.gz
```

2、解压

```
tar -C /usr/local/ -zxvf go1.13.1.linux-amd64.tar.gz 
```

3、添加系统环境变量

```
vim /etc/profile.d/go.sh
export PATH=$PATH:/usr/local/go/bin
source /etc/profile.d/go.sh
```

4、设置GOPATH目录

```
mkdir -p ~/home/user/go
vim /etc/profile.d/gopath.sh
source /etc/profile.d/gopath.sh
```

5、检查是否配置成功

```
echo $GOPATH
go version
```

