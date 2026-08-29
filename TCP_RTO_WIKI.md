# Linux 内核 TCP 超时重传时间（RTO）选择机制实现分析

> 基于 Linux 7.1.0-rc7（Baby Opossum Posse）内核源码进行分析

## 一、整体架构

TCP RTO（Retransmission TimeOut）选择机制由以下核心模块组成：

```
┌─────────────────────────────────────────────────────────────┐
│                     RTO 选择总体流程                          │
├─────────────────────────────────────────────────────────────┤
│ 1. Socket 初始化 → 设置 TCP_TIMEOUT_INIT = 1s                │
│ 2. 接收 ACK → 采样 RTT（Karn 算法 + 时间戳）                  │
│ 3. 用 RTT 样本更新 SRTT / RTTVAR（Van Jacobson 1988 算法）    │
│ 4. 由 SRTT + 4·RTTVAR 计算基础 RTO                            │
│ 5. 边界钳制： tcp_rto_min ≤ RTO ≤ tcp_rto_max                │
│ 6. TCP_REARM: 正常 ACK → 重启定时器 = now + RTO              │
│ 7. TCP_TIMEOUT: 超时触发 → 指数 backoff / 线性 timeout        │
│ 8. CLAMP: 综合 USER_TIMEOUT、TCP_RTO_MAX、tcp_retries2       │
└─────────────────────────────────────────────────────────────┘
```

**关键源文件：**

| 文件 | 关键函数/宏 | 作用 |
|------|------------|------|
| [include/net/tcp.h](file:///workspace/include/net/tcp.h) | `__tcp_set_rto()`、`tcp_rto_min()`、`tcp_bound_rto()`、`TCP_RTO_MIN/MAX` | RTO 公式、上下界、内联工具函数 |
| [include/uapi/linux/tcp.h](file:///workspace/include/uapi/linux/tcp.h) | 用户态 TCP 常量/sysctl ABI | 用户可配置接口 |
| [net/ipv4/tcp.c](file:///workspace/net/ipv4/tcp.c) | `tcp_init_sock()`、tcp_sockopt 处理 | socket 创建时初始化 icsk_rto、rto_min、rto_max |
| [net/ipv4/tcp_input.c](file:///workspace/net/ipv4/tcp_input.c) | `tcp_rtt_estimator()`、`tcp_set_rto()`、`tcp_ack_update_rtt()`、`tcp_rearm_rto()` | RTT 采样、SRTT/RTTVAR 更新、RTO 重新计算、定时器重装 |
| [net/ipv4/tcp_timer.c](file:///workspace/net/ipv4/tcp_timer.c) | `tcp_retransmit_timer()`、`tcp_write_timeout()`、`tcp_clamp_rto_to_user_timeout()`、`retransmits_timed_out()` | 超时回调、指数退避、user_timeout 钳制、连接超时判定 |

---

## 二、常量与默认边界

定义在 [include/net/tcp.h#L160-L173](file:///workspace/include/net/tcp.h#L160-L173)：

```c
#define TCP_RTO_MAX_SEC    120
#define TCP_RTO_MAX        ((unsigned)(TCP_RTO_MAX_SEC * HZ))
#define TCP_RTO_MIN        ((unsigned)(HZ / 5))         // 200ms (@HZ=1000)
#define TCP_TIMEOUT_MIN    (2U)                         // 2 jiffies
#define TCP_TIMEOUT_MIN_US (2*USEC_PER_MSEC)            // 2ms

#define TCP_TIMEOUT_INIT   ((unsigned)(1*HZ))           // 初始 RTO: 1s (RFC 6298 §2.1)
#define TCP_TIMEOUT_FALLBACK ((unsigned)(3*HZ))         // SYN 重传后 fallback: 3s
```

几点说明：
- **RTO_MIN = HZ/5 ≈ 200ms**：RFC 推荐下限，避免局域网 ACK 抖动导致 spurious RTO。
- **RTO_MAX = 120s**：协议规定的理论 RTT 最大值。
- **TCP_TIMEOUT_INIT = 1s**：符合 RFC 6298 §2.1（1s ≤ Initial RTO ≤ 3s）。

---

## 三、Socket 初始化时的 RTO 初始值

在 `tcp_init_sock()` 中创建 TCP 传输控制块时：

[net/ipv4/tcp.c#L433-L442](file:///workspace/net/ipv4/tcp.c#L433-L442)

```c
icsk->icsk_rto = TCP_TIMEOUT_INIT;                            // 1s

rto_max_ms = READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_rto_max_ms);
icsk->icsk_rto_max = msecs_to_jiffies(rto_max_ms);            // 单连接 RTO 上限

rto_min_us = READ_ONCE(sock_net(sk)->ipv4.sysctl_tcp_rto_min_us);
icsk->icsk_rto_min = usecs_to_jiffies(rto_min_us);            // 单连接 RTO 下限
icsk->icsk_delack_max = TCP_DELACK_MAX;
tp->mdev_us = jiffies_to_usecs(TCP_TIMEOUT_INIT);
minmax_reset(&tp->rtt_min, tcp_jiffies32, ~0U);
```

**关键点**：
1. 初始 RTO 是 **1s**（`TCP_TIMEOUT_INIT`），而不是 `TCP_RTO_MIN`（200ms）。
2. `icsk_rto_max` / `icsk_rto_min` 可通过 netns sysctl 或 sockopt 分别覆盖（如 `TCP_RTO_MAX_MS`、`TCP_RTO_MIN_US`）。
3. `tp->mdev_us`（RTT 平均偏差）初始值 1,000,000us 使首个 RTO 保守地落在 `SRTT + 4·mdev = 3·RTT` 附近。

---

## 四、RTT 样本获取（Karn/Partridge + 时间戳）

`skb` 的 ACK 处理时调用 `tcp_ack_update_rtt()`，其位于 [net/ipv4/tcp_input.c#L3459-L3498](file:///workspace/net/ipv4/tcp_input.c#L3459-L3498)：

```c
static bool tcp_ack_update_rtt(struct sock *sk, const int flag,
                               long seq_rtt_us, long sack_rtt_us,
                               long ca_rtt_us, struct rate_sample *rs)
{
    /* 1) 优先使用 SEQ 计时（按发段时间戳），其次是 SACK；
     *    但如果 ACK 涉及重传段（Karn 算法禁止），seq_rtt_us 已被置 -1。
     */
    if (seq_rtt_us < 0)
        seq_rtt_us = sack_rtt_us;

    /* 2) 两者都没有，退回 TS-ECR（时间戳回显），但需满足 RTTM Rule：
     *    仅在 ACK 确实推进了左边界时才采纳 TS-ECR。
     */
    if (seq_rtt_us < 0 && tp->rx_opt.saw_tstamp &&
        tp->rx_opt.rcv_tsecr && flag & FLAG_ACKED)
        seq_rtt_us = ca_rtt_us = tcp_rtt_tsopt_us(tp, 1);

    if (seq_rtt_us < 0)
        return false;

    tcp_update_rtt_min(sk, ca_rtt_us, flag);   // 更新拥塞控制使用的 min-RTT 窗口
    tcp_rtt_estimator(sk, seq_rtt_us);         // 核心 SRTT/RTTVAR 平滑
    tcp_set_rto(sk);                           // 重新计算 icsk_rto
    inet_csk(sk)->icsk_backoff = 0;            // RFC 6298: 有效 RTT → backoff 归零
    return true;
}
```

**样本优先级**：`seq (首次发送段)` > `sack (SACK 确认段)` > `TS-ECR (TCP时间戳回显)`，这可以抵抗中间盒对 TS-ECR 的破坏。

---

## 五、Van Jacobson SRTT/RTTVAR 平滑算法

实现位于 [net/ipv4/tcp_input.c#L1070-L1136](file:///workspace/net/ipv4/tcp_input.c#L1070-L1136)，完全对应 SIGCOMM'88 VJ 算法，但有针对 **RTT 下降时** 的 Eifel 变体修正：

```c
static void tcp_rtt_estimator(struct sock *sk, long mrtt_us)
{
    struct tcp_sock *tp = tcp_sk(sk);
    long m = mrtt_us;
    u32 srtt = tp->srtt_us;

    if (srtt != 0) {
        m -= (srtt >> 3);         // m = error = sample - SRTT_old  (gain α=1/8)
        srtt += m;                // SRTT_new = SRTT_old + m = 7/8·SRTT_old + 1/8·sample
        if (m < 0) {
            m = -m;
            m -= (tp->mdev_us >> 2);
            /* 关键修正：RTT 下降时对 mdev 采用更细的增益 α·β，
             * 防止 RTT 突然减小但抖动估计滞后，使 RTO 下降过慢/过快。
             * 灵感来自 Eifel 检测：当 RTT 降低时，暂停纯 Eifel 的 mdev 增长，
             * 这里通过 m>>=3（额外除 8）将 gain 从 β=1/4 降到 1/32。
             */
            if (m > 0)
                m >>= 3;
        } else {
            m -= (tp->mdev_us >> 2);   // β=1/4
        }
        tp->mdev_us += m;          // MDEV = 3/4 MDEV + 1/4 |error|

        /* 维护 rttvar_us = max(mdev_max_us), 在一个 RTT 窗口内统计最大 MDEV */
        if (tp->mdev_us > tp->mdev_max_us) {
            tp->mdev_max_us = tp->mdev_us;
            if (tp->mdev_max_us > tp->rttvar_us)
                tp->rttvar_us = tp->mdev_max_us;
        }
        /* 每推进一个 snd_una → snd_nxt 窗口：rttvar 衰减至最近的 mdev_max */
        if (after(tp->snd_una, tp->rtt_seq)) {
            if (tp->mdev_max_us < tp->rttvar_us)
                tp->rttvar_us -= (tp->rttvar_us - tp->mdev_max_us) >> 2;
            tp->rtt_seq = tp->snd_nxt;
            tp->mdev_max_us = tcp_rto_min_us(sk);   // 窗口内 mdev 不得低于 RTO_MIN
        }
    } else {
        /* 首次采样：直接将 sample 设为 SRTT，并设 mdev=2·RTT → RTO = SRTT/8 + 4·(mdev/4) ≈ 3·RTT */
        srtt = m << 3;
        tp->mdev_us = m << 1;
        tp->rttvar_us = max(tp->mdev_us, tcp_rto_min_us(sk));
        tp->mdev_max_us = tp->rttvar_us;
        tp->rtt_seq = tp->snd_nxt;
    }
    WRITE_ONCE(tp->srtt_us, max(1U, srtt));
}
```

**SRTT/RTTVAR 在内存中都是左移缩放（SRTT<<3、mdev/RTTVAR 线性 us）**，避免浮点运算。

---

## 六、RTO 的核心公式 `__tcp_set_rto`

定义在 [include/net/tcp.h#L881-L884](file:///workspace/include/net/tcp.h#L881-L884)：

```c
static inline u32 __tcp_set_rto(const struct tcp_sock *tp)
{
    return usecs_to_jiffies((tp->srtt_us >> 3) + tp->rttvar_us);
}
```

结合缩放公式：
- `SRTT = srtt_us >> 3` （还原成 us）
- `RTTVAR = rttvar_us`（us）

因此公式为：

> **RTO = SRTT + 4·MDEV ≈ SRTT + RTTVAR**（因为 `rttvar_us` 是窗口最大偏差，其初值就是 4·MDEV）

这与 RFC 6298 §2.3 推荐的 `RTO = SRTT + 4·RTTVAR` 高度一致，只是 Linux 为了平滑使用了"一个 RTT 窗口内最大 mdev"作为 rttvar_us，使估计更保守。

### 上下界钳制

**最小 RTO**：[tcp_rto_min()](file:///workspace/include/net/tcp.h#L897-L905)

```c
static inline u32 tcp_rto_min(const struct sock *sk)
{
    const struct dst_entry *dst = __sk_dst_get(sk);
    u32 rto_min = READ_ONCE(inet_csk(sk)->icsk_rto_min);

    if (dst && dst_metric_locked(dst, RTAX_RTO_MIN))
        rto_min = dst_metric_rtt(dst, RTAX_RTO_MIN);   // 路由优先级更高
    return rto_min;
}
```

**最大 RTO**：[tcp_rto_max()](file:///workspace/include/net/tcp.h#L871-L879) 通过 `icsk_rto_max`（sysctl TCP_RTO_MAX_MS, 默认 120s）控制，`tcp_bound_rto()` 在重算后裁剪。

在 `tcp_set_rto()` 汇总：

[net/ipv4/tcp_input.c#L1175-L1200](file:///workspace/net/ipv4/tcp_input.c#L1175-L1200)

```c
void tcp_set_rto(struct sock *sk)
{
    const struct tcp_sock *tp = tcp_sk(sk);
    inet_csk(sk)->icsk_rto = __tcp_set_rto(tp);
    tcp_bound_rto(sk);   // min(RTO, icsk_rto_max)
}
```

---

## 七、收到 ACK 后定时器的重装

`RFC 2988 §5` 建议在推进了发送窗口的 ACK 到来后，立即将 RTO 定时器重置为 `now + RTO`。实现为 [tcp_rearm_rto()](file:///workspace/net/ipv4/tcp_input.c#L3524-L3550)：

```c
void tcp_rearm_rto(struct sock *sk)
{
    if (rcu_access_pointer(tp->fastopen_rsk)) return;     // FO 期间不挪用 SYN-ACK 定时器
    if (!tp->packets_out) {
        inet_csk_clear_xmit_timer(sk, ICSK_TIME_RETRANS); // 没有在飞数据 → 取消
    } else {
        u32 rto = icsk->icsk_rto;
        /* 如果已经装了 Loss Probe / REO Timer 且在未来，换算成剩余时间，
         * 保证 RACK/FACK 探测与真正 RTO 不会互相漂移。
         */
        if (icsk->icsk_pending == ICSK_TIME_REO_TIMEOUT ||
            icsk->icsk_pending == ICSK_TIME_LOSS_PROBE) {
            s64 delta_us = tcp_rto_delta_us(sk);
            rto = usecs_to_jiffies(max_t(int, delta_us, 1));
        }
        tcp_reset_xmit_timer(sk, ICSK_TIME_RETRANS, rto, true);
    }
}
```

**注意**：`tcp_ack_update_rtt` → `tcp_set_rto` 之后会清零 `icsk_backoff`，此时使用的是 **不加退避** 的基础 RTO。

---

## 八、RTO 超时：`tcp_retransmit_timer` 与退避策略

当定时器到期 → `tcp_write_timer` → `tcp_write_timer_handler` → `tcp_retransmit_timer(sk)`。

核心入口在 [net/ipv4/tcp_timer.c#L535-L690](file:///workspace/net/ipv4/tcp_timer.c#L535-L690)。

### 8.1 重传与最大尝试次数

```c
void tcp_retransmit_timer(struct sock *sk)
{
    if (!tp->packets_out) return;
    ...
    tcp_enter_loss(sk);                                      // 进入 LOSS 态，窗口掉 1 mss
    tcp_update_rto_stats(sk);                                // mib 统计
    if (tcp_retransmit_skb(sk, tcp_rtx_queue_head(sk), 1) > 0) {
        tcp_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                             TCP_RESOURCE_PROBE_INTERVAL, false);
        goto out;
    }
```

### 8.2 三种退避/超时策略（重点）

```c
out_reset_timer:
    /* 策略 A：薄流（thin stream），前 6 次线性超时不做指数退避。
     * packets_out < 4 且 不在初始慢启动阶段，且 用户开了 tcp_thin_linear_timeouts
     */
    if (sk->sk_state == TCP_ESTABLISHED &&
        (tp->thin_lto || READ_ONCE(net->ipv4.sysctl_tcp_thin_linear_timeouts)) &&
        tcp_stream_is_thin(tp) &&
        icsk->icsk_retransmits <= TCP_THIN_LINEAR_RETRIES) {
        icsk->icsk_backoff = 0;
        icsk->icsk_rto = clamp(__tcp_set_rto(tp),
                               tcp_rto_min(sk),
                               tcp_rto_max(sk));
    }
    /* 策略 B：SYN_SENT 阶段在 tcp_syn_linear_timeouts 次以内也用线性（不翻倍）*/
    else if (sk->sk_state != TCP_SYN_SENT ||
             tp->total_rto >
             READ_ONCE(net->ipv4.sysctl_tcp_syn_linear_timeouts)) {
        /* 策略 C：标准指数退避：backoff++ ; rto<<=1，不超过 RTO_MAX */
        icsk->icsk_backoff++;
        icsk->icsk_rto = min(icsk->icsk_rto << 1, tcp_rto_max(sk));
    }

    /* 综合 USER_TIMEOUT，然后装定时器 */
    tcp_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                         tcp_clamp_rto_to_user_timeout(sk), false);
```

#### 策略 A：薄流线性退避
定义 `thin stream`：`packets_out < 4 && !tcp_in_initial_slowstart(tp)`，见 [include/net/tcp.h#L2418-L2421](file:///workspace/include/net/tcp.h#L2418-L2421)。

对小包交互型业务（SSH/游戏），`packets_out` 少 → F-RTO/TLP 难工作、指数退避会把 RTT 撑爆。Linux 允许最多 6 次重传保持基础 RTO 不翻倍，`TCP_THIN_LINEAR_RETRIES` 控制。

#### 策略 B：SYN 阶段线性退避
`net.ipv4.tcp_syn_linear_timeouts` 次 SYN RTO 内不做 2 倍翻。典型是把 TCP 握手的 RTT 前几个样本留给 RTT 采样，防止首 SYN 丢包后 fallback 直接跳到 3s×2^n。

#### 策略 C：经典指数退避
`icsk_rto = min(icsk_rto << 1, tcp_rto_max)`，同时 `icsk_backoff` 记录翻倍次数。

### 8.3 `USER_TIMEOUT` 钳制 `tcp_clamp_rto_to_user_timeout`

[net/ipv4/tcp_timer.c#L28-L48](file:///workspace/net/ipv4/tcp_timer.c#L28-L48)：

```c
static u32 tcp_clamp_rto_to_user_timeout(const struct sock *sk)
{
    user_timeout = READ_ONCE(icsk->icsk_user_timeout);
    if (!user_timeout) return icsk->icsk_rto;

    elapsed = tcp_time_stamp_ts(tp) - tp->retrans_stamp;
    remaining = user_timeout - elapsed;
    if (remaining <= 0) return 1;    /* 立即触发超时 */
    return min_t(u32, icsk->icsk_rto, msecs_to_jiffies(remaining));
}
```

含义：即使当前 icsk_rto 已经指数涨到 10s，但 `SO_USER_TIMEOUT = 5s`，且自首个未确认段以来已经过了 4.8s，则下一次重传定时器只设 **200ms**，确保用户配置的"从发送起总共 5s 内没确认就断"严格满足。

---

## 九、连接级超时判定 `retransmits_timed_out` + `tcp_write_timeout`

### 9.1 retransmits_timed_out
[net/ipv4/tcp_timer.c#L203-L240](file:///workspace/net/ipv4/tcp_timer.c#L203-L240)。它按 `icsk_retransmits` 次数累计真实的 `jiffies`/`usec` 经过，再与 `boundary`（tcp_retries1/2）与 `timeout`（user_timeout）比较。

采用的是**指数和 + 线性尾部**的等效 timeout 公式（见 194-200 行）：
```
timeout_rto = (2^(min(boundary, log2(rto_max/rto_base))) - 1) * rto_base
            + (boundary - ...) * rto_max
```

这使得 `sysctl tcp_retries2=15` 时，整体连接超时大约落在 13-30 分钟量级（视 RTT 而定）。

### 9.2 tcp_write_timeout 整体逻辑
[net/ipv4/tcp_timer.c#L243-L300](file:///workspace/net/ipv4/tcp_timer.c#L243-L300)：

| 阶段 | 判定边界 | 超时时行为 |
|------|---------|-----------|
| **SYN_SENT / SYN_RECV** | `icsk_syn_retries`（默认 tcp_syn_retries） | 直接标记 expired → RST |
| **ESTABLISHED / 其他** 前半程 | `tcp_retries1`（≈3） | `tcp_mtu_probing` 黑洞探测，`dst_negative_advice` |
| **ESTABLISHED / 其他** 后半程 | `tcp_retries2`（≈15） **或** `icsk_user_timeout` | 标记 expired → `tcp_write_err` → 发送 RST |
| **孤儿 sk**（SOCK_DEAD） | `tcp_orphan_retries` 动态更少 | 省资源地快速回收 |

---

## 十、SYN-ACK RTT 样本

三次握手完成后，Ack 到的 ACK 会触发 `tcp_synack_rtt_meas()`，它把 **SYN-ACK → ACK** 时间作为首个 RTT 样本。[net/ipv4/tcp_input.c#L3501-L3510](file:///workspace/net/ipv4/tcp_input.c#L3501-L3510)：

```c
void tcp_synack_rtt_meas(struct sock *sk, struct request_sock *req)
{
    struct rate_sample rs;
    long rtt_us = -1L;
    if (req && !req->num_retrans && tcp_rsk(req)->snt_synack)
        rtt_us = tcp_stamp_us_delta(tcp_clock_us(), tcp_rsk(req)->snt_synack);
    tcp_ack_update_rtt(sk, FLAG_SYN_ACKED, rtt_us, -1L, rtt_us, &rs);
}
```

条件 `!req->num_retrans` 体现了 **Karn's 算法**：如果 SYN-ACK 重传过，则该样本无效、丢弃（否则 RTT 会被高估导致 RTO 飙升）。此时 `tcp_rtt_estimator` 不会运行 → `srtt_us` 仍为 0 → `TCP_TIMEOUT_FALLBACK = 3s` 作为首个数据 RTO。

---

## 十一、完整调用链总结

```
 tcp_v4_init_sock / tcp_init_transfer
     │
     ├─ icsk->icsk_rto = TCP_TIMEOUT_INIT (1s)
     │   icsk->icsk_rto_max/rto_min 来自 sysctl
     │
 ┌───▼───────────────────┐
 │  收到 ACK (tcp_rcv_established) │
 └───┬───────────────────┘
     │
     ├─ tcp_clean_rtx_queue() 计算 seq_rtt_us / sack_rtt_us / ca_rtt_us
     │     ├─ 涉及重传? seq_rtt_us = -1 (Karn)
     │
     ├─ tcp_ack_update_rtt(seq, sack, ca)
     │    ├─ (sample 选择：seq > sack > TSecr)
     │    ├─ tcp_update_rtt_min()     // BBR/copa 的 min-rtt
     │    ├─ tcp_rtt_estimator(mrtt)  // SRTT = 7/8 + 1/8, MDEV = 3/4 + 1/4
     │    │                         //  +  RTT 下降时 β' 细化
     │    ├─ tcp_set_rto()
     │    │    └─ icsk_rto = clamp(SRTT/8 + rttvar, rto_min, rto_max)
     │    └─ icsk_backoff = 0       // RFC 6298 §5.3
     │
     ├─ tcp_set_xmit_timer()
     │    └─ tcp_rearm_rto()        // 重新装 ICSK_TIME_RETRANS = now + icsk_rto
     │
 ┌───▼─────────────────────┐
 │ 重传定时器触发 (HZ wheel)│
 └───┬─────────────────────┘
     ├─ tcp_write_timer → tcp_write_timer_handler → tcp_retransmit_timer
     │    ├─ tcp_enter_loss / tcp_retransmit_skb
     │    ├─ 选择策略 A/B/C：thin / SYN-linear / 指数翻倍
     │    ├─ tcp_clamp_rto_to_user_timeout → 考虑 SO_USER_TIMEOUT
     │    ├─ tcp_reset_xmit_timer(...)   // 重装
     │    ├─ tcp_write_timeout → retransmits_timed_out(tcp_retries1/2 / user_timeout)
     │    └─ expired → tcp_write_err / tcp_done(ETIMEDOUT)
```

---

## 十二、可调参数速查

| sysctl / sockopt | 默认 | 说明 |
|------------------|------|------|
| `net.ipv4.tcp_rto_min_us` | 200,000 | 全局 RTO 下限（μs），新建 socket 继承到 `icsk_rto_min` |
| `net.ipv4.tcp_rto_max_ms` | 120,000 | 全局 RTO 上限（ms）；新建 socket 继承为 `icsk_rto_max` |
| `TCP_TIMEOUT_INIT` 常量 | 1 s | 首个 RTO 初始值 |
| `net.ipv4.tcp_syn_retries` | 6 | SYN/SYN-ACK 重传次数上限 |
| `net.ipv4.tcp_syn_linear_timeouts` | 0 | SYN 阶段使用线性 RTO 的次数（不翻倍） |
| `net.ipv4.tcp_retries1` | 3 | 触发 MTU 黑洞探测的重传次数 |
| `net.ipv4.tcp_retries2` | 15 | 数据段重传总次数阈值（等效 ETIMEDOUT） |
| `net.ipv4.tcp_thin_linear_timeouts` | 0 | 薄流全局线性 RTO 开关 |
| `SO_USER_TIMEOUT` (ms) | 0 = disabled | 从发送起累计超时，覆盖一切 RTO 退避 |
| `TCP_RTO_MIN_US` sockopt | 继承 sysctl | 单连接覆写 rto_min |
| `TCP_RTO_MAX_MS` sockopt | 继承 sysctl | 单连接覆写 rto_max |
| `RTAX_RTO_MIN`（路由 metric，locked）| — | 若 dst 被管理员 lock，则覆盖 socket 级 rto_min |

---

## 十三、参考的 RFC

- **RFC 6298**：Computing TCP's Retransmission Timer（现行 RTO 计算标准）
- **RFC 2988 / 1122**：老版本，含 3s initial RTO，已被 6298 修订
- **Karn/Partridge SIGCOMM'87**：重传段的 RTT 样本一律丢弃
- **Jacobson SIGCOMM'88**：SRTT / RTTVAR 的 α/β 平滑算法
- **Eifel Detection**：影响 VJ mdev 更新（RTT↓ gain 细化）
- **Thin-stream RFC 8312**：`tcp_thin_linear_timeouts` 的标准依据
