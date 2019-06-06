---
layout: post
title: Ansible实现Inventory动态生成
date: 2019-06-05T11:32:41+00:00
author: mingkai
permalink: /2019/06/cmdb-with-dynamic-ansible-inventory
categories:
  - devops
---


日常执行ansible或者ansible-playbook我们可以通过```-i ```传递以下几种参数来获取其主机inventory信息

- 主机列表string , 以逗号分开
- yaml文件： 配置文件
- ini文件 ： 配置文件
- script或插件 ： 可以执行并返回一个json对象的脚本

对于Ansible系统规模较小的时候，可以直接通过传递静态配置文件实现主机的管理维护，但是随着主机数量的增加，修改主机的配置也越来越多，每次都去修改静态hosts文件变得越来越难以维护。其中官方支持的第四种方式就是通过脚本或者插件的方式来获取动态的inventory信息

本篇文章将介绍如何实现动态的生成inventory配置文件，动态生成主机配置可以通过查询内部的cloud云平台接口，LDAP目录，以及企业内部自身的CMDB系统来获取（实现查询接口即可）。

当前Ansible集成动态inventory配置生成方式有两种：

- 插件方式： Ansible官方推荐使用的方式，通过自己写插件的方式集成到系统中，但是对于版本有要求需要2.4版本及以上使用
- 自定义脚本： 直接编写符合要求的脚本，相对于插件方式需要手动实现比如缓存管理，动态变量和组管理的功能等。

以下就分别介绍两种方式的实现并给出一些代码实例。


#### 1. 开发一个inventory插件

使用ansible的sdk来开发inventory 插件，需要继承已定义好的几个基类和实现其中的两种重要的方法即可。

```python
from ansible.plugins.inventory import BaseInventoryPlugin, Constructable, Cacheable

class InventoryModule(BaseInventoryPlugin, Constructable, Cacheable):
	# 定义模块的名称
    NAME = 'myplugin'
```

两个重要的方法分别是：

- verify_file: 这个方法主要用来判断是否传递的配置文件（比如 config.yaml or config.ini）是可用的或者存在的，比如下面的仅仅判断是否文件名称是否符合预期的定义。 该配置文件与-i传递的脚本没有关系，仅用于脚本中需要的任何其他的配置文件， 比如连接LDAP或CMDB连接信息的配置文件（可以存放在/etc/cmdb.ini下面，检查该文件是否存在或者可用)。

  ```python
  def verify_file(self, path):
      valid = False
      if super(InventoryModule, self).verify_file(path):
          # 简单的验证是否存在或者是否以特定的文件名结尾
          if path.endswith(('virtualbox.yaml', 'virtualbox.yml', 'vbox.yaml', 'vbox.yml')):
              valid = True
      return valid
  ```

- parser()： 函数作为核心处理将完成以下几个重要的功能

  - 载入数据并且缓存数据。
  - 缓存设置，启用缓存或者避免缓存数据
  - 主机管理： 增加主机和组管理到inventory对象
  - 配置管理 : 载入配置信息

  下面的代码中将设置参数并通过查询api接口数据，获取所有inventory的结果，

  ```python
  def parse(self, inventory, loader, path, cache=True):
  
       # 父类初始化
       super(InventoryModule, self).parse(inventory, loader, path, cache)
  
       # 获取配置文件，并提供self.get_option获取配置
       config = self._read_config_data(path)
       # 实例： 获取远程数据
       mysession = apilib.session(user=self.get_option('api_user'),
                                  password=self.get_option('api_pass'),
                                  server=self.get_option('api_server')
       )
       mydata = mysession.getitall()
  
       # 处理获取到的数据并存储，比如按照数据中心或者服务域进行分类处理生成配置
       # 依赖于数据源的分类
       for colo in mydata:
           for server in mydata[colo]['servers']:
               self.inventory.add_host(server['name'])
               self.inventory.set_variable(server['name'], 'ansible_host', server['external_ip'])
  ```



#### 2. 开发一个inventory脚本

使用inventory脚本可以允许使用其他编程语言去生成对应的json数据，只需要满足几个条件即可。首先脚本必须能顾支持--list和--host <hostname>的参数传递查询。当传递--list的时候，需要获取到所有的分组的inventory信息，比如下面的：

```json
{
    "group001": {
        "hosts": ["host001", "host002"],
        "vars": {
            "var1": true,
            "var2": false
        },
        "children": ["group002"]
    },
    "group002": {
        "hosts": ["host003","host004"],
        "vars": {
            "var2": 500
        },
        "children":[]
    }

}
```

当调用--host查询的时候返回所有该主机的环境变量，比如针对上述例子中的主机host001 当我们查询--host host001的时候，返回

```json
{
    "var1":true,
    "var2": false
}
```

返回变量是可选的，如果没有实施的话可以返回一个空的dict json对象

一个可供参考的实例是通过cobbra的服务器获取其对应的管理主机, 程序将读取其中的cobbra.ini文件获取其配置信息。通过配置信息去连接服务器获得主机信息。

对应的配置文件如下

```ini
[cobbler]
host = http://127.0.0.1/cobbler_api
cache_path = /tmp
cache_max_age = 900
```

定义的脚本如下：

```python
#!/usr/bin/env python

import argparse
import os
import re
from time import time
import xmlrpclib

import json

from ansible.module_utils.six import iteritems
from ansible.module_utils.six.moves import configparser as ConfigParser

orderby_keyname = 'owners'  # alternatively 'mgmt_classes'


class CobblerInventory(object):

    def __init__(self):

        """ Main execution path """
        self.conn = None

        self.inventory = dict()  # A list of groups and the hosts in that group
        self.cache = dict()  # Details about hosts in the inventory
        self.ignore_settings = False  # used to only look at env vars for settings.

        # Read env vars, read settings, and parse CLI arguments
        self.parse_env_vars()
        self.read_settings()
        self.parse_cli_args()

        # Cache
        if self.args.refresh_cache:
            self.update_cache()
        elif not self.is_cache_valid():
            self.update_cache()
        else:
            self.load_inventory_from_cache()
            self.load_cache_from_cache()

        data_to_print = ""

        # Data to print
        if self.args.host:
            data_to_print += self.get_host_info()
        else:
            self.inventory['_meta'] = {'hostvars': {}}
            for hostname in self.cache:
                self.inventory['_meta']['hostvars'][hostname] = {'cobbler': self.cache[hostname]}
            data_to_print += self.json_format_dict(self.inventory, True)

        print(data_to_print)

    def _connect(self):
        if not self.conn:
            self.conn = xmlrpclib.Server(self.cobbler_host, allow_none=True)
            self.token = None
            if self.cobbler_username is not None:
                self.token = self.conn.login(self.cobbler_username, self.cobbler_password)

    def is_cache_valid(self):
        """ Determines if the cache files have expired, or if it is still valid """

        if os.path.isfile(self.cache_path_cache):
            mod_time = os.path.getmtime(self.cache_path_cache)
            current_time = time()
            if (mod_time + self.cache_max_age) > current_time:
                if os.path.isfile(self.cache_path_inventory):
                    return True

        return False

    def read_settings(self):
        """ Reads the settings from the cobbler.ini file """

        if(self.ignore_settings):
            return

        config = ConfigParser.SafeConfigParser()
        config.read(os.path.dirname(os.path.realpath(__file__)) + '/cobbler.ini')

        self.cobbler_host = config.get('cobbler', 'host')
        self.cobbler_username = None
        self.cobbler_password = None
        if config.has_option('cobbler', 'username'):
            self.cobbler_username = config.get('cobbler', 'username')
        if config.has_option('cobbler', 'password'):
            self.cobbler_password = config.get('cobbler', 'password')

        # Cache related
        cache_path = config.get('cobbler', 'cache_path')
        self.cache_path_cache = cache_path + "/ansible-cobbler.cache"
        self.cache_path_inventory = cache_path + "/ansible-cobbler.index"
        self.cache_max_age = config.getint('cobbler', 'cache_max_age')

    def parse_env_vars(self):
        """ Reads the settings from the environment """

        # Env. Vars:
        #   COBBLER_host
        #   COBBLER_username
        #   COBBLER_password
        #   COBBLER_cache_path
        #   COBBLER_cache_max_age
        #   COBBLER_ignore_settings

        self.cobbler_host = os.getenv('COBBLER_host', None)
        self.cobbler_username = os.getenv('COBBLER_username', None)
        self.cobbler_password = os.getenv('COBBLER_password', None)

        # Cache related
        cache_path = os.getenv('COBBLER_cache_path', None)
        if(cache_path is not None):
            self.cache_path_cache = cache_path + "/ansible-cobbler.cache"
            self.cache_path_inventory = cache_path + "/ansible-cobbler.index"

        self.cache_max_age = int(os.getenv('COBBLER_cache_max_age', "30"))

        # ignore_settings is used to ignore the settings file, for use in Ansible
        # Tower (or AWX inventory scripts and not throw python exceptions.)
        if(os.getenv('COBBLER_ignore_settings', False) == "True"):
            self.ignore_settings = True

    def parse_cli_args(self):
        """ Command line argument processing """

        parser = argparse.ArgumentParser(description='Produce an Ansible Inventory file based on Cobbler')
        parser.add_argument('--list', action='store_true', default=True, help='List instances (default: True)')
        parser.add_argument('--host', action='store', help='Get all the variables about a specific instance')
        parser.add_argument('--refresh-cache', action='store_true', default=False,
                            help='Force refresh of cache by making API requests to cobbler (default: False - use cache files)')
        self.args = parser.parse_args()

    def update_cache(self):
        """ Make calls to cobbler and save the output in a cache """

        self._connect()
        self.groups = dict()
        self.hosts = dict()
        if self.token is not None:
            data = self.conn.get_systems(self.token)
        else:
            data = self.conn.get_systems()

        for host in data:
            # Get the FQDN for the host and add it to the right groups
            dns_name = host['hostname']  # None
            ksmeta = None
            interfaces = host['interfaces']
            # hostname is often empty for non-static IP hosts
            if dns_name == '':
                for (iname, ivalue) in iteritems(interfaces):
                    if ivalue['management'] or not ivalue['static']:
                        this_dns_name = ivalue.get('dns_name', None)
                        if this_dns_name is not None and this_dns_name is not "":
                            dns_name = this_dns_name

            if dns_name == '' or dns_name is None:
                continue

            status = host['status']
            profile = host['profile']
            classes = host[orderby_keyname]

            if status not in self.inventory:
                self.inventory[status] = []
            self.inventory[status].append(dns_name)

            if profile not in self.inventory:
                self.inventory[profile] = []
            self.inventory[profile].append(dns_name)

            for cls in classes:
                if cls not in self.inventory:
                    self.inventory[cls] = []
                self.inventory[cls].append(dns_name)

            # Since we already have all of the data for the host, update the host details as well

            # The old way was ksmeta only -- provide backwards compatibility

            self.cache[dns_name] = host
            if "ks_meta" in host:
                for key, value in iteritems(host["ks_meta"]):
                    self.cache[dns_name][key] = value

        self.write_to_cache(self.cache, self.cache_path_cache)
        self.write_to_cache(self.inventory, self.cache_path_inventory)

    def get_host_info(self):
        """ Get variables about a specific host """

        if not self.cache or len(self.cache) == 0:
            # Need to load index from cache
            self.load_cache_from_cache()

        if self.args.host not in self.cache:
            # try updating the cache
            self.update_cache()

            if self.args.host not in self.cache:
                # host might not exist anymore
                return self.json_format_dict({}, True)

        return self.json_format_dict(self.cache[self.args.host], True)

    def push(self, my_dict, key, element):
        """ Pushed an element onto an array that may not have been defined in the dict """

        if key in my_dict:
            my_dict[key].append(element)
        else:
            my_dict[key] = [element]

    def load_inventory_from_cache(self):
        """ Reads the index from the cache file sets self.index """

        cache = open(self.cache_path_inventory, 'r')
        json_inventory = cache.read()
        self.inventory = json.loads(json_inventory)

    def load_cache_from_cache(self):
        """ Reads the cache from the cache file sets self.cache """

        cache = open(self.cache_path_cache, 'r')
        json_cache = cache.read()
        self.cache = json.loads(json_cache)

    def write_to_cache(self, data, filename):
        """ Writes data in JSON format to a file """
        json_data = self.json_format_dict(data, True)
        cache = open(filename, 'w')
        cache.write(json_data)
        cache.close()

    def to_safe(self, word):
        """ Converts 'bad' characters in a string to underscores so they can be used as Ansible groups """

        return re.sub(r"[^A-Za-z0-9\-]", "_", word)

    def json_format_dict(self, data, pretty=False):
        """ Converts a dict to a JSON object and dumps it as a formatted string """

        if pretty:
            return json.dumps(data, sort_keys=True, indent=2)
        else:
            return json.dumps(data)


CobblerInventory()
```

