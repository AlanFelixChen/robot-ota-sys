
# robot-ota-sys

为机器人设计的量产级OTA升级系统，支持主控+多MCU节点的安全可靠固件升级。

## 核心特性

- &zwnj;**A/B分区升级**&zwnj;：基于SWUpdate，支持无缝回滚
- &zwnj;**差分升级**&zwnj;：zchunk块级差分，节省带宽
- &zwnj;**多节点协同**&zwnj;：主控协调多MCU同步升级
- &zwnj;**纵深安全**&zwnj;：安全启动 + 签名校验 + 防回滚 + 通道隔离
- &zwnj;**断电恢复**&zwnj;：下载/刷写/重启全链路断电保护
- &zwnj;**量产适配**&zwnj;：工厂通道、产线测试、老化测试

## 技术栈

| 模块 | 技术选型 |
|------|----------|
| 设备端语言 | C/C++ (主控), C (MCU) |
| 云端语言 | Go/Python |
| 通信协议 | MQTT(TLS) + HTTPS |
| 差分算法 | zchunk |
| 签名算法 | ECDSA P-256 |
| 安全存储 | eMMC RPMB + ATECC608 |

## 快速开始

见各模块目录下的 README

## 第三方组件

本项目使用 Apache License 2.0 发布。项目中引用的第三方组件遵循其各自的许可证。
完整列表见 [NOTICE](NOTICE) 文件。

## License

本项目采用 [Apache License 2.0](LICENSE) 开源协议。

