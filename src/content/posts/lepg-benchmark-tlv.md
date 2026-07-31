---
author: 按位或菌
pubDatetime: 2026-07-30T21:52:47.000+08:00
modDatetime: 2026-07-30T21:52:47.000+08:00
title: LEPG 自定义 TLV 协议性能测试调优实践
featured: false
draft: true
tags:
  - Go
  - LEPG
description: 使用 Go Benchmark 和 CPU Profiler 给 LEPG 自定义 TLV 协议实现性能测试并进行分析调优的工作流的主要日志。
---

## 准备工作

在基本实现了 LEPG 的主要 workflow 之后，我开始对 LEPG 进行性能测试。测试分为两个部分：

1. 基础函数性能测试：对装包、拆包、读数处理等基础功能（函数）通过 Go Benchmark 进行性能测试，重点关注单个功能本身的净耗时。
2. 完整功能性能测试：对 LEPG 的完整 workflow 进行延迟、吞吐等指标的性能测试。

第一部分的测试对象包括：

1. 自定义 TLV 协议的 Encode/Decode
2. Server 端的 `handleUpload`

具体到代码中，需要测试以下两个文件：

1. LEPG/internal/msg/msg_bench_test.go
2. LEPG/internal/server/server_bench_test.go

本文将会作为第一部分展示对自定义 TLV 协议的性能测试。

其中 Benchmark 函数的大体结构如下（拿一个示例举例）：

```go
func BenchmarkEncode_1Reading(b *testing.B) {
  // 准备数据
	msg := preBuiltMsg(makeTestReadings(1))

  // 开始计时
	b.ResetTimer()
  
	for i := 0; i < b.N; i++ {
    // 执行要测试的核心逻辑
		sinkBytes, sinkErr = msg.Encode()
	}
}
```

## 第一轮 Benchmark

第一次 Benchmark：commit `ad337923f900c5c660bb67d4377edef19ccc1264`

运行：`go test -bench . -benchmem LEPG/internal/msg -count 5 > msg_bench.txt` 进行五次测试并把结果保存在文件里。文件结果如下：

```
goos: windows
goarch: amd64
pkg: LEPG/internal/msg
cpu: 12th Gen Intel(R) Core(TM) i5-12500H
BenchmarkEncode_12Readings-16             894596              1401 ns/op            2992 B/op         13 allocs/op
BenchmarkEncode_12Readings-16             811753              1638 ns/op            2992 B/op         13 allocs/op
BenchmarkEncode_12Readings-16             749568              1621 ns/op            2992 B/op         13 allocs/op
BenchmarkEncode_12Readings-16             847762              1661 ns/op            2992 B/op         13 allocs/op
BenchmarkEncode_12Readings-16             752926              1586 ns/op            2992 B/op         13 allocs/op
BenchmarkEncode_1Reading-16              1538638               856.6 ns/op           752 B/op         13 allocs/op
BenchmarkEncode_1Reading-16              1392342               880.1 ns/op           752 B/op         13 allocs/op
BenchmarkEncode_1Reading-16              1390030               903.4 ns/op           752 B/op         13 allocs/op
BenchmarkEncode_1Reading-16              1350392               886.5 ns/op           752 B/op         13 allocs/op
BenchmarkEncode_1Reading-16              1474674               871.4 ns/op           752 B/op         13 allocs/op
BenchmarkEncode_100Readings-16            121599              9723 ns/op           20656 B/op         13 allocs/op
BenchmarkEncode_100Readings-16            160518              8611 ns/op           20656 B/op         13 allocs/op
BenchmarkEncode_100Readings-16            113425              9249 ns/op           20656 B/op         13 allocs/op
BenchmarkEncode_100Readings-16            118648              9024 ns/op           20656 B/op         13 allocs/op
BenchmarkEncode_100Readings-16            129435              9193 ns/op           20656 B/op         13 allocs/op
BenchmarkDecodeFrame_12Readings-16         14152             84150 ns/op            4388 B/op         32 allocs/op
BenchmarkDecodeFrame_12Readings-16         13857             79958 ns/op            4389 B/op         32 allocs/op
BenchmarkDecodeFrame_12Readings-16         14984             78889 ns/op            4387 B/op         32 allocs/op
BenchmarkDecodeFrame_12Readings-16         14646             80273 ns/op            4388 B/op         32 allocs/op
BenchmarkDecodeFrame_12Readings-16         15096             92142 ns/op            4387 B/op         32 allocs/op
BenchmarkDecodeFrame_1Reading-16           38372             28360 ns/op            2113 B/op         32 allocs/op
BenchmarkDecodeFrame_1Reading-16           40515             28734 ns/op            2113 B/op         32 allocs/op
BenchmarkDecodeFrame_1Reading-16           43830             28247 ns/op            2113 B/op         32 allocs/op
BenchmarkDecodeFrame_1Reading-16           40219             28862 ns/op            2113 B/op         32 allocs/op
BenchmarkDecodeFrame_1Reading-16           41010             27969 ns/op            2113 B/op         32 allocs/op
BenchmarkDecodeFrame_100Readings-16         2143            549015 ns/op           22051 B/op         32 allocs/op
BenchmarkDecodeFrame_100Readings-16         2067            550953 ns/op           22053 B/op         32 allocs/op
BenchmarkDecodeFrame_100Readings-16         1881            581053 ns/op           22052 B/op         32 allocs/op
BenchmarkDecodeFrame_100Readings-16         2241            556152 ns/op           22050 B/op         32 allocs/op
BenchmarkDecodeFrame_100Readings-16         1771            615962 ns/op           22050 B/op         32 allocs/op
PASS
ok      LEPG/internal/msg       48.039s
```

利用 benchstat 分析多次统计数据：

`PS D:\Nayami\Projects\LEPG> benchstat .\msg_bench.txt`

```
goos: windows
goarch: amd64
pkg: LEPG/internal/msg
cpu: 12th Gen Intel(R) Core(TM) i5-12500H
                           │ .\msg_bench.txt │
                           │     sec/op      │
Encode_12Readings-16            1.315µ ± 29%
Encode_1Reading-16              859.0n ±  6%
Encode_100Readings-16           9.103µ ± 15%
DecodeFrame_12Readings-16       69.10µ ±  4%
DecodeFrame_1Reading-16         26.94µ ±  6%
DecodeFrame_100Readings-16      525.6µ ±  2%
geomean                         14.69µ

                           │ .\msg_bench.txt │
                           │      B/op       │
Encode_12Readings-16            2.922Ki ± 0%
Encode_1Reading-16                752.0 ± 0%
Encode_100Readings-16           20.17Ki ± 0%
DecodeFrame_12Readings-16       4.284Ki ± 0%
DecodeFrame_1Reading-16         2.063Ki ± 0%
DecodeFrame_100Readings-16      21.53Ki ± 0%
geomean                         4.494Ki

                           │ .\msg_bench.txt │
                           │    allocs/op    │
Encode_12Readings-16              13.00 ± 0%
Encode_1Reading-16                13.00 ± 0%
Encode_100Readings-16             13.00 ± 0%
DecodeFrame_12Readings-16         32.00 ± 0%
DecodeFrame_1Reading-16           32.00 ± 0%
DecodeFrame_100Readings-16        32.00 ± 0%
geomean                           20.40
```

主要看第一项的 sec/op，表示每次操作的所需时间。

## Benchmark 精度优化（开始前重置 GC）

Encoding 普遍有较大的浮动误差，可能是因为测试数据的准备阶段中产生的垃圾在开始测试后提前触发了 GC 导致的。故加上了每次开始测试前重置 GC 的操作，GC 重置优化（d625ed1a86ad3324a6162192db7c4f114a931b54）后的 Benchmark 为：

```
goos: windows
goarch: amd64
pkg: LEPG/internal/msg
cpu: 12th Gen Intel(R) Core(TM) i5-12500H
                           │ .\msg_bench.txt │
                           │     sec/op      │
Encode_12Readings-16            1.364µ ± 12%
Encode_1Reading-16              699.9n ±  7%
Encode_100Readings-16           7.027µ ± 20%
DecodeFrame_12Readings-16       71.99µ ±  4%
DecodeFrame_1Reading-16         25.60µ ±  4%
DecodeFrame_100Readings-16      526.6µ ±  1%
geomean                         13.66µ

                           │ .\msg_bench.txt │
                           │      B/op       │
Encode_12Readings-16            2.922Ki ± 0%
Encode_1Reading-16                752.0 ± 0%
Encode_100Readings-16           20.17Ki ± 0%
DecodeFrame_12Readings-16       4.284Ki ± 0%
DecodeFrame_1Reading-16         2.063Ki ± 0%
DecodeFrame_100Readings-16      21.53Ki ± 0%
geomean                         4.494Ki

                           │ .\msg_bench.txt │
                           │    allocs/op    │
Encode_12Readings-16              13.00 ± 0%
Encode_1Reading-16                13.00 ± 0%
Encode_100Readings-16             13.00 ± 0%
DecodeFrame_12Readings-16         32.00 ± 0%
DecodeFrame_1Reading-16           32.00 ± 0%
DecodeFrame_100Readings-16        32.00 ± 0%
geomean                           20.40
```

原来 12 readings 的 29%的浮动（benchmark 实现代码导致的测量误差）被优化掉了，100 readings 仍有较大浮动的原因是因为待测逻辑本身触发的 GC 导致的，其它测试项同理。

故认为：encode 的时间浮动主要是因为触发 GC 带来的 GC pause，decode 相对稳定且时间普遍更长是因为有更大的因素压住了 GC pause 的影响。之后会单独对 Decode 进行性能瓶颈分析。

## 内存分配分析

看一下另外两个指标：`B/op` 和 `allocs/op`。

- B/op：每次调用在堆上分配的字节数
- allocs/op：每次调用在堆上分配的次数

分析这两个指标可以确认：

1. 分配模式是否随输入规模膨胀
2. 是否有不必要的大量分配

B/op 和 allocs/op 一起看：

- 1 reading → 752 B/op, 13 allocs
- 12 readings → 2992 B/op, 13 allocs
- 100 readings → 20656 B/op, 13 allocs

B/op 随 payload 大小增加而增加，allocs 恒定 13，和预期一致。在实际业务中要避免量大频率高的情况，相比之下，量大时频率恒定是可接受的业务代价。

> 另外，就测试过程中而言，以 100 readings 举例，`b.N` 均大于 100000，$20656\ B/op \times 10,0000 = 1.92\ GB$，说明至少在测试过程中会分配约 2 GB 内存，从而频繁导致 GC Pause。这也是上面结果中优化后仍然有一定误差的原因。

## Decode 性能瓶颈分析

Decode 消耗的时间远大于 Encode，所以在跑 benchmark 时开 CPU profiling 然后用 pprof 判断是否在业务逻辑必要开销之外的冗余开销。

一次跑单个 benchmark：`BenchmarkDecodeFrame_12Readings`

`go test -bench BenchmarkDecodeFrame_12Readings -benchmem LEPG/internal/msg -cpuprofile cpu.prof -count 1 -benchtime 5s > msg_bench.txt`

```
PS D:\Nayami\Projects\LEPG> go tool pprof cpu.prof                                                                                            
File: msg.test.exe  
Build ID: D:\Nayami\Projects\LEPG\msg.test.exe2026-07-31 03:13:05.2569106 +0800 CST
Type: cpu
Time: 2026-07-31 03:16:46 CST
Duration: 6.41s, Total samples = 7.01s (109.36%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 5.52s, 78.74% of 7.01s total
Dropped 152 nodes (cum <= 0.04s)
Showing top 10 nodes out of 96
      flat  flat%   sum%        cum   cum%
     4.03s 57.49% 57.49%      4.03s 57.49%  LEPG/internal/utils.CalChecksum
     0.62s  8.84% 66.33%      0.62s  8.84%  runtime.semawakeup
     0.18s  2.57% 68.90%      0.49s  6.99%  runtime.selectgo
     0.17s  2.43% 71.33%      0.17s  2.43%  runtime.runqgrab.osyield.func1
     0.12s  1.71% 73.04%      0.22s  3.14%  runtime.lock2
     0.11s  1.57% 74.61%      0.13s  1.85%  runtime.unlock2
     0.08s  1.14% 75.75%      0.08s  1.14%  internal/runtime/atomic.(*Uint32).CompareAndSwap (inline)
     0.07s     1% 76.75%      0.07s     1%  runtime.memclrNoHeapPointers
     0.07s     1% 77.75%      0.07s     1%  runtime.semasleep
     0.07s     1% 78.74%      0.07s     1%  runtime.sysUnusedOS
```

一半的时间都花在 CalChecksum 上，说明实现有问题。

### CRC 校验优化

看一下 DecodeFrame 函数的实现，只在最后有一个 `CalChecksum` 函数调用用于确认接收信息无误。追溯到 utils 包，实现如下：

```go
func CalChecksum(payload []byte) uint16 {
	crc := uint16(crc16Init)

	for _, b := range payload {
		crc ^= uint16(b) << 8

		for range 8 {
			if crc&0x8000 != 0 {
				crc = (crc << 1) ^ uint16(crc16Poly)
			} else {
				crc <<= 1
			}
		}
	}

	return crc
}
```

现有的 CRC 实现很标准但并不实用，它对每个字节要跑八次迭代移位和条件判断引入大量分支预测开销导致 CPU 流水线频繁清空。故引入查表法用少量空间换取复杂度从 `O(8n)` 下降到 `O(n)`：

```go
var crc16Table [256]uint16

func init() {
	for i := range crc16Table {
		crc := uint16(i) << 8
		for range 8 {
			if crc&0x8000 != 0 {
				crc = (crc << 1) ^ crc16Poly
			} else {
				crc <<= 1
			}
		}
		crc16Table[i] = crc
	}
}

// CalChecksum calculates CRC16-CCITT checksum.
func CalChecksum(payload []byte) uint16 {
	crc := uint16(crc16Init)
	for _, b := range payload {
		crc = (crc << 8) ^ crc16Table[byte(crc>>8)^b]
	}
	return crc
}

```

改成标准的查表法以后重新看 benchmark：

```
goos: windows
goarch: amd64
pkg: LEPG/internal/msg
cpu: 12th Gen Intel(R) Core(TM) i5-12500H
                          │ .\msg_bench.txt │
                          │     sec/op      │
DecodeFrame_12Readings-16       27.29µ ± 3%

                          │ .\msg_bench.txt │
                          │      B/op       │
DecodeFrame_12Readings-16      4.284Ki ± 0%

                          │ .\msg_bench.txt │
                          │    allocs/op    │
DecodeFrame_12Readings-16        32.00 ± 0%
```

时间上缩小六成。剩下还有 27μ，对于我们的场景足够快，然后剩余的可优化项目有：

headerAndPayload() 重新序列化问题：每次解码完又反向拼回 []byte 去算 CRC，9 次分段 io.ReadFull + 9 次 `make([]byte)` 可以改成一次读 HeaderSize+payloadLen+2 字节然后切片。

### 消除冗余内存分配

经过调整（`8f16ccc5b825a44e3afb690b326c49d665bbd000`）：

```
PS D:\Nayami\Projects\LEPG> benchstat .\msg_bench.txt
goos: windows
goarch: amd64
pkg: LEPG/internal/msg
cpu: 12th Gen Intel(R) Core(TM) i5-12500H
                          │ .\msg_bench.txt │
                          │     sec/op      │
DecodeFrame_12Readings-16       12.52µ ± 2%

                          │ .\msg_bench.txt │
                          │      B/op       │
DecodeFrame_12Readings-16      4.160Ki ± 0%

                          │ .\msg_bench.txt │
                          │    allocs/op    │
DecodeFrame_12Readings-16        16.00 ± 0%
```

alloc 砍半，时间大幅缩短（虽然本来一小半就是 benchmark 自己的开销）

## 小结

回顾本次优化，DecodeFrame_12Readings 从最初的 ~84µs 降至 ~12µs，提升约 7 倍；单次解码的内存分配从 32 次降至 16 次。

涉及到三个关键手段：

1. **CRC 查表法**：用 512 字节的预计算表消除逐字节移位和分支预测开销，从 69µs 降到 27µs
2. **合并分段读取**：把 9 次 io 改为一次读取后切片让 alloc 减半
3. **pprof 精准定位**：利用 CPU profiling 追踪定位性能黑洞

到此，TLV 协议性能测试结束。