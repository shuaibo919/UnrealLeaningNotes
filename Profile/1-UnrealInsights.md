参考资料:
- [Unreal官方文档 Unreal Insights](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/unreal-insights-in-unreal-engine)
- [Unreal官方社区 unreal-insights-traces-on-android](https://dev.epicgames.com/community/learning/tutorials/eB9R/unreal-engine-gathering-unreal-insights-traces-on-android)
- [Unreal官方文档 how-to-use-unreal-insights-to-profile-android-games-for-unreal-engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/how-to-use-unreal-insights-to-profile-android-games-for-unreal-engine)

Unreal Insights是虚幻自带的性能分析工具，可以实时查看性能数据，分析卡点。


## 实用Tips

```Trace.SnapshotFile <filename>```:将当前内存内跟踪缓冲区的快照写入一个文件。如果你已经在主动跟踪，它不会中断主动跟踪，而是为这个快照并行地记录第二个跟踪文件

```Trace.Bookmark <name>```:使用给定的字符串名称发射一个Bookmark事件。被记录的Bookmark以竖线的形式出现在Timing Insights中

```Trace.Screenshot <Name> <bIncludeUI>```:可以运行此控制台指令以生成竖线，并通过在Timing Insights中指定快照的true或者false来选择是否包含UI。


## 添加自定义事件
宏魔法，启动！

阅读该官方文档[Unreal官方文档 developer-guide-to-tracing-in-unreal-engine](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/developer-guide-to-tracing-in-unreal-engine)


## 快捷键参考

[Unreal官方文档 Unreal Insights快捷键](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/unreal-insights-reference-in-unreal-engine-5#%E9%94%AE%E7%9B%98%E8%BE%93%E5%85%A5%E5%BF%AB%E6%8D%B7%E9%94%AE)


## 移动端调试

### Android