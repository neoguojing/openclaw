---
summary: OpenClaw 核心、gateway、channels、extensions 的仓库级架构地图
read_when:
  - 首次阅读 OpenClaw 代码结构
  - 规划跨 gateway/channels/agents 的重构
  - 梳理 core 与 plugin/extension 的边界
title: 代码库架构
---

# 代码库架构

最后更新：2026-03-09

## 全局形态

OpenClaw 以 `src/` 为单体运行时核心，主要由三条平面组成：

- **Gateway/控制平面**：WebSocket + HTTP 服务生命周期、鉴权、配对、订阅、方法分发。
- **Channel 平面**：入站/出站适配、路由与会话绑定、渠道 onboarding 与配置。
- **Agent/工具平面**：智能体执行、工具系统、memory/context、多智能体运行时。

## 启动入口

- `openclaw.mjs`：Node 版本守卫、编译缓存、warning filter、加载 `dist/entry.*`。
- `src/index.ts`：dotenv/env/path/runtime guard、日志捕获、CLI program 构建、顶层错误处理。
- `src/cli/program.ts` 与 `src/cli/program/build-program.ts`：命令树装配与上下文注入。

## src 分层地图

- `src/gateway/`：网关运行时、WS 方法、健康检查、重载、节点与 sidecar。
- `src/channels/` + `src/routing/`：渠道元数据、策略、会话/账号路由解析。
- `src/agents/`：智能体执行内核、工具编排、子智能体管理。
- `src/plugins/` + `src/plugin-sdk/`：插件运行时与扩展 SDK 契约。
- `src/config/`、`src/infra/`、`src/process/`、`src/security/`、`src/secrets/`：基础设施层。

## 边界与依赖方向

推荐依赖方向：

1. `cli/commands` → 调用服务/领域模块，避免深层跨域 import。
2. `gateway` → 负责编排，业务逻辑下沉到子模块。
3. `channels` → 共享路由与策略，渠道差异保持隔离。
4. `plugin-sdk` → 对外稳定契约，避免泄露内部实现类型。

## 重构检查清单

- 协议/schema 兼容性（`gateway/protocol`）
- 渠道注册、别名、onboarding 覆盖
- 路由与会话连续性（`src/routing/*`）
- `extensions/*` 兼容性
- CLI 与文档同步（`src/cli`、`src/commands`、`docs/`）

## 相关文档

- [Gateway 网关架构](/concepts/architecture)
- [智能体系统设计](/concepts/agents-architecture)
