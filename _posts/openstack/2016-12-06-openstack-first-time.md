---
layout: post
title:  "OpenStack Mitaka从零开始 初识openstack"
date:   2016-12-06 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


## 因为从零开始、所以下面结论有可能是错误的,后面会随着深入而修正

## [对应项目页](http://releases.openstack.org/mitaka/index.html)

centos7安装添加yum源

```shell
yum install centos-release-openstack-mitaka
```

然后安装yum对应rpm包即可

    以python打头的是实际代码
    openstack打头的是配置文件和外部shell工具例如neutron的2个rpm包为
    python-neutron-8.3.0-1.el7.noarch
    openstack-neutron-8.3.0-1.el7.noarch

### 首先理解一下openstack的架构
通过看Service Projects,来认识openstack,我们先从茫茫多的service中,找出主要的service，并理解这些service的作用及关系

#### 打开devstack,一个自动安装openstack的脚本,找到里面的install部分

```shell
# Install Oslo libraries
install_oslo

# Install client libraries
install_keystoneauth
install_keystoneclient
install_glanceclient
install_cinderclient
install_novaclient
if is_service_enabled swift glance horizon; then
install_swiftclient
fi
if is_service_enabled neutron nova horizon; then
install_neutronclient
fi
if is_service_enabled heat horizon; then
install_heatclient
fi

# Install middleware
install_keystonemiddleware

if is_service_enabled keystone; then
if [ "$KEYSTONE_AUTH_HOST" == "$SERVICE_HOST" ]; then
stack_install_service keystone
configure_keystone
fi
fi

if is_service_enabled swift; then
if is_service_enabled ceilometer; then
install_ceilometermiddleware
fi
stack_install_service swift
configure_swift

# swift3 middleware to provide S3 emulation to Swift
if is_service_enabled swift3; then
# Replace the nova-objectstore port by the swift port
S3_SERVICE_PORT=8080
git_clone $SWIFT3_REPO $SWIFT3_DIR $SWIFT3_BRANCH
setup_develop $SWIFT3_DIR
fi
fi

if is_service_enabled g-api n-api; then
# Image catalog service
stack_install_service glance
configure_glance
fi

if is_service_enabled cinder; then
# Block volume service
stack_install_service cinder
configure_cinder
fi

if is_service_enabled neutron; then
# Network service
stack_install_service neutron
install_neutron_third_party
fi

if is_service_enabled nova; then
# Compute service
stack_install_service nova
cleanup_nova
configure_nova
fi

if is_service_enabled horizon; then
# django openstack_auth
install_django_openstack_auth
# dashboard
stack_install_service horizon
fi

if is_service_enabled heat; then
stack_install_service heat
install_heat_other
cleanup_heat
configure_heat
fi

if is_service_enabled tls-proxy || [ "$USE_SSL" == "True" ]; then
configure_CA
init_CA
init_cert
# Add name to ``/etc/hosts``.
# Don't be naive and add to existing line!
fi


# Extras Install
# --------------

# Phase: install
run_phase stack install

# Install the OpenStack client, needed for most setup commands
if use_library_from_git "python-openstackclient"; then
git_clone_by_name "python-openstackclient"
setup_dev_lib "python-openstackclient"
else
pip_install_gr python-openstackclient
fi
```

### 找到里面主要服务

    1. nova
    2. keystone
    3. neutron
    4. horizon
    5. cinder
    6. swift
    7. glance
    8. heat

#### 5、6、7(swift,cinder,glance)[参考](http://cloud.51cto.com/art/201304/387374.htm)
    参考内容摘要
    Swift——提供对象存储 (Object Storage)，在概念上类似于Amazon S3服务，不过swift具有很强的扩展性、冗余和持久性，也兼容S3 API
    Glance——提供虚机镜像(Image)存储和管理，包括了很多与Amazon AMI catalog相似的功能。(Glance的后台数据从最初的实践来看是存放在Swift的)。
    Cinder——提供块存储(Block Storage)，类似于Amazon的EBS块存储服务，目前仅给虚机挂载使用。
    总结
    Swift   云存储服务(Glance和Cinder的的存储可以使用Swift)、网盘之类、cdn目录之类、无对应需求可以不装
    Glance  虚拟机镜像文件备份、模板景象、快照之类,这个必须装,不然没有办法获取给虚拟机装系统的镜像
    Cinder  虚拟机可挂载的外部硬盘文件管理、应该就是提供创建虚拟机创建以后可以挂载的外部硬盘,虚拟机挂载外部硬盘需要用到这个服务

#### 4(horizon[参考](http://blog.sina.com.cn/s/blog_4cccd8d30102vbr5.html)
    参考内容摘要
    在整个openstack应用体系中，horizon就是整个应用的入口。提供了一个模块化的、基于WEB的图形化界面服务门户。用户可以通过浏览器使用这个WEB图形化界面来访问、控制他们的计算、存储和网络资源。
    总结
    horizon相当于阿里、ucloud用户登陆后可以操作的那个页面，基于django开发，代码目前拆分为horizon和openstack_dashboard
    前者为基本库，后者为整个站点文件
    这个基本是必须安装的,不然真个openstack都要用cli命令去控制

#### 8(heat)[参考](http://www.ibm.com/developerworks/cn/cloud/library/1511_zoupx_openstackheat/index.html)
    参考内容摘要
    OpenStack 从最开始就提供了命令行和 Horizon 来供用户管理资源。
    然而靠敲一行行的命令和在浏览器中的点击相当费时费力。即使把命令行保存为脚本，
    在输入输出，相互依赖之间要写额外的脚本来进行维护，而且不易于扩展。
    用户或者直接通过 REST API 编写程序，这里会引入额外的复杂性，同样不易于维护和扩展。
    这都不利于用户使用 Openstack 来进行大批量的管理，更不利于使用 OpenStack 来编排各种资源以支撑 IT 应用。
    总结
    建立一个mysql集群，需要在页面点击创建N个mysql,申请浮动ip什么等等操作
    heat可以实现一个模板、建立好模板以后直接点一下就能创建好mysql集群了
    可以不用安装

#### 2(keystone)[参考](http://blog.csdn.net/wsfdl/article/details/20492343)
    参考内容摘要
    keystone作为openstack的Identity Service，提供了用户信息管理和完成各个模块认证服务。
    总结
    身份验证服务,一个网站如果把身份验证服务写在站点里,那么当网站要做集群的时候,网站之间需要通信才能解决几个站点通信之类的问题
    同样问题也出现在各种工具里,工具只是登陆到mysql然后做完验证就工作，那么同时有多个人使用工具的时候，就会出现问题
    解决方法
    1、在mysql里加锁，通过mysql的锁达到互斥的目的，但是业务层的逻辑明显不应该由mysql去做
    2、类似tomcat多播形式，但是这种形式也只适合复制session之类的业务，在里面加锁也不适合
    3、专门写一个做身份验证的程序
    so keystone就是这个专门做验证的程序,这个验证程序会和各个service通信
    一套openstack里只需要一个keystone service,但是要对这个keystone service做双机热备和lvs负载均衡,因为所有的service基本都需要和keystone通信
    上面的3点是错误的、高估了keystone的能力,keystone只是一个通过http协议提供认证的服务
    keyston不仅是和用于认证相关、openstack各个服务都要和keystone通信获取认证还有catalog


#### 3(neutron)[参考](http://www.server110.com/openstack/201403/6926.html)
    参考内容摘要
    现在很多已部署的Openstack云还在继续使用nova-network，因为它简单，稳定，尤其是多节点部署的可扩展性和可靠性让人不愿割舍。但是我们必须考虑往Neutron上迁移了
    总结
    openstack处理虚拟网络的服务,用于替代nove-network，在M版中依然可以配置nove-network，新版本应该直接使用neutron而不是nove-network
    neutron和nova-api一样是个controller服务,只负责分配,具体设置网络由各种agent去设置
    比如支持openvswitch需要neutron-plugin-openvswitch-agent(rpm包名openstack-neutron-openvswitch)
    每台openstack的物理机器都要安装这个服务以便设置网络
    neutron是虚拟化很重要的一环、网络虚拟化都是由他负责,各种硬件商都参与这个组件开发


#### 1(nova)比较复杂
    我们先看nova的rpm打包文件
    https://github.com/openstack-packages
    去到去对应版本的spec文件
    https://github.com/openstack-packages/nova/blob/mitaka-rdo/openstack-nova.spec
    可以看到nova拆分为好几部分,顺便感慨下...不再提供init脚本,只提供了systemd的文件....
    nova部分结构比较复杂,直接用一个新的章节来说明

追加说明:所有的spec文件去centos[官网](https://cbs.centos.org/)搞定

    官网会自动重定向到https://cbs.centos.org/koji/

通过搜索对应的rpm包,找到src包并下载下来

有些包可能因为名字问题搜索不到,比如python-keystone其实就是openstack-keystone打包出来的,
直接搜不到....蛋痛还拆成2个RPMs

    openstack-keystone-9.0.2-1.el7.src.rpm (info) (download)
    noarch	(build logs)
    openstack-keystone-9.0.2-1.el7.noarch.rpm (info) (download)
    openstack-keystone-doc-9.0.2-1.el7.noarch.rpm (info) (download)
    python-keystone-9.0.2-1.el7.noarch.rpm (info) (download)
    python-keystone-tests-9.0.2-1.el7.noarch.rpm (info) (download)

顺便放下载源
    http://mirror.centos.org/centos/7/cloud/x86_64/openstack-mitaka/

##### 如果需要改成非systemd,工作量还是比较大,找到一个参考包

    openstack-nova-2014.2.2-2.el6.src.rpm

需要对修改有服务的程序spec文件并写init脚本...所以.....还是升级centos7再测试吧
