# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

NavidromeSquare 是 Navidrome（开源音乐流媒体服务器）的配置文件和部署工具集合。项目目标是简化 Navidrome 的部署和配置过程。

## 项目结构

- `client/` - 客户端相关代码（当前为空）
- `server/` - 服务器相关代码
  - `server/deploy/` - Docker 部署配置
- `doc/` - 文档
  - `doc/install/` - 安装说明文档
- `readme.md` - 项目说明

## 开发环境

由于项目还在初期阶段，暂无标准的构建命令。后续添加代码时应遵循：

- 客户端（如需要）：可考虑使用 Node.js + TypeScript/React
- 服务器（如需要）：可考虑使用 Go（与 Navidrome 保持一致）

## 部署说明

不使用 `docker-compose` 命令，改用 `docker compose` 命令：

```bash
# 启动服务
docker compose up -d

# 停止服务
docker compose down

# 查看日志
docker compose logs -f
```

## 重要配置信息

- 注意检查代码细节后再回答问题
- 暂无 COMMENT 语法需求（仅当使用 PostgreSQL 时使用 COMMENT ON 语句）
- 暂无 JSONB 处理需求（如需要时使用 JsonbTypeHandler）

## 相关链接

- Navidrome 官方文档：https://www.navidrome.org/docs/installation/docker/
