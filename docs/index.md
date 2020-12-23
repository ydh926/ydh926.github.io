## 简介

slime是针对istio的扩展工具。旨在通过简单配置，更自动更便捷的使用istio/envoy高阶功能。其包含了多个模块：

**[配置懒加载](/getting_started/lazyload):** 无须配置SidecarScope，自动按需加载配置/服务发现信息  

**[Http插件管理](/getting_started/plugin):** 使用新的的CRD pluginmanager/envoyplugin包装了可读性，可维护性差的envoyfilter,使得插件扩展更为便捷  

**[自适应限流](/getting_started/ratelimit):** 实现了本地限流，同时可以结合监控信息自动调整限流策略

选择你感兴趣的模块快速开始吧~
