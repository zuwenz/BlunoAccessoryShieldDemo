# GD32C103 多协议工业 IO 板项目功能规划

## 1. 项目目标
- 基于 **GD32C103** 设计一款可量产、可扩展的工业 IO 板。
- 支持多种工业通信协议，重点覆盖：
  - **Modbus RTU / Modbus TCP**
  - **EtherCAT（从站）**
  - **PROFINET IO Device**
  - （可选）EtherNet/IP、CANopen、OPC UA 网关模式
- 提供统一对象模型，将不同协议映射到同一套 IO、参数、诊断数据。

## 2. 典型应用场景
- 产线设备数字量采集与执行控制（DI/DO）
- 模拟量采集与闭环控制（AI/AO）
- 老旧 RS-485 网络接入以太网/工业以太网
- PLC（西门子/倍福/欧姆龙等）统一接入

## 3. 系统功能分层

### 3.1 硬件层
- MCU：GD32C103（主控）
- 通信接口：
  - RS-485（隔离）用于 Modbus RTU
  - 10/100M 以太网 PHY（预留工业以太网扩展）
  - SPI/QSPI（连接 EtherCAT 从站控制器，建议外置 ESC 芯片）
  - 可选 CAN 接口（CANopen）
- IO 资源建议：
  - DI：8~16 路（光耦隔离、支持脉冲计数）
  - DO：8~16 路（晶体管/继电器可选）
  - AI：4~8 路（4-20mA / 0-10V）
  - AO：2~4 路（4-20mA / 0-10V）
- 保护设计：电源反接、浪涌、ESD、过流、短路保护。

### 3.2 固件平台层
- Bootloader（串口/以太网升级，双镜像回滚）
- RTOS（推荐 FreeRTOS）或事件驱动调度
- BSP/HAL 抽象：GPIO、ADC、DAC、UART、SPI、ETH、EEPROM
- 参数管理：掉电保存、版本兼容、默认恢复
- 日志与诊断：故障码、运行统计、看门狗复位原因

### 3.3 协议与应用层
- 协议栈插件化：
  - `protocol_modbus`
  - `protocol_ethercat`
  - `protocol_profinet`
  - `protocol_cip`（预留）
- 数据模型统一：
  - Process Image（周期 IO）
  - Parameter Object（配置参数）
  - Diagnostic Object（报警与状态）
- 本地逻辑：滤波、去抖、阈值报警、简单联锁逻辑

## 4. 重点协议规划

### 4.1 Modbus
- 支持 Modbus RTU（主/从可配置，优先从站）
- 支持 Modbus TCP Server
- 地址映射：
  - Coil -> DO
  - Discrete Input -> DI
  - Input Register -> AI/状态量
  - Holding Register -> 参数与控制命令
- 提供寄存器表自动导出（CSV/Excel）

### 4.2 EtherCAT
- 目标角色：EtherCAT Slave
- 建议架构：GD32C103 + 外置 ESC（如 LAN9252 类）
- 支持 CoE（CANopen over EtherCAT）基础对象字典
- 支持周期同步（DC 基础支持）与 PDO 映射
- 输出 ESI（EtherCAT XML）文件用于主站工程导入

### 4.3 PROFINET
- 目标角色：PROFINET IO Device（RT 级）
- 模块化子槽设计：DI/DO/AI/AO 作为可插拔模块
- 提供 GSDML 文件用于 PLC 工程配置
- 支持设备名、IP 分配、诊断报警、参数记录

## 5. 配置与运维功能
- Web 配置页面（基础版）：网络、协议启停、IO 标定
- 串口 CLI：工厂调试、状态查询、快速恢复
- 远程升级（FOTA/LAN）与版本回退
- 运行状态上报：心跳、温度、电源电压、通信质量

## 6. 安全与可靠性设计
- 通信安全：基础访问控制（口令/角色）
- 配置保护：关键参数写保护与审计日志
- 可靠性：
  - 看门狗 + 任务心跳
  - 通信超时与故障安全输出（Fail-safe）
  - 断电参数一致性校验（CRC）
- 工业环境指标建议：
  - 工作温度：-20°C ~ +70°C
  - EMC 满足工业现场等级（依据目标市场标准细化）

## 7. 项目实施里程碑（建议）
1. **M1 需求冻结（2 周）**：IO 点表、目标协议、成本目标。
2. **M2 原理图/PCB（4 周）**：完成 EVT 硬件打样。
3. **M3 BSP+IO 驱动（4 周）**：基础硬件功能跑通。
4. **M4 Modbus 版本（4 周）**：完成 RTU/TCP 联调并可小批测试。
5. **M5 EtherCAT 版本（6~8 周）**：主站联调、周期稳定性测试。
6. **M6 PROFINET 版本（6~8 周）**：PLC 互操作与一致性测试。
7. **M7 认证与量产（4~8 周）**：EMC、环境、可靠性与文档归档。

## 8. 验收指标（KPI）
- IO 周期刷新：<= 5 ms（协议相关可分档）
- Modbus 读写成功率：>= 99.9%
- EtherCAT 周期稳定抖动：满足目标工况定义
- PROFINET PLC 兼容性：主流品牌至少 2 家通过
- 连续运行稳定性：7x24 小时压力测试无异常

## 9. 风险与规避
- **协议实现复杂度高**：优先引入成熟协议栈或芯片方案。
- **GD32C103 资源瓶颈**：通过外置 ESC、任务裁剪、内存池优化降低风险。
- **认证周期不可控**：在 EV 阶段提前进行预一致性测试。
- **多协议并存冲突**：采用统一对象模型 + 单协议独占运行策略（首版）。

## 10. 首版最小可行产品（MVP）建议
- 硬件：8DI + 8DO + 4AI + 2AO，1xRS485，1xEthernet
- 协议：Modbus RTU + Modbus TCP（首发）
- 扩展：预留 EtherCAT/PROFINET 扩展接口与软件框架
- 工具：寄存器映射文档 + Web 配置 + Bootloader 升级

---

如果你愿意，我下一步可以直接给你：
1) **硬件框图 + 关键器件清单（BOM 级别）**；
2) **寄存器映射表初版（可直接给 PLC 工程师）**；
3) **固件目录结构模板（C 工程骨架）**。
