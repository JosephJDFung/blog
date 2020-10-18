# MongoDB体系结构

## MongoDB包组件结构

MongoDB数据库服务以及一些[工具](https://docs.mongodb.com/manual/reference/program/#mongodb-package-components)。


- 二进制导入导出
    - `mongodump` 创建mongod数据库内容的二进制导出。

    - `mongorestore` 将数据从mongodump数据库转储还原到mongod或mongos中 

    - `bsondump` 将BSON转储文件转换为JSON。

- 数据导入导出
    - `mongoimport` 从扩展JSON，CSV或TSV导出文件导入内容。

    - `mongoexport` 产生存储在mongod实例中的数据的JSON或CSV导出。

- 诊断工具
    - `mongostat` 快速概述当前正在运行的mongod或mongos实例的状态。
    - `mongotop` 概述mongod实例花费在读写数据上的时间。

- GridFS 工具
    - `mongofiles` 	处理GridFS对象中存储在MongoDB实例中的文件。

- `mongoperf`: mongoDB自带工具，用于评估磁盘随机IO性能。

- mongod（数据库服务）

在单机部署中扮演 数据库服务器（提供所有读写功能） 

在副本集部署中，通过配置，可以部署为 primary节点（主服务器，负责写数据，也可以提供查询）、secondary节点（从服务器，它从主节点复制数据，也可以提供查询）、以及arbiter节点（仲裁节点，不保存数据，主要用于参与选举投票） 

在分片集群中，除了在每个分片中扮演上述角色外，还扮演着配置服务器的角色（存储有分片集群的所有元数据信息，mongos的数据路由分发等都要依赖于它）

## MongoDB内建数据库

- `admin` admin库主要存放有数据库帐号相关信息。 

- `local` local数据库永远不会被复制到从节点，可以用来存储限于本地单台服务器的任意集合副本集的配置信息、oplog就存储在local库中。
重要的数据不要存储在local库，因为没有冗余副本，如果这个节点故障，存储在local库的数据就无法正常使用了。 

- `config` config数据库用于分片集群环境，存放了分片相关的元数据信息。
- `test` MongoDB默认创建的一个测试库，连接mongod服务时，如果不指定连接的具体数据库，默认就会连接到test库。