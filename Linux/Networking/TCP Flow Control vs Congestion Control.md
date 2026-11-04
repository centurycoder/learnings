# TCP Flow Control vs Congestion Control

Both limit how much a TCP sender can send, but they protect different things:

- **Flow control:** Can the **receiver** accept more data?
- **Congestion control:** Can the **network path** handle more data?

## Flow control: protect the receiver

Incoming data waits in the receiver's TCP receive buffer until the application reads it. If the application reads too slowly, the buffer fills.

The receiver advertises a **receive window (`rwnd`)** in TCP headers, usually on ACKs, telling the sender how much more data it can accept.

- As the application consumes data, buffer space becomes available.
- As the buffer fills, the advertised window can shrink.
- A **zero window** pauses ordinary data transmission until space becomes available; TCP can still send window probes.

Flow control matters even when the network is uncongested.

## Congestion control: protect the network

A bottleneck link may not forward data as quickly as senders inject it. Queues build up, delays increase, and packets may be dropped.

The sender maintains a **congestion window (`cwnd`)**, limiting *how much unacknowledged data it puts into the network*. Congestion-control algorithms adjust this limit using signals such as packet loss or explicit congestion notifications (ECN); exact behavior depends on the algorithm.

Congestion control matters even when the receiver has plenty of buffer space.

### Slow start

**Slow start is part of congestion control, not flow control.** It helps the sender discover how much data the network can handle without immediately sending a large burst.

1. Begin with a relatively small **congestion window (`cwnd`)**.
2. Increase `cwnd` as acknowledgments arrive. When the sender can fully use the window, growth is roughly exponential—about doubling each **round-trip time (RTT)** under typical conditions.
3. In classic TCP algorithms, reaching the **slow-start threshold (`ssthresh`)** switches growth to the more cautious **congestion avoidance** phase. A congestion signal can interrupt growth earlier; the response depends on the algorithm and signal.

Despite its name, slow start **starts small but grows quickly**. “Slow” contrasts with immediately sending a large burst.

Flow control still applies throughout: a small receiver window (`rwnd`) can limit transmission even while slow start increases `cwnd`.

## Comparison

| Aspect | Flow control | Congestion control |
|---|---|---|
| Protects | Receiver's buffer | Network path |
| Key window | `rwnd` | `cwnd` |
| Who determines it? | Receiver advertises it | Sender computes it |
| Main concern | Receiver cannot consume data fast enough | Network cannot carry data fast enough |

## How they work together

The sender's effective window is approximately:

```text
allowed data in flight = min(rwnd, cwnd)
```

**In flight** means sent but not yet acknowledged. Data already in flight uses part of this allowance; the window is not a fresh allowance for every send.

| Receive window (`rwnd`) | Congestion window (`cwnd`) | Effective window | Limiting factor |
|---:|---:|---:|---|
| 16 KiB | 64 KiB | 16 KiB | Receiver: flow control |
| 64 KiB | 16 KiB | 16 KiB | Network: congestion control |

Windows limit **bytes**, not directly bytes per second. How quickly acknowledgments return also affects throughput.

**Memory aid: flow control protects the endpoint; congestion control protects the path.**
