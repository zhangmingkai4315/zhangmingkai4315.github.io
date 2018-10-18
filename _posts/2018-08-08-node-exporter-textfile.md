---
id: 107
title: 使用textfile收集监控指标
date: 2018-08-08T14:31:11+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=107
permalink: /2018/08/node-exporter-textfile/
categories:
  - devops
tags:
  - prometheus
  - 监控系统
format: aside
---
node_exporter本身除了收集系统指标以外,还可以通过textfile模块来采集用户自己生成的指标,这对于系统监控提供了更多的使用空间和场景. 比如我们通过shell脚本采集的数据结果就可以通过该途径传递出去,用于绘图或告警等.

默认情况下node\_exporter将启用textfile组建,但是需要设置一个采集的路径,所有的生成的监控指标将放在该目录下,并以.prom文件名结尾. 同时node\_exporter启动方式如下:

    ./node_exporter --collector.textfile.directory=$PWD/textfile
    

##### 输出格式

所有自定义生成的指标需要按照如下的方式进行存储,首先使用shell或者python脚本最终写入文件的格式需要如下:

    # HELP example_metric Metric read from /some/path/textfile/example.prom
    # TYPE example_metric untyped
    example_metric 1
    

如果没有写help的话,系统会帮助生成一个简单的help描述,但是如果有多个文件中出现相同的指标名称(example\_metric),需要保证这些指标的help和type都一致,否则采集将出错. 基本格式也可以参考node\_exporter/metrics路径下显示的内容.

##### 任务执行

一般脚本任务(输出指标到.prom文件)会被放入crontab中执行,按照需求设置采集指标的时间, 同时node_exporter采集的时候如果正好文件执行写入操作,可能导致文件出现问题,安全期间我们可以将任务先转移到一个临时文件,然后通过临时文件的重命名进行操作,降低风险.

    */5 * * * * $TEXTFILE/printMetrics.sh > $TEXTFILE/metrics.prom.$$ && mv $TEXTFILE/metrics.prom.$$ $TEXTFILE/metrics.prom
    

##### 指标采集

对于.prom文件的采集,系统会自动的加入采集文件的修改时间,通过该指标我们可以设置一定的告警用于判断,是否文件发生了变化,比如采集指标时间为每10分钟一次,那么修改时间应该<15分钟,否则就应该报警上次的采集未成功. 指标名称:**node\_textfile\_mtime_seconds**, 指标收集时间为unixtime格式时间.

同时除了加载一些探测信息,使用该方式还可以用于静态信息的收集,比如定义的系统角色信息,或者服务器特殊的配置信息等等. 这些也都可以通过metrics的方式进行传递.

##### 采集实例

以下为官方git上提供的一个脚本用于采集文件夹目录大小的shell脚本, 并给出了cron的相关配置

    #!/bin/sh
    #
    # Expose directory usage metrics, passed as an argument.
    #
    # Usage: add this to crontab:
    #
    # */5 * * * * prometheus directory-size.sh /var/lib/prometheus | sponge /var/lib/node_exporter/directory_size.prom
    #
    # sed pattern taken from https://www.robustperception.io/monitoring-directory-sizes-with-the-textfile-collector/
    #
    # Author: Antoine Beaupré <anarcat@debian.org>
    echo "# HELP node_directory_size_bytes Disk space used by some directories"
    echo "# TYPE node_directory_size_bytes gauge"
    du --block-size=1 --summarize "$@" \
      | sed -ne 's/\\/\\\\/;s/"/\\"/g;s/^\([0-9]\+\)\t\(.*\)$/node_directory_size_bytes{directory="\2"} \1/p'
    
    

更多例子,可以访问[Textfile脚本](https://github.com/prometheus/node_exporter/tree/master/text_collector_examples)