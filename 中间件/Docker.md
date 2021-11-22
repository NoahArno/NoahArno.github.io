# 第一章 各种服务的docker安装

## 1.1 MySQL

docker run -d --restart=always --name mysql -v /mydata/mysql/data:/var/lib/mysql -v /mydata/mysql/conf:/etc/mysql -v /mydata/mysql/log:/var/log/mysql -p 3306:3306 -e TZ=Asia/Shanghai -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7 --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci

## 1.2 nginx

1、先随便安装一个nginx

docker run -p 80:80 --name nginx -d nginx:1.10

2、将nginx中的文件复制下来

mkdir nginx

docker container cp nginx:/etc/nginx .（。的后面有空格）

这个要在mydata文件夹里面执行

3、在mydata下执行：

mv nginx conf

mkdir nginx

mv conf nginx

4、将之前的nginx删除

docker run -p 80:80 --name nginx \

\> -v /mydata/nginx/html:/usr/share/nginx/html \

\> -v /mydata/nginx/logs:/var/log/nginx \

\> -v /mydata/nginx/conf:/etc/nginx \

\> -d nginx:1.10

##  1.3 Redis

1、mkdir -p /mydata/redis/conf

2、touch /mydata/redis/conf/redis.conf

3、docker run -p 6379:6379 --name redis -v /mydata/redis/data:/data -v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf -d redis redis-server /etc/redis/redis.conf

## 1.4 Rabbitmq

docker run -d --name rabbitmq -p 5671:5671 -p 5672:5672 -p 4369:4369 -p 25672:25672 -p 15671:15671 -p 15672:15672 rabbitmq:management

## 1.5 elastic

1、docker pull elasticsearch:7.4.2

2、docker pull kibana:7.4.2（可视化操作）

3、

 mkdir -p /mydata/elasticsearch/config

mkdir -p /mydata/elasticsearch/data

echo "http.host: 0.0.0.0">>/mydata/elasticsearch/config/elasticsearch.yml

chmod -R 777 /mydata/elasticsearch/

docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \

-e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms64m -Xmx512m" \

-v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \

-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \

-v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \

-d elasticsearch:7.4.2

4、

mkdir -p /mydata/kibana/config/
 vi /mydata/kibana/config/kibana.yml

\#

\# ** THIS IS AN AUTO-GENERATED FILE **

\#

\# Default Kibana configuration for docker target

server.name: kibana

server.host: "0"

elasticsearch.hosts: [ "http://192.168.31.190:9200" ]

xpack.monitoring.ui.container.elasticsearch.enabled: true

 

docker run -d \

 --name=kibana \

 --restart=always \

 -p 5601:5601 \

 -v /mydata/kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml \

 kibana:7.4.2

5、安装ik分词器

github.com/medcl/elasticsearch-analysis-ik下载对应的ik分词器

将其解压到挂载的/mydata/elastiscearch/plugins/ik/目录下，



但是上述的分词器功能不够，还需要我们**自定义扩展词库**，因此以下步骤：

1、进入ik下的conf目录下

2、vi IKAnalyzer.cfg.xml 将远程连接打开

![image-20211018204950958](IMG/Untitled.assets/image-20211018204950958.png)

3、这里用的是nginx进行转发，将fenci.txt写在nginx的html文件夹下的es目录下

# 1.6 Zipkin

docker run -d -p 9411:9411 openzipkin/zipkin