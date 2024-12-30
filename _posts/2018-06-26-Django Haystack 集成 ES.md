---
layout:     post
title:      Django Haystack 集成 ES
date:       2018-06-26
author:     "FridayLi"
catalog: true
tags:
  - Python
  - Django
---

> 版本: 
> Django:  1.11
> django-haystack: 2.8.1
> elasticsearch: 2.3.5
>其中django版本要求不严格,  elasticsearch 我还尝试了5.4.3和6.3.0, 好像和django-haystack不兼容, 没办法, 又用回了2.3.5, 本文中提到的shield在5.x版本以上已经集成到了x-pack,  而使用x-pack需要额外下载对应的lisence, 当然, 也可以用开源的 [search-guard](https://github.com/floragunncom/search-guard). 

## 安装Java
1. 在orical[官网](http://www.oracle.com/technetwork/java/javase/downloads/jre8-downloads-2133155.html)点同意， 然后复制相应版本的下载连接

            $ cd ~
            $ wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://下载链接"
            $ yum localinstall jre-8u171-linux-x64.rpm

2. 安装完成， 检查一下java版本：

            $ java -version
            java version "1.8.0_171"
            Java(TM) SE Runtime Environment (build 1.8.0_171-b11)
            Java HotSpot(TM) 64-Bit Server VM (build 25.171-b11, mixed mode)

## 安装 ES
1. `rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch`
2. 在目录 /etc/yum.repos.d/  下新建文件 elasticsearch.repo， 并写入以下内容：

            [elasticsearch-2.x]
            name=Elasticsearch repository for 2.x packages
            baseurl=https://artifacts.elastic.co/packages/2.x/yum
            gpgcheck=1
            gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
            enabled=1
            autorefresh=1
            type=rpm-md

3.  离线下载rpm安装包

            $ wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/rpm/elasticsearch/2.3.5/elasticsearch-2.3.5.rpm
            $ sudo rpm --install elasticsearch-2.3.5.rpm

4. 相关目录如下：

            配置文件： /etc/elasticsearch/elasticsearch.yml
            启动文件： /usr/share/elasticsearch/bin/
            插件： /usr/share/elasticsearch/plugins/

## 配置和启动 ES
1. 配置:
vim  /etc/elasticsearch/elasticsearch.yml,  在末尾写入如下内容:

            cluster.name: thor  # 集群名字
            node.name: node-jaim18-37-244  # node名字
            path.data: /data/haystack/data  # data 目录, 没有该目录要提前创建
            path.logs: /data/logs/haystack  # log 目录
            network.host: 0.0.0.0  #　公开连接
            http.port: 9200　　＃　端口号，　默认９２００
            index.analysis.analyzer.default.type: ik  # 5.x 或 6.x 版本不要在这添加index级别的配置

2. 启动:

            $ systemctl daemon-reload
            $ sudo systemctl start elasticsearch.service

启动失败的话可以去path.logs 目录下查看失败日志.

## ik 分词
1. 在[github](https://github.com/medcl/elasticsearch-analysis-ik/releases)下载对应elasticsearch的ik版本（2.3.5对应1.9.5）
2. 解压到 /usr/share/elasticsearch/plugins/ 目录下

            $ wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v1.9.5/elasticsearch-analysis-ik-1.9.5.zip
            $ unzip elasticsearch-analysis-ik-1.9.5.zip -d ik
            $ rm elasticsearch-analysis-ik-1.9.5.zip

3. 创建 index, 设置分词配置

            $ curl -XPUT http://localhost:9200/thor_test

            $ curl -XPOST http://localhost:9200/thor_test/fulltext/_mapping -H 'Content-Type:application/json' -d'
            {
                        "properties": {
                            "content": {
                               "type": "text",
                                "analyzer": "ik_max_word",
                                "search_analyzer": "ik_max_word"
                            }
                        }

                }'

4. 重启生效
`sudo systemctl restart elasticsearch.service`
5. 可以测试下分词结果:

            $ wget http://localhost:9200/_analyze/?analyzer=ik_smart&text=中华人民共和国国歌
            分词结果如下:(可以把ik_smart 改成ik_max_word看看另一种效果)
            "tokens": [
            {
                "token": "中华人民共和国",
                "start_offset": 0,
                "end_offset": 7,
                "type": "CN_WORD",
                "position": 0
            },
            {
                "token": "国歌",
                "start_offset": 7,
                "end_offset": 9,
                "type": "CN_WORD",
                "position": 1
                }
            ]
            }

## Shield (设置账户密码)
把 elasticsearch的data暴露给所有人肯定是不合适的, 接下来, 我们配置一下账户和密码, 需要用到插件 shield.

1. 下载plugin

            $ cd /root/ldy # 下载目录不能是plugin目录
            $ wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/plugin/license/2.3.5/license-2.3.5.zip
            $ wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/plugin/shield/2.4.6/shield-2.3.5.zip

2. 安装

            $ bin/plugin install file:///root/ldy/license-2.3.5.zip
            $ bin/plugin install file:///root/ldy/shield-2.3.5.zip

3. 配置账户密码

            $ bin/shield/esusers useradd es_admin -r admin  # 用户名 es_admin, 角色是admin, 回车后输入密码123456

4. 重启
`sudo systemctl restart elasticsearch.service`

## django 配置
1. 安装和配置haystack(网上教程比较多, 我就不copy了)
2. 配置 elasktic
在settings里如下配置:

            from urllib.parse import urlparse  # python 3

            parsed = urlparse('http://es_admin:123456@127.0.0.1:9200')  # 如果es不在本机, 记得更改IP
            HAYSTACK_CONNECTIONS = {
                    'default': {
                        'ENGINE': 'thor.elastic_backend.CustomElasticSearchEngine',
                        'URL': parsed.hostname,
                        'INDEX_NAME': 'thor_test',
                        'KWARGS': {
                            'port': parsed.port,
                            'http_auth': (parsed.username, parsed.password),
                            'use_ssl': False,
                        }
                    },
               }

3. 建立索引
`python manage.py rebuild_index`,  若建立成功, 则说明大功告成.

