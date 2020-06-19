## hadoop集群搭建
### 单机hadoop搭建
#### 安装java
![图片](./img/1.png)
![图片](./img/2.png)

#### 禁用ipv6
![图片](./img/3.png)

#### 解压hadoop包

![图片](./img/4.png)

#### 构建DataNode，NameNode
![图片](./img/5.png)

#### 更新环境变量
![图片](./img/6.png)

![图片](./img/7.png)

![图片](./img/8.png)

![图片](./img/9.png)
#### 更新配置文件
#### etc/hadoop/core-site.xml：
![图片](./img/10.png)
#### etc/hadoop/hdfs-site.xml：
![图片](./img/11.png)
#### etc/hadoop/yarn-site.xml
![图片](./img/12.png)
![图片](./img/13.png)
![图片](./img/14.png)
#### 格式化namenode，启动hadoop
![图片](./img/15.png)
![图片](./img/16.png)
![图片](./img/17.png)

### 集群搭建
![图片](./img/18.png)
#### 免密登录ssh
![图片](./img/19.png)

![图片](./img/20.png)
#### salver2,3
![图片](./img/21.png)
#### 更新配置文件
#### core-site.xml
![图片](./img/22.png)
#### yarn-site.xml
![图片](./img/23.png)
#### /etc/hadoop/slaves:
![图片](./img/24.png)

![图片](./img/25.png)
#### 初始化hdfs namenode -format
- 如果直接格式clusterID将不一致,需要删除datanode,namenode,logs目录
![图片](./img/26.png)

#### 启动集群
#### start-dfs.sh
![图片](./img/27.png)
#### start-yarn.sh
![图片](./img/28.png)
#### Start Job History Server
![图片](./img/29.png)
####  Jps 查看进程:
![图片](./img/30.png)
#### slavers：
![图片](./img/31.png)



