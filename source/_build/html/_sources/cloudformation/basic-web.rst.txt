.. _basic-web:

创建一个基础三层架构
====================

.. contents::


生成 Key Pair
-----------------

先按照文档指导， `生成或导入 EC2 密钥对 <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair>`__。

部署模板
---------------

从 `这个位置 <https://www.kiking.team/cdk-web.template.json>`__ 下载 CloudFormation 模板。
打开 `宁夏区域 CloudFormation 控制台 <https://cn-northwest-1.console.amazonaws.cn/cloudformation>`__， 创建一个 Stack。


选择模板文件
>>>>>>>>>>>>>>

选择 **Upload a template file**，选择刚下载的模板文件。


指定 Stack 名称
>>>>>>>>>>>>>>>>>>>>

给 Stack 指点一个方便记忆的名称。
在 KeyPair 下拉列表中，选择刚创建密钥对。
其它不变。
连续点击下一步。

创建 Stack
>>>>>>>>>>>>>

在 **Review** 页面，勾选 ``I acknowledge that Amazon CloudFormation might create IAM resources.`` 点击 **Create Stack**

模板会创造以下资源

- 一个带有3个公有子网、3个私有子网和3个隔离子网的 VPC，CIDR 为 172.31.0.0/16
- 在3个公有子网分别创建一个 NAT Gateway
- 一台安装 Windows 2019 的 EC2 实例
- 一台安装 Amazon Linux 2 的 Linux 实例作为跳板机
- 一个应用负载均衡器，侦听 80 端口，转发请求到 Windows 实例
- 一个 Aurora 集群，包括一个主实例和一个只读实例

连接到 Windows 实例
>>>>>>>>>>>>>>>>>>>>>>

Windows 下以 PuTTY 为例，添加一个到 Linux 实例的 Session，包括地址、用户名、私钥，然后在 ``Connection/SSH/Tunnels`` 中添加一个 Tunnel：

- Source port: 50001
- Destination: Windows实例内网地址:3389


Linux 和 Mac 下可以执行 

.. code:: bash

    ssh -N -L 50001:Windows实例内网地址:3389 ec2-user@Linux实例公网地址

打开远程桌面客户端，连接 127.0.0.1:50001，输入用户名 Administrator，密码获取方式参考 `文档 <https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/EC2_GetStarted.html#ec2-connect-to-instance-windows>`__

测试
>>>>>>>>>>>

- 在 Windows 实例上开启一个 Web 服务，侦听 80 端口，比如 IIS，回到 EC2 控制台，找到 **Load Balancers**，选择模板创建的 Load Balancer，在 Description 中找到 DNS name，输入到浏览器地址栏
- 在 Windows 实例上安装 MySQL Workbench，在 `Secret Manager <https://cn-northwest-1.console.amazonaws.cn/secretsmanager/home?region=cn-northwest-1#!/listSecrets>`__ 找到 RDS 连接信息，连接到 RDS

