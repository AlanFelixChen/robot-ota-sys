
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

docs文档：

| 序号 | 文件名 | 文档定位 | 核心内容（简要） |
| --- | --- | --- | --- |
| 1 | architecture.md | 系统架构文档 | 七层架构图（云端→边缘→应用→中间件→OS→驱动→硬件），各层模块职责说明，安全信任链，数据流向 |
| 2 | design.md | 系统设计文档 | 升级主流程（PRECHECK→DOWNLOAD→VERIFY→FLASH→ACTIVATE→COMMIT），回滚流程，MCU协同升级时序，关键决策记录（SWUpdate选型、zchunk选型、分区布局考量） |
| 3 | state-machine.md | 状态机设计文档 | 状态定义（IDLE/PRECHECK/DOWNLOADING/VERIFYING/FLASHING/ACTIVATING/COMMITTING/ROLLBACK/ERROR），状态转移条件，异常处理路径 |
| 4 | differential.md | 差分升级文档 | zchunk差分算法原理，差分包生成流程，设备端还原流程，压缩率对比数据，适用场景与限制 |
| 5 | security.md | 安全设计文档 | 信任链（ATECC608→U-Boot→Kernel），ECDSA P-256签名验签流程，TLS双向认证，RPMB密钥存储，威胁模型分析 |
| 6 | protocol.md | 接口协议文档 | MQTT Topic定义与JSON Schema，HTTPS固件下载接口，UART/CAN MCU通信帧格式，错误码定义表 |
| 7 | power-failure.md | 掉电保护文档 | 各阶段掉电风险分析，Flash写入原子性保障，UBIFS日志恢复机制，U-Boot回滚保护，测试验证方法 |
| 8 | production.md | 生产部署文档 | 云端服务部署（Docker Compose），设备端Flash分区烧录，U-Boot环境变量配置，证书部署，产线烧录流程 |
| 9 | test-plan.md | 测试方案文档 | 功能测试用例（全量/差分/MCU升级），异常测试（断网/断电/存储满），性能测试（耗时/压缩率），安全测试（签名伪造/重放攻击） |
| 10 | mcu-upgrade-spec.md | MCU升级规范 | MCU Bootloader最小要求，UART/CAN升级帧协议，分块传输与校验，版本兼容矩阵，升级失败恢复机制 |

文档间依赖关系：
architecture.md  ← 总纲，所有文档的基础
    ├── design.md  ← 核心流程设计
    │       ├── state-machine.md  ← 状态机细化
    │       └── differential.md   ← 差分方案细化
    ├── security.md  ← 安全设计
    ├── protocol.md  ← 接口协议
    ├── power-failure.md  ← 掉电保护
    ├── production.md  ← 部署运维
    ├── test-plan.md  ← 测试方案
    └── mcu-upgrade-spec.md  ← MCU规范


## 第三方组件

本项目使用 Apache License 2.0 发布。项目中引用的第三方组件遵循其各自的许可证。
完整列表见 [NOTICE](NOTICE) 文件。

## License

本项目采用 [Apache License 2.0](LICENSE) 开源协议。

## 版权声明
Copyright © 2026 AlanFelixChen

除非另有说明，本项目的所有代码、文档、设计文件均为项目作者所有。