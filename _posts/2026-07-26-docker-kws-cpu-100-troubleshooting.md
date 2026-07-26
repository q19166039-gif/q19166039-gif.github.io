---
title: "Docker 容器 CPU 100% 排查实录：Open-XiaoAI KWS 模块的音频回环"
date: 2026-07-26 00:00:00 +0800
categories: [运维, Docker]
tags: [docker, cpu, troubleshooting, kws, linux, container]
---

## 问题现象

宿主机 `top` 看到一个 root 进程 `python main.py` 占用了 **110% CPU**（单核跑满 + 一点），已经持续运行了 17 小时。机器整体空闲 85.9%，没有内存压力，但这个进程吃满了一个核。

```
  PID USER      PR  NI    VIRT    RES    SHR    %CPU  %MEM     TIME+ COMMAND
425868 root      20   0 4519152   2.8g  61060 S 110.0  17.8 147:14.59 python
```

## 第一步：定位进程来源

通过 `/proc` 查看进程的 cgroup 和 parent PID：

```bash
cat /proc/425868/cgroup
# 输出: 0::/system.slice/docker-d7f6c5effdb9...scope
```

确认这是 **Docker 容器内的进程**。检查容器信息：

```bash
docker ps --filter id=d7f6c5effdb9
```

- 容器名：`open-xiaoai-bridge`
- 镜像：`ghcr.io/coderzc/open-xiaoai-bridge:latest`
- 端口映射：4399, 9092

这是一个小爱音箱接入 AI 后端的桥接服务（Open-XiaoAI Bridge）。

## 第二步：查看容器启动命令

```bash
docker inspect d7f6c5effdb9 --format '{{.Config.Cmd}}'
```

启动命令非常关键：

```bash
source /app/.venv/bin/activate
&& OPENCLAW_VAL="${OPENCLAW_ENABLE:-${OPENCLAW_ENABLED:-}}"
&& if [[ "${XIAOZHI_ENABLE:-}" =~ ^(1|true|yes)$ ]]
   || [[ "$OPENCLAW_VAL" =~ ^(1|true|yes)$ ]]
   || [[ "${OPENAI_ENABLE:-}" =~ ^(1|true|yes)$ ]]
then python core/services/audio/kws/keywords.py
fi
&& python main.py
```

注意到启动脚本中有一个**条件判断**：如果启用了 `XIAOZHI_ENABLE` / `OPENCLAW_ENABLE` / `OPENAI_ENABLE` 中的任何一个，就会先跑 `keywords.py`（关键词唤醒模块，KWS）。

## 第三步：看容器日志

```bash
docker logs d7f6c5effdb9 --tail 80
```

日志反复出现以下模式：

```
[KWS] 检测到语音，开始 KWS 检测
[KWS] 检测到持续静音（512ms），暂停 KWS，本次 KWS 监听时长 564ms
[KWS] 检测到语音，开始 KWS 检测
[KWS] 检测到持续静音（512ms），暂停 KWS，本次 KWS 监听时长 565ms
[KWS] 检测到语音，开始 KWS 检测
[KWS] 检测到持续静音（512ms），暂停 KWS，本次 KWS 监听时长 410ms
```

**循环极快**——最短的一次仅间隔 **410ms**，说明 KWS 模块在持续处理音频输入：

1. 检测到声音 → 开始唤醒词检测
2. 持续静音 512ms → 暂停
3. 立即又检测到声音 → 再次开始
4. 无限循环...

## 第四步：确认 CPU 占用

```bash
docker stats d7f6c5effdb9 --no-stream
```

```
CPU %     MEM USAGE / LIMIT     MEM %
102.41%   2.702GiB / 15.62GiB   17.30%
```

确认容器确实占用 **102.41% CPU**（约 1 个核跑满）。

## 根因分析

KWS 模块持续跑满 CPU，有几种可能：

### 1. ⚠️ 音频回环（最可能）

小爱音箱的**喇叭声音被麦克风拾取**，形成死循环：
- 音箱播放 AI 回复 → 麦克风听到声音 → KWS 触发开始检测
- 播放结束 → 512ms 静音 → 暂停 KWS
- 但隔壁环境还在持续播放或者有环境噪声 → 又被触发
- 最短 410ms 的循环周期，与语音片段播放时长吻合

这就是典型的**回环（loopback）**问题——AI 的回复被自己当成"用户说话"来处理了。

### 2. 🎯 正常行为

如果用户确实需要一直开着唤醒词监听，单核满载就是预期行为——CPU 跑 KWS 语音模型就是这样。

### 3. ⚙️ 环境变量误开启

`XIAOZHI_ENABLE` / `OPENCLAW_ENABLE` 等环境变量被设为 `1` 或 `true`，但本意可能是想关掉而不是开启。

## 解决方案

根据实际需求选择：

### 方案 A：关闭唤醒词（推荐，如果不需要语音唤醒）

```bash
# 停止容器
docker compose down
# 在 docker-compose.yml 或环境变量中将以下变量设为空
# XIAOZHI_ENABLE=
# OPENCLAW_ENABLE=
# OPENAI_ENABLE=
# 重启
docker compose up -d
```

### 方案 B：确认环境变量

```bash
docker inspect d7f6c5effdb9 --format '{{json .Config.Env}}'
```

查看哪些变量被设置为 `1`/`true`/`yes`。

### 方案 C：验证回环假设

暂时关闭音箱的播放输出，观察 CPU 是否下降：
```bash
# 如果 CPU 立刻下降，就是回环
# 如果依然跑满，可能是正常监听行为
```

### 方案 D：容器内无 ps/top 命令的处理

这个 Alpine 基础镜像没有预装 `ps` 和 `top`，遇到此类精简镜像时可以用以下方式调试：

```bash
# 直接用 docker stats 看资源
docker stats <container> --no-stream

# 看 /proc 下的信息
docker exec <container> cat /proc/1/status

# 或者临时安装
docker exec <container> apk add procps
```

## 总结

这次排查的启示：

1. **Docker 容器 CPU 100% 先看启动命令**——`docker inspect` 看 Cmd 和 Env 往往直接揭示根因
2. **日志模式比数值更有价值**——KWS 日志中 410ms 的循环周期比 CPU 百分比更能说明问题
3. **音频回环是 IoT/智能音箱开发中的常见坑**——麦克风拾取了扬声器的输出，形成自我触发的死循环
4. **精简容器没有 ps/top 很正常**——用 `docker stats` 和 `/proc` 接口一样能拿到足够的信息

如果你的 Docker 容器 CPU 异常跑满，别急着重启——先看看日志里有没有快速循环的模式，再检查启动命令中的条件判断，往往比盲目加资源更有效。
