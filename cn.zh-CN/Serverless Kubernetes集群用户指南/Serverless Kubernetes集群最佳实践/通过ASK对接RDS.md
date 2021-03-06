通过ASK对接RDS 
===============================

本文将介绍如何通过ECI对接RDS。

目前RDS支持五种数据库引擎，您可以根据业务需求来选择。具体请参见[数据库引擎介绍。](https://help.aliyun.com/document_detail/26174.html?spm=a2c4g.11186623.6.585.13b933238aKH0l)

如下使用RDS MySQL实例为例：

前提条件 
-------------------------

* 已创建RDS实例，并且在实例上配置数据库用户帐号。RDS实例创建方法请参见[如何创建RDS实例。](https://help.aliyun.com/document_detail/26124.html?spm=a2c4g.11186623.6.621.1c0f60ff1Q46IO)

  




<!-- -->

* K8S集群中正确部署了virtual-kubelet（Serverless Kubernetes默认已经安装，请参见[创建Serverless Kubernetes集群](https://help.aliyun.com/document_detail/86377.html#task-e3c-311-ydb)。）

  





**注意**

建议RDS实例与K8S集群属于同一VPC网络下，当然也可以跨VPC网络通过公网方式建立连接；本次示例使用同一VPC网络下。

设置白名单 
--------------------------

**说明**



RDS的白名单包括两种类型，IP白名单和安全组，如下：

* IP白名单

  




添加IP地址，允许这些IP地址访问该RDS实例。默认的IP白名单只包含默认IP地址127.0.0.1，表示任何设备均无法访问该RDS实例。

* 安全组

  




安全组是一种虚拟防火墙，用于控制安全组中的ECI实例的出入流量。在RDS白名单中添加安全组后，该安全组中的ECI实例就可以访问RDS实例。



* 内网方式

  




1、在[RDS管理控制台](https://rdsnext.console.aliyun.com/?accounttraceid=dfbef66e3a954ab098b4f9498e785c19fsek#/detail/rm-bp1tlj66mf2lp5826/link/instanceLink?region=cn-hangzhou)中点击 **数据库连接** 获取内网地址。

![host](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/8699305951/p132323.png)

内网地址为数据库的host。

2、设置白名单

添加创建Pod时绑定的vpc实例或者vsw实例的网段，也可以直接添加安全组实例。

![sg、vsw](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/8699305951/p132325.png)

* 公网方式

  




1、申请外网地址

公网也叫外网，通过公网访问RDS就是使用RDS实例的外网地址进行访问。RDS实例默认不提供外网地址，如果要通过公网访问，请在RDS控制台 \> 数据库连接页面申请外网地址。

![1](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/8699305951/p131788.png)
**说明**

* 外网地址会降低实例的安全性，请谨慎使用。

  

* 为了获得更快的传输速率和更高的安全性，建议您将应用迁移到与您的RDS实例在同一地域且网络类型相同的ECI实例，然后使用内网地址。

  




<!-- -->

* 若需要公网访问详细请参见[如何连接RDS数据库。](https://help.aliyun.com/knowledge_detail/41843.html#concept-41843-zh)

  




有了外网地址之后，就可以使用外网地址连接到RDS实例。

2、设置白名单

* ECI实例所属vsw绑定到NAT网关，获取它的公网IP地址添加到RDS白名单中。

  

* 添加ECI绑定EIP实例的弹性公网地址。

  



**说明**

当RDS开通外网地址访问，ECI实例若连接RDS需要具备[外网访问](https://help.aliyun.com/document_detail/99146.html?spm=a2c4g.11174283.6.584.7f9f4b5brwjsvb)能力。

* 方式一：VPC绑定NAT网关+EIP；把ECI所绑定的VSW实例公网IP地址添加到白名单中（建议使用）。

  

* 方式二：实例直接绑定EIP；直接添加ECI绑定的弹性公网IP地址到白名单中。

  





**注意**

白名单可以让RDS实例得到高级别的访问安全保护，建议您定期维护白名单。设置白名单不会影响RDS实例的正常运行。具体请参见[如何设置白名单](https://help.aliyun.com/document_detail/43185.html?spm=5176.13643035.0.0.1c2b14500OaU7l)。



Kubernetes方式 
---------------------------------

以下使用内网方式连接RDS数据库：

创建Pod所使用的vsw实例网段添加到RDS白名单中。

![q](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/9699305951/p132335.png)![a](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/9699305951/p132336.png)



1、 **创建ConfigMap** 

ConfigMap将您的环境配置信息和容器镜像解耦，便于应用配置的修改，在[容器服务管理控制台](https://cs.console.aliyun.com/?spm=5176.12818093.0.dcsk.77c516d0ufbVnC&accounttraceid=db0633bd94804c87b88561eaec1fd8d9eqcw#/k8s/configmap/list)中点击 **应用配置 \> 配置项** 创建。

![1](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/9699305951/p132273.png)

连接云数据库需要的信息写入configMap中。

![2](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/9699305951/p132280.png)
**注意**

host获取RDS数据库中的内网地址。

![2](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/9699305951/p132337.png)

2、 **创建Secret** 

当您需要储存机密信息时可以使用Secret将配置详细信息传递给应用的安全方式，例如您的数据库名称、用户和密码等详细信息，您可以将其作为环境变量注入到应用中。

![secret](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/9699305951/p132281.png)

添加RDS数据库用户和密码。

![secret1](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/9699305951/p132282.png)

3、创建Pod对接RDS

在[容器服务管理控制台](https://cs.console.aliyun.com/?spm=5176.12818093.0.dcsk.77c516d0ufbVnC&accounttraceid=db0633bd94804c87b88561eaec1fd8d9eqcw#/k8s/deployment/list)点击 **应用 \> 无状态** 使用模板创建Pod。
**说明**

由于RDS与原生的数据库服务完全兼容，所以您可以使用任何通用的数据库客户端连接到RDS实例，且连接方法类似。如下示例使用python连接数据库。

    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        name: rds-test
      name: rds-test
    spec:
      containers:
      - name: test-rds
        image: registry.cn-hangzhou.aliyuncs.com/eci_open/sqlclient:latest # 一个python job用来连接云数据库的镜像
    imagePullPolicy: IfNotPresent
        command: [ "/bin/bash", "-c", "python3 /testapp/mysqlclient.py"]
        env:
        - name: MYSQL_HOST # 请注意这里和 ConfigMap 中的键名是不一样的
          valueFrom:
            configMapKeyRef:
              name: sqlconfig  # 这个值来自 ConfigMap
              key: host  # 需要取值的键
        - name: MYSQL_PORT
          valueFrom:
            configMapKeyRef:
              name: sqlconfig
              key: port
        - name: MYSQL_DB
          valueFrom:
            configMapKeyRef:
              name: sqlconfig
              key: database
        - name: MYSQL_USERNAME
          valueFrom:
              secretKeyRef:
                name: sqlsecret
                key: username
        - name: MYSQL_PWD
          valueFrom:
            secretKeyRef:
              name: sqlsecret
              key: password
      restartPolicy: Never





本示例镜像使用一个python脚本连接云数据库并插入两条数据。如下：

    import pymysql
    import os
    
    config = {
        'host': str(os.getenv('MYSQL_HOST')),
        'port': int(os.getenv('MYSQL_PORT')),
        'user': str(os.getenv('MYSQL_USERNAME')),
        'password': str(os.getenv('MYSQL_PWD')),
        'database': str(os.getenv('MYSQL_DB')),
    }
    
    
    def mysqlClient():
        db = pymysql.connect(**config)
        try:
            cursor = db.cursor()
            cursor.execute("INSERT INTO username(user) VALUES('Mrs')")
            cursor.execute("INSERT INTO username(user) VALUES('xiao')")
            cursor.execute("SELECT*FROM username")
            result = cursor.fetchall()
        except Exception as e:
            print('System Error: ',e)
        finally:
            db.commit()
            cursor.close()
            db.close()
        return result
    
    
    if__name__ == '__main__':
        mysqlClient()




**说明**

连接RDS数据库的方式有公网访问和内网访问两种，以上所用方式为内网访问，建议使用内网访问的方式保证传输速率和安全性。

4、成功对接RDS数据库效果

![dms-db](//static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/9699305951/p132492.png)



常见问题 
-------------------------

**注意**

数据库无法连接的排查方法请根据当前环境的实际情况，选择对应的排查方法。

1、容器组通过内网无法访问RDS实例

2、外网无法访问RDS实例

详细信息请参见[常见问题解析](https://help.aliyun.com/knowledge_detail/96028.html)。



数据库无法连接的常见场景

* [网络类型不同](https://help.aliyun.com/knowledge_detail/96028.html#title-gty-1nq-r8g)

  

* [专有网络不同](https://help.aliyun.com/knowledge_detail/96028.html#title-hvd-0r0-yiv)

  

* [域名解析失败或错误](https://help.aliyun.com/knowledge_detail/96028.html#title-6oq-qzd-4fa)

  

* [地域不同](https://help.aliyun.com/knowledge_detail/96028.html#title-0t4-i4f-1iv)

  

* [IP白名单设置有误](https://help.aliyun.com/knowledge_detail/96028.html#title-xgw-uw3-8jm)

  

* [只读实例未设置白名单](https://help.aliyun.com/knowledge_detail/96028.html#title-8xc-gt0-2l2)

  

* [内外网地址使用错误](https://help.aliyun.com/knowledge_detail/96028.html#title-k7f-geh-qri)

  

* [连接数已满](https://help.aliyun.com/knowledge_detail/96028.html#title-owv-w9q-xa3)

  

* [用户名或密码错误](https://help.aliyun.com/knowledge_detail/96028.html#JeQbH)

  

* [无法解析地址](https://help.aliyun.com/knowledge_detail/96028.html#CbSPT)

  










