.. _openvpn-to-rds:

##############################
通过 OpenVPN 连接到 RDS
##############################

.. contents::

*******************************************
安装 OpenVPN 
*******************************************

先安装 epel
=========================

- 如果 OS 是 Amazon Linux 2，运行 

.. code:: bash

    sudo amazon-linux-extras install epel

- CentOS 的话，运行 
  
.. code:: bash

  sudo yum install epel-release

正式安装 OpenVPN
======================

.. code:: bash

    sudo yum install openvpn` 


生成证书
----------------------

在本地计算机下载 easyrsa

.. code:: bash

    git clone https://github.com/OpenVPN/easy-rsa
    cd easy-rsa/easyrsa3`

初始化目录

.. code:: bash

    ./easyrsa init-pki

生成根证书

.. code:: bash

    ./easyrsa build-ca

会提示输入密码保护私钥：

.. code:: bash
    
    > key passphrase:

再输入一个唯一标识：

.. code:: bash

    Common Name: 

生成证书请求，先生成服务器端，再生成客户端

.. code:: bash

    ./easyrsa gen-req vpn-server
    ./easyrsa gen-req vpn-client`

同样可能会提示输入密码和唯一标识

签发证书

.. code:: bash
    
    ./easyrsa sign-req server vpn-server
    ./easyrsa sign-req client vpn-client

生成 Diffie–Hellman key 

.. code:: bash

    ./easyrsa gen-dh

*******************************************
配置 OpenVPN
*******************************************

复制以下文件到 EC2 上的 **/etc/openvpn/server** 目录: 

.. code:: 

    easyrsa/easyrsa3/pki/ca.crt 
    easyrsa/easyrsa3/pki/dh.pem 
    easyrsa/easyrsa3/pki/issued/vpn-server.crt
    easyrsa2/easyrsa3/pki/private/vpn-server.key

复制配置模版
=========================

.. code:: bash

    cd /etc/openvpn/server
    cp /usr/share/doc/openvpn-2.4.11/sample/sample-config-files/server.conf .

修改配置文件
=========================

去掉 ``topology subnet`` 前的注释分号

将 **ca、cert、key、dh** 改为正确的文件名

把 **server** 改为不会和线下和 VPC 冲突的网段

增加 ``push "route vpc网段"``，让客户端可以访问除 EC2 外的云端资源

生成 ta.key
=========================

.. code:: bash

    openvpn --genkey --secret ta.key

*******************************************
启动 OpenVPN
*******************************************

.. code:: bash

    systemctl start openvpn.server@server


如果设置了 PEM pass phase，需要运行 ``systemd-tty-ask-password-agent`` 输入密码

关闭EC2实例的 `Source/Destination check <https://docs.aws.amazon.com/vpc/latest/userguide/VPC_NAT_Instance.html#EIP_Disable_SrcDestCheck>`__

打开EC2实例的路由转发
=========================

.. code:: bash

    echo 1 > /proc/sys/net/ipv4/ip_forward

*******************************************
配置客户端
*******************************************

同样需复制 ``ca.crt, vpn-client.key, vpn-client.crt`` 到 **/etc/openvpn** 目录，还需要从 EC2 复制 **ta.key** 文件

复制配置文件模版

.. code:: bash

    sudo su -
    cd /etc/openvpn/client
    cp /usr/share/doc/openvpn-2.4.11/sample/sample-config-files/client.conf .

修改 **client.conf**

``remote`` 填入 EC2 公网地址

修改 **ca、cert、key**，指向正确的文件

*******************************************
修改云端网络配置
*******************************************

EC2安全组
=================

添加入站规则，允许 vpn-client 公网地址到 **1194** 端口

RDS安全组
================

添加入站规则，允许 VPN 网段到 3306 端口

在 EC2 通过 ping 包得到 RDS 内网地址

添加路由
===============

修改 RDS 所以子网的路由表，添加到 VPN 网段的路由，下一跳选择安装了 OpenVPN 的 EC2 实例

客户端运行 OpenVPN，然后通过上面得到的 RDS 内网地址访问
