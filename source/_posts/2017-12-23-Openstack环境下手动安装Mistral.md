---
title: Openstack环境下手动安装Mistral
date: 2017-12-23 23:12:22
categories:
  - "Openstack"
tags:
  - "Openstack"
  - "Mistral"
  - "Tacker"
---

# 概述
在openstack平台中能够成功安装Tacker，但是安装的Tacker并不能用，因为在tacker中创建VIM时需要调用Mistral工作流组件。因此本文就来介绍在openstack环境中手动安装Mistral的过程。

<!--more-->

**注：** 安装的openstack是在Ubuntu 16.04系统下的Ocata版本；本文中涉及的密码都统一设置成 *“openstack”*。

参考[*官方文档*](https://docs.openstack.org/mistral/latest/install/installation_guide.html)

# 一、安装必要组件
```
$ apt-get install python-dev python-setuptools python-pip libffi-dev \
  libxslt1-dev libxml2-dev libyaml-dev libssl-dev
```
# 二、安装Mistral server
## 1、下载Mistral源码，并进入下载目录
    $ git clone https://github.com/openstack/mistral.git
    $ cd mistral

## 2、安装Mistral环境依赖包
    $ pip install -r requirements.txt
## 3、安装Mistral
    $ python setup.py install

## 4、生成配置文件
    $ oslo-config-generator --config-file tools/config/config-generator.mistral.conf --output-file etc/mistral.conf

## 5、创建Mistral日志文件和配置文件夹
    # mkdir -p /etc/mistral /var/log/mistral

## 6、复制配置文件到配置文件夹
    # cp etc/* /etc/mistral/

## 7、修改配置文件
```
# vi /etc/mistral/mistral.conf 

[keystone_authtoken]
auth_uri = http://controller:5000
auth_version = 3
identity_uri = http://controller:35357/
admin_user = admin
admin_password = openstack
admin_tenant_name = admin

[database]
connection = mysql+pymysql://mistral:openstack@controller/mistral 

[DEFAULT]
transport_url = rabbit://openstack:openstack@controller
```

## 8、创建数据库
```
# mysql
MariaDB [(none)]> CREATE DATABASE mistral;
MariaDB [mistral]> GRANT ALL PRIVILEGES ON mistral.* TO 'mistral'@'localhost' IDENTIFIED BY 'openstack';
MariaDB [mistral]> GRANT ALL PRIVILEGES ON mistral.* TO 'mistral'@'%' IDENTIFIED BY 'openstack';
MariaDB [mistral]> flush privileges;
MariaDB [mistral]> exit;
```

## 9、创建服务和endpoint
```
$ openstack service create --name mistral --description "Openstack Workflow service" workflow
$ openstack endpoint create --region RegionOne workflow public http://controller:8989/v2
$ openstack endpoint create --region RegionOne workflow internal http://controller:8989/v2
$ openstack endpoint create --region RegionOne workflow admin http://controller:8989/v2
```

## 10、初始化数据库信息
```
root@controller:/home/openstack# mistral-db-manage --config-file /etc/mistral/mistral.conf upgrade head

INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 001, Kilo release
INFO  [alembic.runtime.migration] Running upgrade 001 -> 002, Kilo
INFO  [alembic.runtime.migration] Running upgrade 002 -> 003, cron_trigger_constraints
INFO  [alembic.runtime.migration] Running upgrade 003 -> 004, add description for execution
INFO  [alembic.runtime.migration] Running upgrade 004 -> 005, Increase executions_v2 column size from JsonDictType to JsonLongDictType
INFO  [alembic.runtime.migration] Running upgrade 005 -> 006, add a Boolean column 'processed' to the table  delayed_calls_v2
INFO  [alembic.runtime.migration] Running upgrade 006 -> 007, Move system flag to base definition
INFO  [alembic.runtime.migration] Running upgrade 007 -> 008, Increase size of state_info column from String to Text
INFO  [alembic.runtime.migration] Running upgrade 008 -> 009, Add database indices
INFO  [alembic.runtime.migration] Running upgrade 009 -> 010, add_resource_members_v2_table
INFO  [alembic.runtime.migration] Running upgrade 010 -> 011, add workflow id for execution
INFO  [alembic.runtime.migration] Running upgrade 011 -> 012, add event triggers table
INFO  [alembic.runtime.migration] Running upgrade 012 -> 013, split_execution_table_increase_names
INFO  [alembic.runtime.migration] Running upgrade 013 -> 014, fix_past_scripts_discrepancies
INFO  [alembic.runtime.migration] Running upgrade 014 -> 015, add_unique_keys_for_non_locking_model
INFO  [alembic.runtime.migration] Running upgrade 015 -> 016, Increase size of task_executions_v2.unique_key
INFO  [alembic.runtime.migration] Running upgrade 016 -> 017, Add named lock table
INFO  [alembic.runtime.migration] Running upgrade 017 -> 018, increate_task_execution_unique_key_size
INFO  [alembic.runtime.migration] Running upgrade 018 -> 019, Change scheduler schema.
INFO  [alembic.runtime.migration] Running upgrade 019 -> 020, add type to task execution
INFO  [alembic.runtime.migration] Running upgrade 020 -> 021, Increase environments_v2 column size from JsonDictType to JsonLongDictType
```

## 11、添加自带的action
```
root@controller:/home/openstack# mistral-db-manage --config-file /etc/mistral/mistral.conf populate
*输出结果可能为：*
No handlers could be found for logger "mistral.actions.openstack.action_generator.base"
*也可能会出错：*
……
2017-12-22 22:21:24.486 16802 INFO mistral.actions.openstack.action_generator.base [-] Processing OpenStack action mapping from file: /usr/local/lib/python2.7/dist-packages/mistral/actions/openstack/mapping.json
2017-12-22 22:21:24.551 16802 INFO mistral.actions.openstack.action_generator.base [-] Processing OpenStack action mapping from file: /usr/local/lib/python2.7/dist-packages/mistral/actions/openstack/mapping.json
2017-12-22 22:21:24.761 16802 INFO mistral.actions.openstack.action_generator.base [-] Processing OpenStack action mapping from file: /usr/local/lib/python2.7/dist-packages/mistral/actions/openstack/mapping.json
2017-12-22 22:21:24.878 16802 INFO mistral.actions.openstack.action_generator.base [-] Processing OpenStack action mapping from file: /usr/local/lib/python2.7/dist-packages/mistral/actions/openstack/mapping.json
2017-12-22 22:21:24.883 16802 WARNING oslo_config.cfg [-] Option "auth_uri" from group "keystone_authtoken" is deprecated for removal (The auth_uri option is deprecated in favor of www_authenticate_uri and will be removed in the S  release.).  Its value may be silently ignored in the future.
^C2017-12-22 22:21:25.382 16802 CRITICAL Mistral [-] Unhandled error: KeyboardInterrupt
2017-12-22 22:21:25.382 16802 ERROR Mistral Traceback (most recent call last):
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/bin/mistral-db-manage", line 10, in <module>
2017-12-22 22:21:25.382 16802 ERROR Mistral     sys.exit(main())
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/mistral/db/sqlalchemy/migration/cli.py", line 137, in main
2017-12-22 22:21:25.382 16802 ERROR Mistral     CONF.command.func(config, CONF.command.name)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/mistral/db/sqlalchemy/migration/cli.py", line 75, in do_populate
2017-12-22 22:21:25.382 16802 ERROR Mistral     action_manager.sync_db()
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/mistral/services/action_manager.py", line 80, in sync_db
2017-12-22 22:21:25.382 16802 ERROR Mistral     register_action_classes()
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/mistral/services/action_manager.py", line 126, in register_action_classes
2017-12-22 22:21:25.382 16802 ERROR Mistral     _register_dynamic_action_classes()
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/mistral/services/action_manager.py", line 86, in _register_dynamic_action_classes
2017-12-22 22:21:25.382 16802 ERROR Mistral     actions = generator.create_actions()
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/mistral/actions/openstack/action_generator/base.py", line 143, in create_actions
2017-12-22 22:21:25.382 16802 ERROR Mistral     client_method = class_.get_fake_client_method()
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/mistral/actions/openstack/base.py", line 75, in get_fake_client_method
2017-12-22 22:21:25.382 16802 ERROR Mistral     return cls._get_client_method(cls._get_fake_client())
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/mistral/actions/openstack/actions.py", line 380, in _get_fake_client
2017-12-22 22:21:25.382 16802 ERROR Mistral     return cls._get_client_class()(session=sess)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/ironic_inspector_client/v1.py", line 88, in __init__
2017-12-22 22:21:25.382 16802 ERROR Mistral     super(ClientV1, self).__init__(**kwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/ironic_inspector_client/common/http.py", line 134, in __init__
2017-12-22 22:21:25.382 16802 ERROR Mistral     region_name=region_name)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/dist-packages/keystoneauth1/session.py", line 856, in get_endpoint
2017-12-22 22:21:25.382 16802 ERROR Mistral     return auth.get_endpoint(self, **kwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/dist-packages/keystoneauth1/identity/base.py", line 212, in get_endpoint
2017-12-22 22:21:25.382 16802 ERROR Mistral     service_catalog = self.get_access(session).service_catalog
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/dist-packages/keystoneauth1/identity/base.py", line 136, in get_access
2017-12-22 22:21:25.382 16802 ERROR Mistral     self.auth_ref = self.get_auth_ref(session)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/dist-packages/keystoneauth1/identity/generic/base.py", line 198, in get_auth_ref
2017-12-22 22:21:25.382 16802 ERROR Mistral     return self._plugin.get_auth_ref(session, **kwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/dist-packages/keystoneauth1/identity/v3/base.py", line 167, in get_auth_ref
2017-12-22 22:21:25.382 16802 ERROR Mistral     authenticated=False, log=False, **rkwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/dist-packages/keystoneauth1/session.py", line 766, in post
2017-12-22 22:21:25.382 16802 ERROR Mistral     return self.request(url, 'POST', **kwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/dist-packages/positional/__init__.py", line 101, in inner
2017-12-22 22:21:25.382 16802 ERROR Mistral     return wrapped(*args, **kwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/dist-packages/keystoneauth1/session.py", line 616, in request
2017-12-22 22:21:25.382 16802 ERROR Mistral     resp = send(**kwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/dist-packages/keystoneauth1/session.py", line 674, in _send_request
2017-12-22 22:21:25.382 16802 ERROR Mistral     resp = self.session.request(method, url, **kwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/requests/sessions.py", line 508, in request
2017-12-22 22:21:25.382 16802 ERROR Mistral     resp = self.send(prep, **send_kwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/requests/sessions.py", line 618, in send
2017-12-22 22:21:25.382 16802 ERROR Mistral     r = adapter.send(request, **kwargs)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/requests/adapters.py", line 440, in send
2017-12-22 22:21:25.382 16802 ERROR Mistral     timeout=timeout
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/urllib3/connectionpool.py", line 601, in urlopen
2017-12-22 22:21:25.382 16802 ERROR Mistral     chunked=chunked)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/local/lib/python2.7/dist-packages/urllib3/connectionpool.py", line 380, in _make_request
2017-12-22 22:21:25.382 16802 ERROR Mistral     httplib_response = conn.getresponse(buffering=True)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/httplib.py", line 1136, in getresponse
2017-12-22 22:21:25.382 16802 ERROR Mistral     response.begin()
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/httplib.py", line 453, in begin
2017-12-22 22:21:25.382 16802 ERROR Mistral     version, status, reason = self._read_status()
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/httplib.py", line 409, in _read_status
2017-12-22 22:21:25.382 16802 ERROR Mistral     line = self.fp.readline(_MAXLINE + 1)
2017-12-22 22:21:25.382 16802 ERROR Mistral   File "/usr/lib/python2.7/socket.py", line 480, in readline
2017-12-22 22:21:25.382 16802 ERROR Mistral     data = self._sock.recv(self._rbufsize)
2017-12-22 22:21:25.382 16802 ERROR Mistral KeyboardInterrupt
2017-12-22 22:21:25.382 16802 ERROR Mistral
```
纠结了很久后发现这些都不用太在意，直接跳过，哈哈！

# 三、安装Mistral client
## 1、下载Mistral-client源码
    $ git clone git://git.openstack.org/openstack/python-mistralclient.git -b stable/ocata
    $ cd python-mistralclient

## 2、安装Mistral-client模块
    $ pip install -r requirements.txt
    $ python setup.py install



# 四、安装Mistral horizon
## 1、下载Mistral-horizon源码
    $ git clone https://git.openstack.org/openstack/mistral-dashboard.git -b stable/ocata
    $ cd mistral-dashboard/

## 2、安装Mistral-horizon模块
    $ pip install -r requirements.txt
    $ python setup.py install

## 3、复制一个文件
    # cp -b mistraldashboard/enabled/_50_mistral.py /usr/share/openstack-dashboard/openstack_dashboard/enabled/_50_mistral.py

## 4、重启apache2服务
    # service apache2 restart
安装好Mistral-horizon后，admin用户登录dashboard界面就可以看到Mistral相关的workflow，如图：
![Mistral-horizon界面](https://raw.githubusercontent.com/cao0507/Pictures/master/blog/mistral%20horizon.png)


# 五、运行Mistral server
运行下面的第一条命令：
```
root@controller:/home/openstack/mistral# python mistral/cmd/launch.py --server all --config-file /etc/mistral/mistral.conf

|\\    //|           ||                       ||
||\\  //||      __   ||      __      __       ||
|| \\// || ||  //  ||||||  ||  \\  //  \\     ||
||  \/  ||     \\    ||    ||     ||    \\    ||
||      || ||   \\   ||    ||     ||    /\\   ||
||      || || __//   ||_// ||      \\__// \\_ ||
Mistral Workflow Service, version 6.0.0

Launching server components [engine,event-engine,api,executor]...
2017-12-22 22:42:58.373 16966 INFO mistral.event_engine.default_event_engine [-] Starting event notification task...
2017-12-22 22:42:58.571 16966 INFO mistral.event_engine.default_event_engine [-] Found 0 event triggers.
/usr/local/lib/python2.7/dist-packages/oslo_messaging/server.py:341: FutureWarning: blocking executor is deprecated. Executor default will be removed. Use explicitly threading or eventlet instead in version 'pike' and will be removed in version 'rocky'
  category=FutureWarning)
2017-12-22 22:42:58.913 16966 WARNING oslo_config.cfg [req-46885c1b-de53-4bba-958d-97484cd17783 - - - - -] Option "auth_uri" from group "keystone_authtoken" is deprecated for removal (The auth_uri option is deprecated in favor of www_authenticate_uri and will be removed in the S  release.).  Its value may be silently ignored in the future.
2017-12-22 22:42:58.915 16966 WARNING oslo_config.cfg [req-46885c1b-de53-4bba-958d-97484cd17783 - - - - -] Option "auth_uri" from group "keystone_authtoken" is deprecated. Use option "www_authenticate_uri" from group "keystone_authtoken".
2017-12-22 22:42:58.925 16966 WARNING keystonemiddleware.auth_token [req-46885c1b-de53-4bba-958d-97484cd17783 - - - - -] AuthToken middleware is set with keystone_authtoken.service_token_roles_required set to False. This is backwards compatible but deprecated behaviour. Please set this to True.
2017-12-22 22:42:58.926 16966 WARNING keystonemiddleware.auth_token [req-46885c1b-de53-4bba-958d-97484cd17783 - - - - -] Use of the auth_admin_prefix, auth_host, auth_port, auth_protocol, identity_uri, admin_token, admin_user, admin_password, and admin_tenant_name configuration options was deprecated in the Mitaka release in favor of an auth_plugin and its related options. This class may be removed in a future release.
2017-12-22 22:42:58.931 16966 INFO oslo.service.wsgi [req-46885c1b-de53-4bba-958d-97484cd17783 - - - - -] mistral_api listening on 0.0.0.0:8989
2017-12-22 22:42:58.932 16966 INFO oslo_service.service [req-46885c1b-de53-4bba-958d-97484cd17783 - - - - -] Starting 4 workers
API server started.
API server started.
API server started.
API server started.
Event engine server started.
Executor server started.
Engine server started.
```

# 六、测试一下Mistral是否可用
```
openstack@controller:~/mistral/etc$ mistral workbook-list
+--------+--------+------------+------------+
| Name   | Tags   | Created at | Updated at |
+--------+--------+------------+------------+
| <none> | <none> | <none>     | <none>     |
+--------+--------+------------+------------+

openstack@controller:~/mistral/etc$ mistral action-list
+--------+--------+-----------+--------+-------------+--------+------------+------------+
| ID     | Name   | Is system | Input  | Description | Tags   | Created at | Updated at |
+--------+--------+-----------+--------+-------------+--------+------------+------------+
| <none> | <none> | <none>    | <none> | <none>      | <none> | <none>     | <none>     |
+--------+--------+-----------+--------+-------------+--------+------------+------------+
```
OK，成功了，开心！！！

---
