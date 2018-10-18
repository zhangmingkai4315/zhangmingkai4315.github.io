---
id: 84
title: Prometheus源码笔记-config模块
date: 2018-07-25T17:22:56+00:00
author: mingkai
layout: post
guid: http://mikezhang.cc/?p=84
permalink: /2018/07/prometheus-config/
categories:
  - devops
tags:
  - golang
  - prometheus
  - readcode
format: aside
---
config模块主要用于配置文件的加载与管理操作，从config.yml文件中传递的数据将会用于生成一个config结构体对象，全局的config定义如下。通过`"gopkg.in/yaml.v2"`解析yaml模块并按照所属的组再细分到单独的配置项目中，比如global和alert组下的配置。本节将通过一个实例来解释模块对于config的配置过程。

    <br />type Config struct {
        GlobalConfig   GlobalConfig    `yaml:"global"`
        AlertingConfig AlertingConfig  `yaml:"alerting,omitempty"`
        RuleFiles      []string        `yaml:"rule_files,omitempty"`
        ScrapeConfigs  []*ScrapeConfig `yaml:"scrape_configs,omitempty"`
    
        RemoteWriteConfigs []*RemoteWriteConfig `yaml:"remote_write,omitempty"`
        RemoteReadConfigs  []*RemoteReadConfig  `yaml:"remote_read,omitempty"`
    
        // Catches all undefined fields and must be empty after parsing.
        XXX map[string]interface{} `yaml:",inline"`
    
        // original is the input from which the config was parsed.
        original string
    }
    

另外对于每个配置项目都设置了缺省的配置量比如下面的定义GlobalConfig，用于加载配置文件中属于global区域的配置文件，定义如下：

    <br />// GlobalConfig configures values that are used across other configuration
    // objects.
    type GlobalConfig struct {
        // How frequently to scrape targets by default.
        ScrapeInterval model.Duration `yaml:"scrape_interval,omitempty"`
        // The default timeout when scraping targets.
        ScrapeTimeout model.Duration `yaml:"scrape_timeout,omitempty"`
        // How frequently to evaluate rules by default.
        EvaluationInterval model.Duration `yaml:"evaluation_interval,omitempty"`
        // The labels to add to any timeseries that this Prometheus instance scrapes.
        ExternalLabels model.LabelSet `yaml:"external_labels,omitempty"`
    
        // Catches all undefined fields and must be empty after parsing.
        XXX map[string]interface{} `yaml:",inline"`
    }
    
    

缺省的DefaultGlobal的配置如下：

    <br />    // DefaultGlobalConfig is the default global configuration.
    DefaultGlobalConfig = GlobalConfig{
            ScrapeInterval:     model.Duration(1 * time.Minute),
            ScrapeTimeout:      model.Duration(10 * time.Second),
            EvaluationInterval: model.Duration(1 * time.Minute),
        }
    
    
    

函数UnmarshalYAML用于将缺省的与实际配置的进行合并，形成最终的配置

    func (c *GlobalConfig) UnmarshalYAML(unmarshal func(interface{}) error) error {
        // Create a clean global config as the previous one was already populated
        // by the default due to the YAML parser behavior for empty blocks.
        gc := &GlobalConfig{}
        type plain GlobalConfig
        if err := unmarshal((*plain)(gc)); err != nil {
            return err
        }
        if err := checkOverflow(gc.XXX, "global config"); err != nil {
            return err
        }
        // First set the correct scrape interval, then check that the timeout
        // (inferred or explicit) is not greater than that.
        if gc.ScrapeInterval == 0 {
            gc.ScrapeInterval = DefaultGlobalConfig.ScrapeInterval
        }
        if gc.ScrapeTimeout > gc.ScrapeInterval {
            return fmt.Errorf("global scrape timeout greater than scrape interval")
        }
        if gc.ScrapeTimeout == 0 {
            if DefaultGlobalConfig.ScrapeTimeout > gc.ScrapeInterval {
                gc.ScrapeTimeout = gc.ScrapeInterval
            } else {
                gc.ScrapeTimeout = DefaultGlobalConfig.ScrapeTimeout
            }
        }
        if gc.EvaluationInterval == 0 {
            gc.EvaluationInterval = DefaultGlobalConfig.EvaluationInterval
        }
        *c = *gc
        return nil
    }
    

其他config区域的配置基本一致，此处不一一列出。

模块中的测试部分包含一个testdata目录存储了实际测试中需要的配置内容，比如conf和rules等配置。