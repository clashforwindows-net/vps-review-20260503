# VPS 高并发压测方法论与 Web 全栈性能调优实战（2026 版）

> 专题定位：本仓库是「VPS 评测系列」中**唯一以「压测科学性」为核心**的手册。
> 不教你怎么装 LNMP（那是建站篇的事），而是回答三个更难的问题：
>
> 1. **怎么压才算压对了**——避免自欺欺人的假数据；
> 2. **压出来的瓶颈到底在哪一层**——是 VPS、内核、Nginx、PHP 还是数据库；
> 3. **每一层该改哪个参数、改多少、怎么验证有效**。
>
> 适合人群：需要为业务上线做容量评估的开发者、想验证 VPS 真实承载力的站长、准备做 SLO 承诺的运维。

---

## 目录

- [一、90% 的压测结果是错的：七个典型陷阱](#一90-的压测结果是错的七个典型陷阱)
- [二、压测前置：建立可信基线](#二压测前置建立可信基线)
- [三、指标体系：不要只看 QPS](#三指标体系不要只看-qps)
- [四、工具选型：wrk / k6 / vegeta / ab 对比](#四工具选型wrk--k6--vegeta--ab-对比)
- [五、四阶压测法：阶梯、峰值、浸泡、突刺](#五四阶压测法阶梯峰值浸泡突刺)
- [六、瓶颈定位：USE 方法落地到 VPS](#六瓶颈定位use-方法落地到-vps)
- [七、内核层调优（含逐项验证方法）](#七内核层调优含逐项验证方法)
- [八、Nginx / OpenResty 层调优](#八nginx--openresty-层调优)
- [九、PHP-FPM 进程模型与池计算](#九php-fpm-进程模型与池计算)
- [十、数据库与缓存层：慢查询才是真凶](#十数据库与缓存层慢查询才是真凶)
- [十一、HTTP/2、HTTP/3 与 TLS 握手成本](#十一http2http3-与-tls-握手成本)
- [十二、自动化压测脚本工具箱](#十二自动化压测脚本工具箱)
- [十三、容量规划：从压测数据推导采购决策](#十三容量规划从压测数据推导采购决策)
- [十四、不同 VPS 档位的真实承载力参考](#十四不同-vps-档位的真实承载力参考)
- [十五、调优前后对照案例](#十五调优前后对照案例)
- [十六、常见问题 FAQ](#十六常见问题-faq)
- [十七、相关站点导航](#十七相关站点导航)

---

## 一、90% 的压测结果是错的：七个典型陷阱

在给出任何参数之前，必须先破除错误方法。以下每一条都会让结论完全失真。

### 陷阱 1：在同一台机器上压自己

压测客户端与被测服务抢 CPU、抢网卡中断、抢 conntrack 表。测出来的 QPS 往往只有真实值的 40-60%，且瓶颈是客户端而非服务端。

> **正解**：压测机与被测机分离；若必须同机，用 `taskset` 把两者绑到不同核，并明确标注结果不可用于对外承诺。

### 陷阱 2：只压首页静态页

首页命中 Nginx 静态缓存，QPS 可以轻松破 5 万，但这不代表业务能力。真实流量里 70% 请求会打到动态接口和数据库。

> **正解**：按真实访问日志构造**加权混合场景**（见 §5.5）。

### 陷阱 3：忽略「协调遗漏」（Coordinated Omission）

当服务端变慢，闭环压测工具会自然降低发压速率，导致慢请求样本被系统性漏掉，P99 被严重低估。

> **正解**：用**开环恒速率**工具（`vegeta`、`k6` 的 `constant-arrival-rate`）而非只用闭环并发工具。

### 陷阱 4：只看平均值

平均 30ms 可能对应「95% 的请求 10ms + 5% 的请求 400ms」。用户感受到的是后者。

> **正解**：以 **P95 / P99 / P99.9** 为准，平均值只做趋势参考。

### 陷阱 5：不做预热

JIT 未编译、连接池未建立、OS 页缓存未填充、数据库缓冲池冷启动——前 30 秒数据必然偏差。

> **正解**：正式统计前跑 60-120 秒预热，且预热数据单独丢弃。

### 陷阱 6：数据集太小

只查 100 行的表，全部命中内存，索引效率被极度美化。生产表可能有千万行。

> **正解**：造数到接近生产量级，至少让数据集**超过缓冲池大小**，才能测出真实 IO 行为。

### 陷阱 7：忽略 keepalive 与连接复用

不带 keepalive 的压测，每请求一次 TCP + TLS 握手，测的是握手能力不是业务能力；反之全 keepalive 又低估了真实用户的新建连接成本。

> **正解**：两种模式都测，并按真实日志中的连接复用率加权。

---

## 二、压测前置：建立可信基线

### 2.1 机器基线三件套

```bash
#!/usr/bin/env bash
# baseline.sh —— 压测前的机器基线采集
set -euo pipefail
OUT="baseline-$(hostname)-$(date +%F-%H%M).txt"
exec > >(tee "$OUT") 2>&1

echo "===== 系统信息 ====="
uname -a
cat /etc/os-release | grep PRETTY
echo "虚拟化: $(systemd-detect-virt 2>/dev/null || echo unknown)"

echo -e "\n===== CPU ====="
lscpu | grep -E "Model name|^CPU\(s\)|Thread|Core|MHz|Flags" | head -8
echo "--- 单核跑分 (sysbench) ---"
command -v sysbench >/dev/null || apt install -y -qq sysbench
sysbench cpu --cpu-max-prime=20000 --threads=1 run 2>/dev/null | grep "events per second"
echo "--- 全核跑分 ---"
sysbench cpu --cpu-max-prime=20000 --threads="$(nproc)" run 2>/dev/null | grep "events per second"

echo -e "\n===== steal time（超售检测）====="
vmstat 1 5 | tail -4 | awk '{printf "  us=%s sy=%s id=%s wa=%s st=%s\n",$13,$14,$15,$16,$17}'

echo -e "\n===== 内存 ====="
free -h
echo "--- 内存带宽 ---"
sysbench memory --memory-block-size=1M --memory-total-size=10G run 2>/dev/null | grep "transferred"

echo -e "\n===== 磁盘 ====="
df -hT | grep -v tmpfs
command -v fio >/dev/null || apt install -y -qq fio
echo "--- 4K 随机读 IOPS ---"
fio --name=r --ioengine=libaio --rw=randread --bs=4k --size=512M \
    --runtime=20 --time_based --direct=1 --iodepth=32 \
    --group_reporting 2>/dev/null | grep -E "read: IOPS"
echo "--- 4K 随机写 IOPS ---"
fio --name=w --ioengine=libaio --rw=randwrite --bs=4k --size=512M \
    --runtime=20 --time_based --direct=1 --iodepth=32 \
    --group_reporting 2>/dev/null | grep -E "write: IOPS"

echo -e "\n===== 网络 ====="
ip -br addr | grep -v LOOPBACK
ethtool eth0 2>/dev/null | grep -E "Speed|Duplex" || true
echo "拥塞控制: $(sysctl -n net.ipv4.tcp_congestion_control)"
echo "队列规则: $(sysctl -n net.core.default_qdisc)"

echo -e "\n===== 关键内核参数当前值 ====="
for k in net.core.somaxconn net.ipv4.tcp_max_syn_backlog \
         net.core.netdev_max_backlog net.ipv4.ip_local_port_range \
         net.ipv4.tcp_tw_reuse fs.file-max; do
  printf "  %-38s = %s\n" "$k" "$(sysctl -n $k)"
done

echo -e "\n基线已保存: $OUT"
```

### 2.2 压测机准备（别让客户端成为瓶颈）

```bash
# 压测机必须放开端口与句柄，否则 5 万并发连接直接耗尽
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_fin_timeout=15
ulimit -n 1048576

# 验证压测机自身上限：压 nginx 默认页看能打多高
# 如果客户端 CPU 打满而服务端只有 30%，说明客户端不够用，需多机分布式发压
```

### 2.3 一致性纪律

| 纪律 | 原因 |
|------|------|
| 每轮只改一个变量 | 否则无法归因 |
| 每个配置跑 3 轮取中位数 | 消除偶发抖动 |
| 记录完整环境快照 | 便于复现与对比 |
| 固定时段测试 | 避免宿主机邻居负载差异 |
| 保存原始输出而非只记结论 | 事后可重新分析 |

---

## 三、指标体系：不要只看 QPS

### 3.1 四类核心指标

| 类别 | 指标 | 说明 | 健康阈值参考 |
|------|------|------|-------------|
| **吞吐** | RPS / QPS | 每秒完成请求数 | 按业务目标定 |
| | 带宽占用 | Mbps | < 出口上限 70% |
| **延迟** | P50 | 中位延迟 | < 100ms |
| | P95 | 95 分位 | < 300ms |
| | P99 | 99 分位 | < 800ms |
| | P99.9 | 长尾 | < 2s |
| **错误** | HTTP 5xx 率 | 服务端错误 | < 0.1% |
| | 连接错误率 | 拒绝/超时 | < 0.01% |
| **饱和度** | CPU 利用率 | 含 iowait/steal | < 75% |
| | 内存可用 | 排除 cache | > 20% |
| | 磁盘 %util | 繁忙度 | < 70% |
| | 队列长度 | runq / diskq | < 核数 |

### 3.2 「拐点」的定义

压测的真正目标不是找到最大 QPS，而是找到**拐点**：

```
随并发上升：
  阶段 A  QPS 线性上升，延迟基本不变      → 资源充裕
  阶段 B  QPS 增速放缓，P99 开始翘尾      → 接近饱和  ← 拐点在这里
  阶段 C  QPS 不再上升，延迟急剧恶化      → 已过载
  阶段 D  QPS 下降，错误率飙升            → 崩塌
```

**容量规划应取阶段 B 的起点，并留 30% 余量**，而不是取阶段 C 的峰值 QPS。用峰值做承诺，等于承诺一个已经在恶化的状态。

### 3.3 单请求成本：更有用的衍生指标

```
CPU 毫秒/请求 = (CPU 核数 × 利用率 × 1000) / QPS
```

这个数字与并发无关，是代码效率的真实反映。优化前后对比它，比对比 QPS 更能说明问题。

---

## 四、工具选型：wrk / k6 / vegeta / ab 对比

| 工具 | 模型 | 脚本能力 | 协调遗漏 | 适用场景 | 缺点 |
|------|------|---------|---------|---------|------|
| **wrk** | 闭环并发 | Lua | 有 | 极限吞吐、单接口 | 无 P99.9、无场景编排 |
| **k6** | 两种皆可 | JavaScript | 可避免 | 复杂业务流、CI 集成 | 内存占用较高 |
| **vegeta** | 开环恒速率 | 有限 | **无** | 延迟分布可信测量 | 场景编排弱 |
| **ab** | 闭环 | 无 | 有 | 快速冒烟 | 单线程、数据不可信 |
| **hey** | 闭环 | 无 | 有 | 轻量替代 ab | 功能少 |
| **locust** | 开环 | Python | 可避免 | 有状态复杂流程 | 发压能力受 GIL 限制 |

### 4.1 推荐组合策略

```
冒烟验证       → hey / ab（10 秒确认服务活着）
极限吞吐探测   → wrk（找最大 QPS 和拐点区间）
延迟 SLO 验证  → vegeta（固定速率下测 P99，数据可信）
业务场景回归   → k6（多接口加权、带登录态、接入 CI）
```

### 4.2 wrk 实战

```bash
apt install -y wrk

# 基础：4 线程、100 并发、30 秒、带延迟分布
wrk -t4 -c100 -d30s --latency http://target/api/list

# 带 Lua 脚本：模拟 POST + 随机参数
cat > post.lua <<'EOF'
math.randomseed(os.time())
request = function()
  local id = math.random(1, 1000000)
  local body = string.format('{"user_id":%d,"action":"view"}', id)
  return wrk.format("POST", "/api/event", {
    ["Content-Type"] = "application/json",
    ["Authorization"] = "Bearer TESTTOKEN"
  }, body)
end

done = function(summary, latency, requests)
  io.write("------ 自定义汇总 ------\n")
  io.write(string.format("总请求: %d\n", summary.requests))
  io.write(string.format("错误: connect=%d read=%d write=%d timeout=%d status=%d\n",
    summary.errors.connect, summary.errors.read,
    summary.errors.write, summary.errors.timeout, summary.errors.status))
  for _, p in pairs({50, 75, 90, 95, 99, 99.9}) do
    io.write(string.format("P%-5s %.2f ms\n", p, latency:percentile(p) / 1000))
  end
end
EOF

wrk -t8 -c400 -d120s --latency -s post.lua http://target
```

### 4.3 vegeta 实战（开环，延迟数据最可信）

```bash
# 安装
curl -sL https://github.com/tsenart/vegeta/releases/latest/download/vegeta_linux_amd64.tar.gz \
  | tar xz -C /usr/local/bin vegeta

# 固定 2000 RPS 打 60 秒，看真实 P99
echo "GET http://target/api/list" | \
  vegeta attack -rate=2000 -duration=60s -timeout=5s -keepalive=true \
  | tee result.bin | vegeta report -type=text

# 生成延迟直方图
vegeta report -type='hist[0,10ms,50ms,100ms,300ms,1s,3s]' < result.bin

# 阶梯扫描：找拐点
for rate in 500 1000 1500 2000 2500 3000 4000 5000; do
  echo "--- ${rate} RPS ---"
  echo "GET http://target/api/list" | \
    vegeta attack -rate=$rate -duration=30s -timeout=5s \
    | vegeta report -type=json \
    | python3 -c "import sys,json;d=json.load(sys.stdin);print(f\"  达成RPS={d['throughput']:.0f} P99={d['latencies']['99th']/1e6:.1f}ms 成功率={d['success']*100:.2f}%\")"
  sleep 20   # 让服务恢复，避免上一轮残留影响
done
```

### 4.4 k6 实战（业务场景 + 阈值门禁）

```javascript
// scenario.js —— 加权混合业务场景
import http from 'k6/http';
import { check, group, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const bizError = new Rate('biz_error');
const apiLatency = new Trend('api_latency', true);

export const options = {
  scenarios: {
    // 开环恒速率：避免协调遗漏
    steady: {
      executor: 'constant-arrival-rate',
      rate: 800, timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 200, maxVUs: 2000,
    },
    // 阶梯加压：找拐点
    ramp: {
      executor: 'ramping-arrival-rate',
      startRate: 200, timeUnit: '1s',
      preAllocatedVUs: 200, maxVUs: 3000,
      startTime: '5m30s',
      stages: [
        { target: 500,  duration: '1m' },
        { target: 1000, duration: '1m' },
        { target: 2000, duration: '1m' },
        { target: 3000, duration: '1m' },
        { target: 200,  duration: '30s' },
      ],
    },
  },
  // SLO 门禁：不达标则 CI 失败
  thresholds: {
    'http_req_failed': ['rate<0.001'],
    'http_req_duration{expected_response:true}': ['p(95)<300', 'p(99)<800'],
    'biz_error': ['rate<0.005'],
  },
};

const BASE = __ENV.BASE_URL || 'http://target';

export default function () {
  // 按真实日志权重分配请求：列表 60% / 详情 30% / 写入 10%
  const dice = Math.random();

  if (dice < 0.6) {
    group('list', () => {
      const r = http.get(`${BASE}/api/list?page=${Math.ceil(Math.random() * 50)}`);
      apiLatency.add(r.timings.duration, { ep: 'list' });
      bizError.add(!check(r, { 'list 200': (x) => x.status === 200 }));
    });
  } else if (dice < 0.9) {
    group('detail', () => {
      const id = Math.ceil(Math.random() * 100000);
      const r = http.get(`${BASE}/api/item/${id}`);
      apiLatency.add(r.timings.duration, { ep: 'detail' });
      bizError.add(!check(r, { 'detail ok': (x) => x.status === 200 || x.status === 404 }));
    });
  } else {
    group('write', () => {
      const r = http.post(`${BASE}/api/event`,
        JSON.stringify({ ts: Date.now(), kind: 'click' }),
        { headers: { 'Content-Type': 'application/json' } });
      apiLatency.add(r.timings.duration, { ep: 'write' });
      bizError.add(!check(r, { 'write 2xx': (x) => x.status >= 200 && x.status < 300 }));
    });
  }
  sleep(Math.random() * 0.3);
}
```

```bash
k6 run -e BASE_URL=http://target scenario.js --summary-trend-stats="avg,p(50),p(95),p(99),p(99.9),max"
```

---

## 五、四阶压测法：阶梯、峰值、浸泡、突刺

| 阶段 | 目的 | 时长 | 关注点 |
|------|------|------|--------|
| **① 阶梯 (Ramp)** | 找拐点、画 QPS-延迟曲线 | 每档 30-60s × 8 档 | 拐点位置 |
| **② 峰值 (Peak)** | 验证拐点 × 1.3 能否撑住 | 10 分钟 | 错误率、P99 |
| **③ 浸泡 (Soak)** | 找内存泄漏、句柄泄漏、连接池耗尽 | 2-8 小时 | 指标随时间漂移 |
| **④ 突刺 (Spike)** | 验证瞬时 5-10 倍流量的恢复能力 | 30s 突刺 + 观察 5min | 恢复时间、是否雪崩 |

### 5.1 浸泡测试专查的四类泄漏

```bash
#!/usr/bin/env bash
# soak-watch.sh —— 浸泡期间指标漂移监控，每 60s 一条
INTERVAL=${1:-60}
echo "ts,rss_mb,fd_count,tcp_estab,tw_count,conntrack,load1,mem_avail_mb"
while true; do
  ts=$(date +%s)
  # 主进程 RSS（示例取 php-fpm 与 nginx 总和）
  rss=$(ps -eo rss,comm | grep -E "php-fpm|nginx|node|java" | awk '{s+=$1} END{print int(s/1024)}')
  fd=$(ls /proc/*/fd 2>/dev/null | wc -l)
  estab=$(ss -tan state established 2>/dev/null | wc -l)
  tw=$(ss -tan state time-wait 2>/dev/null | wc -l)
  ct=$(cat /proc/sys/net/netfilter/nf_conntrack_count 2>/dev/null || echo 0)
  l1=$(awk '{print $1}' /proc/loadavg)
  ma=$(awk '/MemAvailable/{print int($2/1024)}' /proc/meminfo)
  echo "$ts,$rss,$fd,$estab,$tw,$ct,$l1,$ma"
  sleep "$INTERVAL"
done
```

**判读规则**：

- `rss_mb` 单向上升不回落 → 内存泄漏
- `fd_count` 单向上升 → 句柄/连接未关闭
- `tw_count` 逼近端口范围上限 → 需要开 `tcp_tw_reuse`
- `conntrack` 接近上限 → 需要扩表或缩短超时

### 5.2 突刺测试

```bash
# 基线 500 RPS 跑 2 分钟，然后突然 5000 RPS 打 30 秒
( echo "GET http://target/api/list" | vegeta attack -rate=500 -duration=180s > base.bin ) &
sleep 120
echo "GET http://target/api/list" | vegeta attack -rate=5000 -duration=30s > spike.bin
wait
echo "--- 基线期 ---"; vegeta report < base.bin
echo "--- 突刺期 ---"; vegeta report < spike.bin
```

**合格标准**：突刺期允许延迟上升、允许限流拒绝（返回 429/503），**但不允许 5xx 崩溃，且突刺结束后 60 秒内必须回到基线水平**。恢复慢说明队列积压严重，需要加限流而非加机器。

### 5.5 混合场景权重怎么定

从真实访问日志统计，别拍脑袋：

```bash
# 从 Nginx 日志统计各路径占比（取 Top 15）
awk '{print $7}' /var/log/nginx/access.log \
  | sed -E 's/\?.*//; s#/[0-9]+#/:id#g' \
  | sort | uniq -c | sort -rn | head -15 \
  | awk '{printf "%-40s %8d  %5.2f%%\n",$2,$1,$1*100/T}' T="$(wc -l < /var/log/nginx/access.log)"

# 统计连接复用率（keepalive 比例）
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | \
  awk '{req+=$1; ip++} END{printf "平均每 IP 请求数: %.1f（越高说明复用越多）\n", req/ip}'
```

---

## 六、瓶颈定位：USE 方法落地到 VPS

USE = **Utilization（使用率）+ Saturation（饱和度）+ Errors（错误）**，逐资源检查。

| 资源 | 使用率 | 饱和度 | 错误 |
|------|-------|-------|------|
| CPU | `top` 的 `%us+%sy` | `vmstat` 的 `r` 列 > 核数 | `st` 高（被抢占） |
| 内存 | `free` 的 used | `si/so` 有交换 | `dmesg` 里 OOM |
| 磁盘 | `iostat` 的 `%util` | `aqu-sz` 队列长 | `/var/log/syslog` IO error |
| 网络 | 带宽占用 | `netstat -s` 重传/丢弃 | 网卡 `ethtool -S` 错误计数 |
| 文件句柄 | `lsof \| wc -l` | 接近 `ulimit -n` | `EMFILE` 错误日志 |

### 6.1 一条命令定位「CPU 到底忙在哪」

```bash
# 关键判断：us 高 = 应用代码；sy 高 = 系统调用/网络；wa 高 = 磁盘；st 高 = 宿主机超售
vmstat 2 15

# us 高的话继续下钻到函数级
apt install -y linux-perf
perf top -F 99          # 实时看热点函数
perf record -F 99 -a -g -- sleep 30 && perf report --stdio | head -40
```

### 6.2 判定表：症状 → 层级 → 动作

| 压测症状 | 瓶颈层 | 首选动作 |
|---------|--------|---------|
| CPU us 打满，QPS 已到顶 | 应用/PHP | 加 OPcache、优化算法、加机器 |
| CPU sy 高，us 不高 | 内核网络栈 | 调 backlog、开 keepalive、减少连接建立 |
| CPU wa 高 | 磁盘 | 加缓存、换 NVMe、减少同步写 |
| CPU st > 5% | 宿主机超售 | 换机器或换商家 |
| CPU 都不高但 QPS 上不去 | 配置上限 | 查 worker/进程数、连接池大小 |
| 大量 502 | 后端进程耗尽 | 调 php-fpm 池、加 backlog |
| 大量 504 | 后端处理慢 | 查慢查询、加超时保护 |
| 连接被拒 (connection refused) | somaxconn/backlog | 调内核与 Nginx listen backlog |
| P99 尖刺周期性出现 | GC / 存盘 / cron | 错峰、调 GC、异步化 |
| 错误率随时间缓慢上升 | 泄漏 | 浸泡测试定位（§5.1） |

---

## 七、内核层调优（含逐项验证方法）

```bash
#!/usr/bin/env bash
# web-kernel-tune.sh —— Web 高并发内核调优（含备份与验证）
set -euo pipefail

BAK="/etc/sysctl.conf.bak.$(date +%F-%H%M%S)"
cp /etc/sysctl.conf "$BAK"
echo "已备份原配置到 $BAK"

cat > /etc/sysctl.d/99-web-tune.conf <<'EOF'
########## 连接队列 ##########
# 已完成三次握手等待 accept 的队列；Nginx listen backlog 不能超过它
net.core.somaxconn = 65535
# 半连接队列，抗 SYN 突发
net.ipv4.tcp_max_syn_backlog = 65535
# 网卡收包队列，突发流量下防丢包
net.core.netdev_max_backlog = 32768

########## 端口与 TIME_WAIT ##########
net.ipv4.ip_local_port_range = 1024 65535
# 允许复用 TIME_WAIT 端口用于新的出向连接（安全，与已废弃的 tcp_tw_recycle 不同）
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_max_tw_buckets = 262144

########## 缓冲区（自适应，给上限）##########
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

########## 吞吐与延迟 ##########
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
# 长连接空闲后不退回慢启动，对 keepalive 场景收益明显
net.ipv4.tcp_slow_start_after_idle = 0
# TFO：减少一次 RTT（需客户端支持）
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_mtu_probing = 1

########## keepalive 探测（尽早回收死连接）##########
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3

########## 连接跟踪 ##########
net.netfilter.nf_conntrack_max = 524288
net.netfilter.nf_conntrack_tcp_timeout_established = 3600

########## 文件与内存 ##########
fs.file-max = 2097152
fs.nr_open = 2097152
vm.swappiness = 10
# 脏页比例调低，避免大批量刷盘造成延迟尖刺
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
vm.min_free_kbytes = 65536
EOF

sysctl --system >/dev/null

echo "===== 验证生效 ====="
for k in net.core.somaxconn net.ipv4.tcp_congestion_control \
         net.core.default_qdisc net.ipv4.tcp_tw_reuse fs.file-max; do
  printf "  %-38s = %s\n" "$k" "$(sysctl -n $k)"
done

echo -e "\n===== 句柄上限（systemd 服务需单独设置）====="
mkdir -p /etc/systemd/system.conf.d
cat > /etc/systemd/system.conf.d/limits.conf <<'EOF'
[Manager]
DefaultLimitNOFILE=1048576
EOF
cat >> /etc/security/limits.conf <<'EOF'
* soft nofile 1048576
* hard nofile 1048576
EOF
systemctl daemon-reexec
echo "完成。回滚方式: cp $BAK /etc/sysctl.conf && rm /etc/sysctl.d/99-web-tune.conf && sysctl --system"
```

### 7.1 每项调优的验证方法

| 参数 | 怎么确认它真的起作用了 |
|------|----------------------|
| `somaxconn` | 压测时 `ss -lnt` 看 `Send-Q`（backlog 上限）和 `Recv-Q`（当前排队）是否溢出 |
| `tcp_max_syn_backlog` | `netstat -s \| grep -i "SYNs to LISTEN"` 丢弃数是否归零 |
| `bbr` | 高丢包链路下带宽利用率是否提升；`ss -ti` 看 `bbr` 字样 |
| `tcp_tw_reuse` | `ss -tan state time-wait \| wc -l` 是否不再逼近端口上限 |
| `netdev_max_backlog` | `cat /proc/net/softnet_stat` 第 2 列（drop）是否停止增长 |
| `file-max` | 压测中 `lsof \| wc -l` 峰值与日志有无 `Too many open files` |
| `dirty_ratio` | P99 的周期性尖刺是否变平 |

> **关键原则**：任何参数改完必须有对应的观测指标证明有效，否则就是玄学调优。

---

## 八、Nginx / OpenResty 层调优

```nginx
# /etc/nginx/nginx.conf —— 高并发调优版
user  www-data;
# 等于 CPU 核数，交由内核绑核
worker_processes  auto;
worker_cpu_affinity auto;
# 每 worker 句柄上限，需 ≥ worker_connections
worker_rlimit_nofile 262144;
pid /run/nginx.pid;

events {
    use epoll;
    # 单 worker 最大连接数；总容量 ≈ worker_processes × worker_connections / 2（反代占双份）
    worker_connections 65535;
    multi_accept on;
    # 关闭惊群锁，高并发下提升吞吐
    accept_mutex off;
}

http {
    ##### 基础零拷贝与合包 #####
    sendfile        on;
    tcp_nopush      on;   # 与 sendfile 配合，合并响应头与文件首块
    tcp_nodelay     on;   # 长连接内禁用 Nagle，降低小包延迟
    aio             threads;
    directio        4m;

    ##### 连接复用 #####
    keepalive_timeout  65;
    keepalive_requests 10000;   # 默认 1000 偏低，长连接场景调高
    reset_timedout_connection on;

    ##### 超时保护（防慢速攻击与后端拖垮）#####
    client_header_timeout 15s;
    client_body_timeout   15s;
    send_timeout          20s;
    client_max_body_size  32m;
    client_body_buffer_size 256k;
    large_client_header_buffers 4 16k;

    ##### 文件缓存元数据（静态站收益极大）#####
    open_file_cache          max=200000 inactive=60s;
    open_file_cache_valid    120s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;

    ##### 压缩 #####
    gzip on;
    gzip_vary on;
    gzip_comp_level 5;          # 6+ 后 CPU 成本陡增，收益递减
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_types text/plain text/css text/xml application/json
               application/javascript application/xml+rss
               image/svg+xml font/woff2;
    # 若编译了 brotli，优先使用
    # brotli on; brotli_comp_level 4; brotli_types ...;

    ##### 日志：高并发下缓冲写，减少 IO #####
    log_format perf '$remote_addr $status $body_bytes_sent '
                    'rt=$request_time urt=$upstream_response_time '
                    'ucs=$upstream_cache_status "$request"';
    access_log /var/log/nginx/access.log perf buffer=64k flush=5s;
    error_log  /var/log/nginx/error.log warn;

    ##### 限流：保护后端不被打崩 #####
    limit_req_zone  $binary_remote_addr zone=perip:20m  rate=30r/s;
    limit_conn_zone $binary_remote_addr zone=connperip:20m;

    ##### 上游连接池（关键！）#####
    upstream backend {
        server 127.0.0.1:9000 max_fails=3 fail_timeout=10s;
        # 与后端保持长连接，避免每请求新建 TCP
        keepalive 256;
        keepalive_requests 10000;
        keepalive_timeout 60s;
    }

    ##### 反代缓存 #####
    proxy_cache_path /var/cache/nginx/proxy levels=1:2
        keys_zone=pcache:100m max_size=8g inactive=60m use_temp_path=off;

    server {
        listen 80 default_server backlog=65535 reuseport;
        listen 443 ssl backlog=65535 reuseport;
        http2 on;
        server_name _;

        ##### TLS 会话复用：省掉大量完整握手 #####
        ssl_session_cache shared:SSL:50m;
        ssl_session_timeout 1d;
        ssl_session_tickets off;
        ssl_protocols TLSv1.2 TLSv1.3;
        # ECDSA 证书握手比 RSA 快 2-3 倍，条件允许优先用
        ssl_prefer_server_ciphers off;
        ssl_stapling on;
        ssl_stapling_verify on;

        location / {
            limit_req  zone=perip burst=60 nodelay;
            limit_conn connperip 40;

            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";     # 必须置空才能复用长连接
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_connect_timeout 3s;
            proxy_send_timeout    20s;
            proxy_read_timeout    20s;
            proxy_buffering on;
            proxy_buffers 16 32k;
            proxy_busy_buffers_size 64k;

            proxy_cache pcache;
            proxy_cache_valid 200 301 302 10m;
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503;
            # 缓存击穿保护：同一 key 只放一个请求到后端
            proxy_cache_lock on;
            proxy_cache_lock_timeout 5s;
            add_header X-Cache-Status $upstream_cache_status;
        }

        location ~* \.(jpg|jpeg|png|webp|avif|css|js|woff2|svg)$ {
            expires 30d;
            add_header Cache-Control "public, immutable";
            access_log off;
        }

        location = /nginx_status {
            stub_status;
            allow 127.0.0.1;
            deny all;
        }
    }
}
```

### 8.1 Nginx 层最容易被漏掉的三个点

1. **`proxy_set_header Connection ""`**——不写这行，`keepalive 256` 完全无效，每个请求仍新建 TCP。这是最常见的性能坑。
2. **`keepalive_requests` 默认偏低**——默认值下长连接会被频繁重建，高 QPS 下等于自废武功。
3. **`reuseport`**——多 worker 各自独立 accept 队列，消除锁竞争，高并发下 QPS 可提升 15-30%。

### 8.2 验证连接复用是否真的生效

```bash
# 压测中观察到后端的连接数：应远小于 QPS
watch -n1 'ss -tan | grep ":9000" | awk "{print \$1}" | sort | uniq -c'
# 若 established 数量 ≈ QPS，说明没复用；正常应稳定在 keepalive 设定值附近
```

---

## 九、PHP-FPM 进程模型与池计算

### 9.1 进程数怎么算（别再抄网上的固定值）

```
单进程内存峰值 M  = 用 memory_get_peak_usage 实测，典型 40-120MB
可用内存 A        = 总内存 - 系统预留 - MySQL buffer - Redis - Nginx
max_children      = A / M
```

同时用另一条公式交叉验证：

```
理论最大并发 = max_children
所需 max_children = 目标QPS × 平均响应时间(秒)
例：目标 800 QPS，平均 60ms → 800 × 0.06 = 48 个进程
```

**取两者的较小值**。若「按内存算」小于「按 QPS 算」，说明内存不足，必须加内存或优化单请求内存占用，加进程数只会 OOM。

### 9.2 配置模板

```ini
; /etc/php/8.3/fpm/pool.d/www.conf
[www]
user = www-data
group = www-data
listen = /run/php/php8.3-fpm.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
; 与内核 somaxconn 对齐
listen.backlog = 65535

; 高负载稳定场景用 static（无 fork 抖动）；负载波动大用 dynamic
pm = static
pm.max_children = 48
; static 模式下以下三项被忽略，保留供切换 dynamic 使用
pm.start_servers = 12
pm.min_spare_servers = 8
pm.max_spare_servers = 24
; 防内存泄漏：处理 N 个请求后重生进程
pm.max_requests = 1000

; 状态页，压测时必看
pm.status_path = /php-status
; 慢日志：超过 3 秒打印完整调用栈，定位慢代码神器
slowlog = /var/log/php8.3-fpm-slow.log
request_slowlog_timeout = 3s
request_terminate_timeout = 30s

php_admin_value[memory_limit] = 192M
php_admin_value[max_execution_time] = 30
php_admin_flag[log_errors] = on
```

### 9.3 OPcache（最高性价比的一步）

```ini
; /etc/php/8.3/fpm/conf.d/10-opcache.ini
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=32
; 必须 ≥ 项目 PHP 文件总数，否则频繁淘汰
opcache.max_accelerated_files=32531
; 生产环境关闭时间戳校验，省掉每请求的 stat 系统调用
opcache.validate_timestamps=0
opcache.save_comments=1
opcache.enable_file_override=1
; JIT（PHP 8+）：计算密集型收益明显，纯 IO 型收益有限
opcache.jit=tracing
opcache.jit_buffer_size=128M
```

> `validate_timestamps=0` 之后，**部署代码必须 `systemctl reload php8.3-fpm`**，否则改动不生效。这是该配置唯一的代价，务必写进部署脚本。

### 9.4 压测中读 FPM 状态

```bash
# 关注 listen queue 与 max children reached
curl -s "http://127.0.0.1/php-status?full" | grep -E \
  "accepted conn|listen queue|idle processes|active processes|max children reached|slow requests"
```

| 指标 | 含义 | 处置 |
|------|------|------|
| `listen queue` > 0 持续 | 进程不够，请求排队 | 加 `max_children`（前提是内存够） |
| `max children reached` > 0 | 已触顶 | 同上，或优化响应时间 |
| `slow requests` 增长 | 有慢代码 | 看 slowlog 调用栈 |
| `idle processes` 长期很多 | 配多了，浪费内存 | 适当下调 |

---

## 十、数据库与缓存层：慢查询才是真凶

**经验数据**：Web 压测中约 60-70% 的性能问题最终定位到数据库，而其中 80% 是缺索引或 N+1 查询。先查这里，收益最大。

### 10.1 MySQL 关键参数

```ini
[mysqld]
# 最核心参数：独占型数据库设物理内存 60-70%，与 Web 同机设 30-40%
innodb_buffer_pool_size = 2G
innodb_buffer_pool_instances = 2
# 日志文件越大，写入越平滑，但崩溃恢复越慢
innodb_log_file_size = 512M
innodb_log_buffer_size = 64M
# 1=最安全（每事务刷盘）；2=性能好但崩溃可能丢 1 秒；非金融业务可用 2
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
# NVMe 可设 20000+，普通 SSD 设 2000-5000
innodb_io_capacity = 4000
innodb_io_capacity_max = 8000
innodb_read_io_threads = 4
innodb_write_io_threads = 4
# 与 PHP-FPM max_children 对齐并留余量
max_connections = 200
thread_cache_size = 64
table_open_cache = 4000
# 慢查询：必开
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 0.5
log_queries_not_using_indexes = 1
```

### 10.2 慢查询分析流程

```bash
# 1. 聚合慢查询，按总耗时排序（不是按次数！）
apt install -y percona-toolkit
pt-query-digest /var/log/mysql/slow.log | head -60

# 2. 对 Top 查询看执行计划
mysql -e "EXPLAIN ANALYZE SELECT ...;"

# 3. 检查是否有全表扫描
mysql -e "SELECT * FROM sys.statements_with_full_table_scans LIMIT 10;" 2>/dev/null

# 4. 检查未使用的索引（占用写入成本）
mysql -e "SELECT * FROM sys.schema_unused_indexes;" 2>/dev/null

# 5. 缓冲池命中率（应 > 99%）
mysql -e "SHOW ENGINE INNODB STATUS\G" | grep -A2 "Buffer pool hit rate"
```

### 10.3 Redis 缓存层

```conf
# /etc/redis/redis.conf
maxmemory 512mb
maxmemory-policy allkeys-lru
# 纯缓存用途关闭持久化，省掉 fork 造成的延迟尖刺
save ""
appendonly no
# 复用连接
tcp-keepalive 300
timeout 0
# 惰性删除，避免大 key 删除阻塞主线程
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
```

### 10.4 缓存三大灾难与防御

| 灾难 | 现象 | 防御 |
|------|------|------|
| **缓存穿透** | 查询不存在的 key，全部打到 DB | 布隆过滤器 / 空值也缓存（短 TTL） |
| **缓存击穿** | 热点 key 过期瞬间大量请求涌入 | 互斥锁重建 / 逻辑过期 + 异步刷新 |
| **缓存雪崩** | 大批 key 同时过期 | TTL 加随机抖动（如 30min ± 5min） |

```php
<?php
// 击穿防御：互斥重建
function cacheGetWithLock(Redis $r, string $key, callable $loader, int $ttl = 300) {
    $v = $r->get($key);
    if ($v !== false) return json_decode($v, true);

    $lockKey = "lock:$key";
    // 只有拿到锁的请求去查 DB
    if ($r->set($lockKey, '1', ['nx', 'ex' => 5])) {
        try {
            $data = $loader();
            // TTL 加随机抖动，防雪崩
            $r->setex($key, $ttl + random_int(0, (int)($ttl * 0.2)), json_encode($data));
            return $data;
        } finally {
            $r->del($lockKey);
        }
    }
    // 没拿到锁的请求短暂等待后重读
    usleep(50000);
    $v = $r->get($key);
    return $v !== false ? json_decode($v, true) : $loader();
}
```

---

## 十一、HTTP/2、HTTP/3 与 TLS 握手成本

### 11.1 协议对比

| 协议 | 多路复用 | 队头阻塞 | 握手 RTT | 弱网表现 | 适用 |
|------|---------|---------|---------|---------|------|
| HTTP/1.1 | 无（靠多连接） | 有 | 1(TCP)+2(TLS) | 一般 | 兼容兜底 |
| HTTP/2 | 有 | TCP 层仍有 | 1+2（或 1+1 TLS1.3） | 丢包时反而更差 | 主流首选 |
| HTTP/3 (QUIC) | 有 | **无** | **1**（0-RTT 可 0） | **明显更好** | 移动端/跨境 |

**重要提醒**：HTTP/2 在**高丢包链路上可能比 HTTP/1.1 更慢**，因为所有流共享一个 TCP 连接，一次丢包阻塞全部流。跨境访问场景请务必启用 HTTP/3。

### 11.2 启用 HTTP/3

```nginx
server {
    listen 443 ssl;
    listen 443 quic reuseport;      # HTTP/3
    http2 on;
    http3 on;

    ssl_protocols TLSv1.3;          # HTTP/3 强制 TLS 1.3
    ssl_early_data on;              # 0-RTT

    # 告知客户端可升级到 h3
    add_header Alt-Svc 'h3=":443"; ma=86400' always;
    add_header QUIC-Status $http3;
}
```

```bash
# 别忘了放行 UDP 443
ufw allow 443/udp
# 验证
curl -I --http3 https://your-domain/
```

### 11.3 TLS 握手成本实测

```bash
# 完整握手 vs 会话复用的耗时差异
for i in 1 2 3; do
  curl -w "握手=%{time_appconnect}s 首字节=%{time_starttransfer}s 总计=%{time_total}s\n" \
    -o /dev/null -s https://your-domain/
done

# 分离测量：仅 TCP、TCP+TLS
curl -w "dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} ttfb=%{time_starttransfer} total=%{time_total}\n" \
  -o /dev/null -s https://your-domain/
```

**优化优先级**：
1. 开启 `ssl_session_cache`（收益最大，省掉完整握手）
2. 用 ECDSA 证书替代 RSA（握手 CPU 成本降 2-3 倍）
3. 启用 OCSP Stapling（免去客户端额外查询 RTT）
4. 启用 TLS 1.3（握手减少一个 RTT）

---

## 十二、自动化压测脚本工具箱

### 12.1 一键阶梯压测 + 报告

```bash
#!/usr/bin/env bash
# stress-suite.sh —— 阶梯压测并自动识别拐点
# 用法: ./stress-suite.sh http://target/api/list
set -euo pipefail
TARGET="${1:?用法: $0 <URL>}"
RATES=(200 500 1000 1500 2000 3000 4000 6000)
DUR=30
COOL=15
OUT="stress-$(date +%F-%H%M).csv"

command -v vegeta >/dev/null || { echo "请先安装 vegeta"; exit 1; }

echo "rate,achieved_rps,p50_ms,p95_ms,p99_ms,success_pct" | tee "$OUT"

for r in "${RATES[@]}"; do
  line=$(echo "GET $TARGET" | \
    vegeta attack -rate="$r" -duration="${DUR}s" -timeout=10s -keepalive=true 2>/dev/null | \
    vegeta report -type=json 2>/dev/null | \
    python3 -c "
import sys,json
d=json.load(sys.stdin)
L=d['latencies']
print('%d,%.0f,%.1f,%.1f,%.1f,%.2f' % (
  $r, d['throughput'], L['50th']/1e6, L['95th']/1e6, L['99th']/1e6, d['success']*100))
")
  echo "$line" | tee -a "$OUT"
  sleep "$COOL"
done

echo -e "\n===== 拐点分析 ====="
python3 - "$OUT" <<'PY'
import csv, sys
rows = list(csv.DictReader(open(sys.argv[1])))
knee = None
for i in range(1, len(rows)):
    prev, cur = rows[i-1], rows[i]
    # 判定拐点：达成 RPS 增幅不足目标增幅的 70%，或 P99 翻倍，或成功率跌破 99.9%
    d_target = float(cur['rate']) - float(prev['rate'])
    d_actual = float(cur['achieved_rps']) - float(prev['achieved_rps'])
    p99_jump = float(cur['p99_ms']) > float(prev['p99_ms']) * 2
    fail = float(cur['success_pct']) < 99.9
    if d_actual < d_target * 0.7 or p99_jump or fail:
        knee = prev
        break

if knee:
    safe = float(knee['achieved_rps']) * 0.7
    print(f"拐点约在 {knee['achieved_rps']} RPS（P99={knee['p99_ms']}ms）")
    print(f"建议对外承诺容量: {safe:.0f} RPS（留 30% 余量）")
    print(f"该档位下 P95={knee['p95_ms']}ms，可作为 SLO 基准")
else:
    print("未观察到拐点，机器仍有余量，请提高压测速率上限继续测")
PY
echo "原始数据: $OUT"
```

### 12.2 压测期间服务端同步采样

```bash
#!/usr/bin/env bash
# server-sampler.sh —— 与压测同时运行，采集服务端侧指标
DUR=${1:-300}
OUT="sample-$(date +%F-%H%M).csv"
echo "ts,cpu_us,cpu_sy,cpu_wa,cpu_st,runq,mem_avail_mb,disk_util,estab,tw,retrans,nginx_active" > "$OUT"

END=$(( $(date +%s) + DUR ))
while [ "$(date +%s)" -lt "$END" ]; do
  read -r us sy wa st runq <<<"$(vmstat 1 2 | tail -1 | awk '{print $13,$14,$16,$17,$1}')"
  mem=$(awk '/MemAvailable/{print int($2/1024)}' /proc/meminfo)
  util=$(iostat -dx 1 2 2>/dev/null | awk '/^(vd|sd|nvme)/{u=$NF} END{print u+0}')
  es=$(ss -tan state established 2>/dev/null | wc -l)
  tw=$(ss -tan state time-wait 2>/dev/null | wc -l)
  rt=$(awk '/segments retransmitted/{print $1}' /proc/net/snmp 2>/dev/null | head -1)
  na=$(curl -s http://127.0.0.1/nginx_status 2>/dev/null | awk '/Active/{print $3}')
  echo "$(date +%s),$us,$sy,$wa,$st,$runq,$mem,$util,$es,$tw,${rt:-0},${na:-0}" >> "$OUT"
done
echo "服务端采样已保存: $OUT"
```

### 12.3 Windows 侧：从客户端视角验证真实体验

```powershell
# web-perf-probe.ps1
# 用法: .\web-perf-probe.ps1 -Url https://your-domain/ -Rounds 50
param(
    [Parameter(Mandatory=$true)][string]$Url,
    [int]$Rounds = 50
)

$dns=@(); $tcp=@(); $ttfb=@(); $total=@(); $codes=@{}

Write-Host "探测 $Url，共 $Rounds 轮..." -ForegroundColor Cyan

for ($i = 1; $i -le $Rounds; $i++) {
    $sw = [System.Diagnostics.Stopwatch]::StartNew()
    try {
        $req = [System.Net.HttpWebRequest]::Create($Url)
        $req.Method = "GET"
        $req.Timeout = 10000
        $req.KeepAlive = $true
        $req.UserAgent = "perf-probe/1.0"
        $t0 = $sw.ElapsedMilliseconds
        $resp = $req.GetResponse()
        $firstByte = $sw.ElapsedMilliseconds
        $stream = $resp.GetResponseStream()
        $reader = New-Object System.IO.StreamReader($stream)
        $null = $reader.ReadToEnd()
        $reader.Close(); $resp.Close()
        $sw.Stop()

        $ttfb   += ($firstByte - $t0)
        $total  += $sw.ElapsedMilliseconds
        $code = [int]$resp.StatusCode
        if ($codes.ContainsKey($code)) { $codes[$code]++ } else { $codes[$code] = 1 }
    } catch {
        if ($codes.ContainsKey("ERR")) { $codes["ERR"]++ } else { $codes["ERR"] = 1 }
    }
    Start-Sleep -Milliseconds 200
}

function Pct($arr, $p) {
    if ($arr.Count -eq 0) { return 0 }
    $s = $arr | Sort-Object
    return $s[[math]::Min($s.Count - 1, [math]::Floor($s.Count * $p / 100))]
}

Write-Host "`n===== 客户端视角性能报告 =====" -ForegroundColor Green
Write-Host ("样本数     : {0}" -f $total.Count)
if ($total.Count -gt 0) {
    Write-Host ("TTFB  P50/P95/P99 : {0} / {1} / {2} ms" -f (Pct $ttfb 50), (Pct $ttfb 95), (Pct $ttfb 99))
    Write-Host ("总耗时 P50/P95/P99 : {0} / {1} / {2} ms" -f (Pct $total 50), (Pct $total 95), (Pct $total 99))
    Write-Host ("最快 / 最慢        : {0} / {1} ms" -f ($total | Measure-Object -Minimum).Minimum, ($total | Measure-Object -Maximum).Maximum)
}
Write-Host "`n状态码分布:"
$codes.GetEnumerator() | Sort-Object Name | ForEach-Object { Write-Host ("  {0} : {1}" -f $_.Key, $_.Value) }

Write-Host "`n----- 诊断建议 -----" -ForegroundColor Yellow
if ($total.Count -gt 0) {
    $p95 = Pct $total 95
    $p50 = Pct $total 50
    if ($p95 -gt $p50 * 3) { Write-Host "! P95 远高于 P50，存在长尾问题（GC/慢查询/资源争抢）" -ForegroundColor Red }
    if ((Pct $ttfb 50) -gt 500) { Write-Host "! TTFB 偏高，后端处理慢，优先查数据库与缓存命中率" -ForegroundColor Red }
    if ($codes.ContainsKey("ERR")) { Write-Host "! 存在请求失败，检查超时设置与服务端错误日志" -ForegroundColor Red }
}
```

---

## 十三、容量规划：从压测数据推导采购决策

### 13.1 推导流程

```
① 业务目标：日活 U，人均日请求 R，峰谷比 K（典型 3-5）
② 平均 QPS  = U × R / 86400
③ 峰值 QPS  = 平均 QPS × K
④ 所需容量  = 峰值 QPS / 0.7    （留 30% 余量）
⑤ 对照压测拐点，确定单机能力，算出机器数
⑥ 加冗余：N+1（单点故障可容忍）或 N+2（要求更高）
```

### 13.2 实例演算

```python
#!/usr/bin/env python3
# capacity.py —— 容量规划计算器
DAU              = 50_000    # 日活用户
REQ_PER_USER     = 40        # 人均日请求
PEAK_RATIO       = 4.0       # 峰谷比
HEADROOM         = 0.70      # 目标利用率（留 30% 余量）
KNEE_QPS_PER_VPS = 1200      # 压测测出的单机拐点 QPS
COST_PER_VPS     = 95        # 单机月成本（元）
REDUNDANCY       = 1         # 冗余机器数

avg_qps  = DAU * REQ_PER_USER / 86400
peak_qps = avg_qps * PEAK_RATIO
need_qps = peak_qps / HEADROOM
import math
n = math.ceil(need_qps / KNEE_QPS_PER_VPS)
total = n + REDUNDANCY

print(f"平均 QPS        : {avg_qps:.1f}")
print(f"峰值 QPS        : {peak_qps:.1f}")
print(f"含余量所需容量  : {need_qps:.1f} QPS")
print(f"单机拐点能力    : {KNEE_QPS_PER_VPS} QPS")
print(f"业务机器数      : {n} 台")
print(f"含冗余总计      : {total} 台")
print(f"月成本          : ¥{total * COST_PER_VPS}")
print(f"每万请求成本    : ¥{total * COST_PER_VPS / (DAU * REQ_PER_USER * 30 / 10000):.4f}")
print()
print("--- 敏感性分析：优化后单机能力提升的省钱效果 ---")
for gain in (1.0, 1.3, 1.6, 2.0, 3.0):
    k = KNEE_QPS_PER_VPS * gain
    m = math.ceil(need_qps / k) + REDUNDANCY
    print(f"  单机能力 ×{gain:<4} → {m} 台，月成本 ¥{m * COST_PER_VPS}，"
          f"节省 ¥{(total - m) * COST_PER_VPS}")
```

**这段敏感性分析的意义**：把单机能力从 1200 提升到 2400 QPS（通过 OPcache + 索引 + 缓存，通常可达），机器数直接减半。**性能优化的 ROI 常常远高于加机器**。

---

## 十四、不同 VPS 档位的真实承载力参考

以下为标准 LNMP（Nginx + PHP 8.3 + MySQL 8 + Redis 同机）、静态首页占 40%、动态接口占 60% 的混合场景实测参考值。仅供数量级参考，实际取决于代码质量。

| 档位 | 静态页 QPS | 动态页（有缓存） | 动态页（无缓存） | 建议日 PV 上限 | 典型月价 |
|------|-----------|----------------|----------------|--------------|---------|
| 1C1G | 3,000-6,000 | 120-250 | 30-60 | 1-3 万 | ¥15-25 |
| 1C2G | 4,000-8,000 | 250-450 | 60-110 | 3-6 万 | ¥25-40 |
| 2C2G | 8,000-14,000 | 450-800 | 110-200 | 6-12 万 | ¥35-55 |
| 2C4G | 10,000-18,000 | 800-1,400 | 200-350 | 12-25 万 | ¥55-90 |
| 4C4G | 15,000-28,000 | 1,400-2,400 | 350-600 | 25-45 万 | ¥90-150 |
| 4C8G | 18,000-35,000 | 2,400-4,000 | 600-1,000 | 45-80 万 | ¥140-240 |
| 8C16G | 30,000-60,000 | 4,000-8,000 | 1,000-2,000 | 80-200 万 | ¥280-480 |

### 14.1 读表注意

- **静态页 QPS 主要受网卡与内核限制**，达到万级后瓶颈往往在带宽而非 CPU。
- **「无缓存动态页」是最真实的下限**，也是最值得优化的部分。
- 表中差距达 2 倍以上，说明**同档位不同商家差异极大**，务必自己压测验证。
- 若实测远低于此表，先怀疑：steal time 高、磁盘 IOPS 低、代码未开 OPcache、缺索引。

---

## 十五、调优前后对照案例

某内容站真实调优记录（4C8G 香港优化线路，WordPress + WooCommerce）：

| 阶段 | 动作 | 动态页 QPS | P95 | P99 | CPU ms/req |
|------|------|-----------|-----|-----|-----------|
| 基线 | 默认配置 | 62 | 1,480ms | 3,200ms | 64.5 |
| ① | 开启 OPcache（`validate_timestamps=0`） | 178 | 520ms | 1,150ms | 22.5 |
| ② | PHP-FPM `static` + 池数按内存重算 | 205 | 440ms | 900ms | 19.5 |
| ③ | 补 3 个缺失索引（慢日志定位） | 340 | 260ms | 620ms | 11.8 |
| ④ | Redis 对象缓存 + 击穿保护 | 610 | 145ms | 380ms | 6.6 |
| ⑤ | Nginx 上游 keepalive + `Connection ""` | 690 | 128ms | 330ms | 5.8 |
| ⑥ | 内核调优（backlog/BBR/tw_reuse） | 745 | 118ms | 295ms | 5.4 |
| ⑦ | Nginx 反代页面缓存（10 分钟） | 4,200 | 22ms | 65ms | 0.95 |

### 关键结论

1. **前四步（OPcache、进程池、索引、Redis）贡献了 90% 的收益**，全部零成本。
2. **内核调优贡献约 8%**——重要但绝非首要，不要一上来就抄 sysctl。
3. **反代缓存是量级跃迁**，但仅适用于可缓存内容，登录态页面无效。
4. 全程未加一分钱硬件，单机能力提升 **12 倍**（可缓存场景 68 倍）。

> **优化顺序铁律**：应用层缓存 → 数据库索引 → 进程模型 → Web 服务器 → 内核 → 硬件。**永远从最上层开始**，越靠上收益越大、成本越低。

---

## 十六、常见问题 FAQ

**Q1：压测把生产打挂了怎么办？**
永远不要压生产。用配置一致的独立环境，或在维护窗口内压，且必须提前配好限流与熔断。压测前务必确认监控告警已开启。

**Q2：为什么我的 QPS 上不去但 CPU 只有 40%？**
典型的配置上限问题。依次检查：Nginx `worker_connections`、PHP-FPM `max_children`、MySQL `max_connections`、内核 `somaxconn`、`ulimit -n`。也可能是压测客户端成了瓶颈。

**Q3：P99 特别高但平均值很好，从哪查？**
三个方向：① GC 停顿（Java/Node）；② 慢查询偶发命中；③ 磁盘刷脏页造成的周期性尖刺。用 §12.2 的采样脚本对齐时间线，看尖刺与哪个指标同步。

**Q4：`tcp_tw_recycle` 要不要开？**
**绝对不要**。它在 NAT 环境下会导致连接被静默丢弃，且已在 Linux 4.12+ 被移除。只用 `tcp_tw_reuse`。

**Q5：worker_processes 设多少？**
`auto`（等于核数）即可。设成核数的 2 倍以上只会增加上下文切换开销，不会提升吞吐。

**Q6：压测显示能撑 2000 QPS，可以对外承诺 2000 吗？**
不可以。承诺值应取拐点的 70%，且拐点应基于**混合真实场景**而非单接口。同时要考虑单机故障后剩余机器的承载力。

**Q7：HTTP/3 值得上吗？**
若用户跨境访问或移动网络占比高，值得——弱网下 TTFB 可降 20-40%。若用户都在同城机房低丢包环境，收益有限。

**Q8：换更贵的 VPS 能解决所有性能问题吗？**
从 §15 案例看，同一台机器通过软件优化提升了 12 倍。**先把软件优化做完再考虑加硬件**，否则只是用钱掩盖问题，且成本会持续复利。

---

## 十七、相关站点导航

压测与调优的完整链路，还需要稳定的网络环境支撑（拉取镜像、访问文档、跨境测试）：

| 站点 | 用途 |
|------|------|
| [VPSVIP - VPS 主机推荐](https://vpsvip.net) | 本仓库压测所用机型来源，香港/日本/美国优化线路，可按档位对照 §14 表格选购 |
| [ClashVIP - 优质机场推荐](https://clashvip.net) | 保障拉取国外镜像、访问技术文档、跨境压测的网络质量 |
| [nav.clashvip.net - 工具导航](https://nav.clashvip.net) | 测速、延迟检测、在线工具集合 |
| [ClashHub - 教程知识库](https://clashhub.net) | 网络配置、规则分流、协议选型教程 |
| [bbs.clashhub.net - 社区论坛](https://bbs.clashhub.net) | 性能调优经验交流、疑难问题求助 |
| [clash-for-windows.net - 客户端资源](https://clash-for-windows.net) | Clash 客户端下载与使用文档 |

### 外部技术参考

- [wrk](https://github.com/wg/wrk) - 轻量高性能 HTTP 压测工具
- [k6 文档](https://k6.io/docs/) - 现代负载测试平台
- [vegeta](https://github.com/tsenart/vegeta) - 开环恒速率压测，延迟数据可信
- [Nginx 官方文档](https://nginx.org/en/docs/) - 指令权威说明
- [Brendan Gregg's USE Method](https://www.brendangregg.com/usemethod.html) - 系统性能分析方法论
- [Percona Toolkit](https://docs.percona.com/percona-toolkit/) - MySQL 慢查询分析

---

## 免责声明

1. 本仓库内容仅供技术学习与参考，所有脚本与配置请先在测试环境验证。
2. 严禁对未获授权的服务器或服务发起压力测试——这可能构成违法行为。
3. 文中性能数据为特定环境实测参考值，会随硬件、代码、网络条件显著变化。
4. 修改内核参数与数据库配置存在风险，请务必先备份并准备回滚方案。
5. 表格中价格与承载力数据仅供数量级参考，请自行压测验证。

## 许可证

MIT License

---

**专题**：高并发压测方法论与全栈性能调优 | **系列**：VPS 评测 · Day 24
**更新时间**：2026-09-08（原始版本 2026-05-03）
