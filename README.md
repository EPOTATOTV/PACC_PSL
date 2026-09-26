# PACC Rule Language

> PRL 是 PACC 玩家端反作弊系统使用的检测规则 DSL 的完整实现。

[![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk)](https://openjdk.org/projects/jdk/21/)
[![Maven](https://img.shields.io/badge/Maven-3.9%2B-blue?logo=apache-maven)](https://maven.apache.org/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Build](https://img.shields.io/badge/build-passing-brightgreen)](https://github.com/EPOTATOTV/PACC_PSL/actions)

---

## 项目简介

PACC Rule Language（PRL）是 PACC 玩家端反作弊系统使用的检测规则 DSL。本模块是它的完整实现：词法分析、语法分析、强类型检查、IR 与 `.prlc` 字节码编译、寄存器式虚拟机、多层沙箱、标准库、规则热加载与版本管理、静态分析，以及调试器与 Profiler。

实现只用 JDK 标准库，运行时不依赖任何第三方库。`maven-enforcer-plugin` 会在构建期拦下往主代码里新增的 compile / runtime 依赖。

> **边界**：PRL 只做本地检测、留痕和红屏强制警告。规则里没有封禁玩家的能力；`ban_feature` 禁用的是一个客户端功能开关，不是账号。

---

## ✨ 核心特性

| 特性 | 说明 |
|------|------|
| **完整编译器管线** | 词法分析 → 语法分析 → 强类型检查 → 三地址码 IR → `.prlc` 字节码 |
| **寄存器式虚拟机** | 自研字节码解释器，执行编译后的检测规则 |
| **多层沙箱** | 宿主上下文接口与沙箱边界，规则运行在受控环境中 |
| **标准库** | 内置函数与检测原语，宿主未接管的函数交给标准库处理 |
| **规则热加载** | 看守规则目录，文件修改后自动重编，编译失败保留旧规则 |
| **版本管理与灰度** | 规则版本控制与灰度发布支持 |
| **静态分析** | 空指针、死循环、安全、冲突、复杂度等静态检查 |
| **调试器与 Profiler** | 断点、条件断点、日志点、单步、调用栈；性能分析 |

---

## 架构设计

PRL 遵循清晰的编译-执行分离架构：

```
┌──────────────────────────────────────────────────────────────┐
│                        源代码 (.prl)                          │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  lexer ──▶ parser ──▶ check ──▶ ir ──▶ bytecode (.prlc)     │
│  词法     语法分析    类型检查   三地址码  字节码生成          │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  vm ──▶ runtime ──▶ sandbox ──▶ stdlib                      │
│  寄存器式虚拟机    运行时值      沙箱边界     标准库            │
└──────────────────────┬───────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│  engine ──▶ analysis ──▶ debugger ──▶ profiler               │
│  规则管理/热加载   静态分析     调试器        性能分析          │
└──────────────────────────────────────────────────────────────┘
```

### 三条核心约束

- **不封禁玩家**：只做本地检测与记录、红屏警告、远程查端，不接入也不开发任何游戏服务器封禁能力
- **只取玩家本地数据**：所有检测数据都来自玩家设备本机采集，不接入任何游戏服务器 API 或数据库
- **沙箱隔离**：规则执行在受控沙箱内，无法逃逸到宿主环境

---

## 技术栈

| 层级 | 技术 |
|------|------|
| 语言 | Java 21 |
| 构建 | Maven 3.9+ |
| 运行时依赖 | 无（仅 JDK 标准库） |
| 构建期依赖检查 | maven-enforcer-plugin |
| 字节码格式 | `.prlc`（PRL 自定义字节码） |
| 虚拟机 | 寄存器式字节码解释器 |

---

## 快速开始

### 环境要求

- JDK 21
- Maven 3.9 及以上

### 构建与测试

```bash
# 跑单元测试
mvn -B -ntp test

# 完整校验：enforcer 依赖检查 + 单元测试
mvn -B -ntp verify

# 产出 target/pacc-rule-language-1.0.0.jar
mvn -B -ntp package
```

### Maven 坐标

```xml
<dependency>
    <groupId>com.potatotv</groupId>
    <artifactId>pacc-rule-language</artifactId>
    <version>1.0.0</version>
</dependency>
```

> 版本号独立于 PACC 产品版本走，主项目按精确版本锁定。

---

## 使用示例

### 编译一条规则

```java
PrlHostContext host = PrlHostContext.EMPTY; // 宿主没接管的函数交给标准库

PrlCompiler compiler = new PrlCompiler(host);
CompileResult result = compiler.compileChecked(source);

if (!result.ok()) {
    System.out.println(result.describe()); // 诊断带行列号
}
```

### 装进规则管理器再执行

```java
RuleManager manager = new RuleManager(host);
manager.loadRule("kill_aura", source);

Map<String, Object> input = Map.of(
    "player", playerContext,
    "attack_events", attackEvents,
    "system_state", systemState
);

for (DetectionResult alert : manager.executeAll(input)) {
    System.out.println(alert.type() + " " + alert.confidence());
}
```

### 规则热加载

```java
try (RuleHotLoader loader = new RuleHotLoader(Path.of("rules"), manager)) {
    loader.start();
}
```

看守一个规则目录，文件改了自动重编（编译失败会保留旧规则）。

---

## 示例规则

`examples/` 下有五个能直接编译的规则：

| 规则文件 | 说明 |
|----------|------|
| [kill_aura.prl](https://github.com/EPOTATOTV/PACC_PSL/blob/master/examples/kill_aura.prl) | 杀戮光环检测 |
| [speed_hack.prl](https://github.com/EPOTATOTV/PACC_PSL/blob/master/examples/speed_hack.prl) | 速度作弊检测 |
| [fly.prl](https://github.com/EPOTATOTV/PACC_PSL/blob/master/examples/fly.prl) | 飞行作弊检测 |
| [autoclicker.prl](https://github.com/EPOTATOTV/PACC_PSL/blob/master/examples/autoclicker.prl) | 自动点击检测 |
| [xray.prl](https://github.com/EPOTATOTV/PACC_PSL/blob/master/examples/xray.prl) | X-Ray 透视检测 |

---

## 包结构

| 包 | 职责 |
|---|------|
| `lexer` | 词法分析，产出 `Token` |
| `parser` | 递归下降语法分析，产出 AST |
| `ast` | AST 节点（`RuleFile`、`Rule`、`Expression`、`Statement` 等） |
| `types` | 类型模型、内置函数签名表、宿主类型注册表 |
| `check` | 类型检查与诊断 |
| `ir` | 三地址码 IR 生成 |
| `bytecode` | IR 到 `.prlc` 字节码，以及文件读写 |
| `vm` | 寄存器式字节码解释器 |
| `runtime` | 运行时值、宿主对象、运行时异常 |
| `sandbox` | 宿主上下文接口与沙箱边界 |
| `stdlib` | 标准库实现 |
| `compiler` | 编译门面 `PrlCompiler` |
| `engine` | 规则管理、热加载、版本与灰度 |
| `analysis` | 静态分析：空指针、死循环、安全、冲突、复杂度 |
| `debugger` | 调试器：断点、条件断点、日志点、单步、调用栈 |
| `profiler` | Profiler：性能分析 |

---

## 边界说明

PRL 只做本地检测、留痕和红屏强制警告。规则里没有封禁玩家的能力；`ban_feature` 禁用的是一个客户端功能开关，不是账号。

---

## 许可证

本项目采用 MIT 许可证。详见 [LICENSE](LICENSE) 文件。

---

<p align="center">
  <sub>Made with ❤️ by EPOTATOTV</sub>
</p>
