AWS Lambda-Power-Tuning-and-Payload-Testing

# Finding the Knee: AWS Lambda Memory Tuning from 128 MB to 3008 MB and API Load Testing


A serverless microservice (API Gateway → Lambda → DynamoDB) load-tested at seven memory
configurations with Postman, cross-referenced with AWS Lambda Power Tuning, to find where increasing
Lambda memory stops paying for itself — in latency and in dollars.

The goal was to compare Lambda memory settings using two approaches:

1. AWS Lambda Power Tuning for function-level cost and execution-time analysis.
2. Postman load testing for end-to-end API performance validation through API Gateway, Lambda and DynamoDB.

# Executive Summary

The results show that Lambda memory is not only a memory setting. It is a performance and cost lever.

Increasing memory can improve available compute power and reduce duration, but the relationship between memory, execution time, throughput and cost is not linear.

The optimal Lambda memory setting depends on the business goal:

## Which memory size should you pick?

| Goal | Candidate | Evidence (this test) |
|---|---|---|
| Lowest function-level cost | **256 MB** | $0.60 per 1M invocations — cheapest of seven |
| Fastest function-level execution | **1536 MB** | 46 ms isolated invocation (Power Tuning) |
| Best end-to-end API throughput | **2048 MB** | 34.93 req/s · 203 ms avg — but see note |
| Lowest tail latency | **1536 MB** | P99 271 ms · slowest request 1,321 ms |
| Strongest cost-performance balance | **1024 MB** | 210 ms for $0.87/M — 2.8× the speed of the cheapest at 1.46× the cost |
| Avoid due to poor cost-benefit | **3008 MB** | $6.16/M (10.28×) for the same 205 ms as 1536 MB, with a 54% worse tail |

> **Note on throughput.** This is a closed-loop test, so throughput is bound to latency
> (effective concurrency held at ~7.1 across all seven runs). The 1024–3008 MB results
> (210 / 205 / 203 / 205 ms) sit inside a 3.3% band — 2048 MB leads, but the gap is within
> run-to-run variance. Treat this range as a plateau, not a ranking.

# Test Architecture

The API Gateway endpoint invoked an AWS Lambda function that performed a DynamoDB-backed operation.

The same payload was used across each test run:

{
  "operation": "list",
  "tableName": "lambda-apigateway",
  "payload": {}
}

<img width="2400" height="2000" alt="image" src="https://github.com/user-attachments/assets/e508c79d-9491-46c7-944c-7a15b7699565" />

## Why this exists

On AWS Lambda, memory is not just RAM — it is the dial that also allocates vCPU. AWS
allocates CPU proportionally to configured memory, with one full vCPU at approximately
1,769 MB. That makes "how much memory?" an architecture decision with a recurring cost
consequence, not a checkbox.

The common advice is "give it more memory, it'll run faster and might even cost less."
That is true — right up until it isn't. This benchmark finds the point where it stops
being true, and prices it.

<img width="3000" height="1638" alt="image" src="https://github.com/user-attachments/assets/eab7b241-43f5-44aa-ac97-2e0a55c7d660" />


## Method
 
| Parameter | Value |
|---|---|
| Architecture | Postman → Amazon API Gateway (REST) → AWS Lambda → Amazon DynamoDB |
| Endpoint | `POST /Prod/DynamoDBManager` |
| Operation | `list` against DynamoDB table `lambda-apigateway` |
| Load profile | 10 virtual users, 2 minutes, 30-second ramp-up |
| Memory sizes | 128, 256, 512, 1024, 1536, 2048, 3008 MB |
| Runs | One Postman performance run per memory size, all other variables held constant |
| Supplementary | AWS Lambda Power Tuning (Step Functions) across 128–3008 MB |
 
Each memory size was tested with the same collection, same request, same endpoint and the
same load profile. Only the Lambda memory configuration changed between runs.
 
---
 
## Consolidated results
 
| Lambda memory | ~vCPU | Total requests | Throughput (req/s) | Avg (ms) | P95 (ms) | P99 (ms) | Max (ms) | Errors | Cost / 1M invocations | vs cheapest |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 128 MB | 0.07 | 1,921 | 15.97 | 451 | 521 | 938 | 3,641 | 0.00% | $0.81 | 1.35× |
| **256 MB** | 0.14 | 2,817 | 23.42 | 305 | 349 | 543 | 2,243 | 0.00% | **$0.60** | **1.00×** |
| 512 MB | 0.29 | 3,650 | 30.34 | 234 | 258 | 344 | 1,771 | 0.00% | $0.78 | 1.31× |
| **1024 MB** | 0.58 | 4,052 | 33.69 | **210** | 232 | 293 | 1,361 | 0.00% | $0.87 | 1.46× |
| **1536 MB** | 0.87 | 4,153 | 34.51 | 205 | 222 | **271** | **1,321** | 0.00% | $1.17 | 1.95× |
| 2048 MB | 1.16 | 4,205 | 34.93 | **203** | 220 | 271 | 1,410 | 0.00% | $1.75 | 2.92× |
| 3008 MB | 1.70 | 4,153 | 34.53 | 205 | 224 | 282 | 2,031 | 0.00% | **$6.16** | **10.28×** |
 
`~vCPU` is derived as `memory ÷ 1769`, per the AWS Lambda CPU allocation model.
 
### Power Tuning: isolated function cost and duration
 
Power Tuning measures the function in isolation (no API Gateway, no network), so its
durations are lower than the end-to-end Postman figures. All seven sizes were tested by
both tools.
 
| Memory | Invocation time (ms) | Invocation cost (USD) | Cost / 1M | vs cheapest |
|---:|---:|---:|---:|---:|
| 128 MB | 377 | $0.00000081 | $0.81 | 1.35× |
| **256 MB** | 141 | **$0.00000060** | **$0.60** | **1.00×** |
| 512 MB | 91 | $0.00000078 | $0.78 | 1.31× |
| 1024 MB | 51 | $0.00000087 | $0.87 | 1.46× |
| **1536 MB** | **46** | $0.00000117 | $1.17 | 1.95× |
| 2048 MB | 52 | $0.00000175 | $1.75 | 2.92× |
| 3008 MB | 126 | **$0.00000616** | **$6.16** | **10.28×** |
 
| Power Tuning verdict | Winner |
|---|---|
| Best cost | **256 MB** |
| Best time | **1536 MB** |
| Worst cost | **3008 MB** |
| Worst time | **128 MB** |

 <img width="1914" height="922" alt="image" src="https://github.com/user-attachments/assets/67c8397a-9d97-47a7-b54f-808fb75c6807" />

> **On the cost figures.** These are read from the Power Tuning results chart (Lambda
> compute only — they exclude the $0.20 per million request charge, API Gateway and
> DynamoDB). They were independently cross-checked against AWS's published pricing model,
> `GB × billed seconds × $0.0000166667`, and agree within 3.4% at every one of the seven
> sizes. Example for 256 MB: `0.25 GB × 0.141 s × $0.0000166667 = $0.000000587` versus
> $0.000000600 read from the chart.
 
### Lambda memory → vCPU allocation
 
Lambda does not expose a CPU setting. CPU is allocated in proportion to configured memory,
and **at 1,769 MB a function has the equivalent of one full vCPU**. Memory is configurable
from 128 MB to 10,240 MB, topping out at roughly 5.79 vCPU (6 cores).
 
```
vCPU ≈ memory (MB) ÷ 1,769
```
 
| Lambda memory | vCPU | % of one vCPU | Measured invocation time | What it means |
|---:|---:|---:|---:|---|
| 128 MB | 0.072 | 7.2% | 377 ms | Default. Severely CPU-starved. |
| 256 MB | 0.145 | 14.5% | 141 ms | Cheapest per invocation. |
| 512 MB | 0.289 | 28.9% | 91 ms | Gains still arriving. |
| 1024 MB | 0.579 | 57.9% | 51 ms | The knee — best value. |
| **1536 MB** | **0.868** | **86.8%** | **46 ms** | **Fastest. Just under one full vCPU.** |
| **1,769 MB** | **1.000** | **100%** | — | **One full vCPU. Single-threaded code cannot go faster past here.** |
| 2048 MB | 1.158 | 115.8% | 52 ms | Second vCPU begins — unusable to single-threaded code. |
| 3008 MB | 1.700 | 170.0% | 126 ms | Worst cost. No faster. |
| 10,240 MB | 5.789 | 578.9% | — | Lambda maximum (6 cores). |
 
**This is the mechanism behind the whole benchmark.** The handler is single-threaded Python,
so it can consume at most one vCPU. The fastest result — 1536 MB at 46 ms — sits at 0.87
vCPU, just below that ceiling. Every size above 1,769 MB allocates a *second* core that
single-threaded code physically cannot use, which is precisely why 2048 MB and 3008 MB
bought nothing but bill.
 
The knee was not a coincidence. It is where the workload ran out of core to consume.
 
> **Correction worth noting.** The lab instructions this exercise came from state that
> 128 MB gives 0.0625 vCPU and 1024 MB gives 0.5 vCPU — a mapping that implies one vCPU at
> 2048 MB. AWS documents the threshold at **1,769 MB**, which gives 0.072 and 0.579
> respectively. The figures in this table use AWS's documented model.
 
### Marginal returns per step
 
| Step | Memory change | Avg response time | Improvement |
|---|---|---|---:|
| 128 → 256 MB | 2× | 451 → 305 ms | **32.4%** |
| 256 → 512 MB | 2× | 305 → 234 ms | **23.3%** |
| 512 → 1024 MB | 2× | 234 → 210 ms | **10.3%** |
| 1024 → 1536 MB | 1.5× | 210 → 205 ms | 2.4% |
| 1536 → 2048 MB | 1.33× | 205 → 203 ms | 1.0% |
| 2048 → 3008 MB | 1.5× | 203 → 205 ms | **−1.0%** |

<img width="3600" height="1974" alt="image" src="https://github.com/user-attachments/assets/549489a7-948f-4f47-b270-dda74a592bee" />

# 128 MB Benchmark Data
<img width="680" height="951" alt="Screenshot 2026-07-15 at 17 36 37" src="https://github.com/user-attachments/assets/0ff33e26-be42-4d99-9362-8b511e7ffe3b" />
<img width="1293" height="799" alt="128MB_Screenshot 2026-07-15 at 14 08 17" src="https://github.com/user-attachments/assets/253d609d-2a8a-4ae1-9452-c5fb7fac364d" />


# 256 MB Benchmark Data
<img width="669" height="945" alt="Screenshot 2026-07-15 at 17 37 09" src="https://github.com/user-attachments/assets/795f3d7f-71bb-4743-af10-c39b0527ba0b" />
<img width="1296" height="941" alt="256MB Screenshot 2026-07-15 at 14 42 51" src="https://github.com/user-attachments/assets/2426052d-5ba7-403b-a3da-ab084341f0b7" />


# 512 MB Benchmark Data
<img width="674" height="954" alt="Screenshot 2026-07-15 at 17 37 43" src="https://github.com/user-attachments/assets/6fb6d3c3-4cc8-4ea1-96c6-754f50bd1e5f" />
<img width="1305" height="954" alt="512MB_Screenshot 2026-07-15 at 14 56 21" src="https://github.com/user-attachments/assets/30c7edb3-9ba2-42b4-85fd-efd6618d69b8" />

# 1024 MB Benchmark Data
<img width="668" height="952" alt="Screenshot 2026-07-15 at 17 38 15" src="https://github.com/user-attachments/assets/7a0c9663-3860-46ec-ba21-4d7ceab93b79" />
<img width="1293" height="924" alt="1024MB_Screenshot 2026-07-15 at 15 01 54" src="https://github.com/user-attachments/assets/d6908ca0-8342-460a-bbba-377ca9108c7f" />


# 1536 MB Benchmark Data
<img width="666" height="942" alt="Screenshot 2026-07-15 at 17 38 49" src="https://github.com/user-attachments/assets/6741ef6c-7666-4e9b-8dc3-6a202324deab" />
<img width="1316" height="954" alt="1536MB_Screenshot 2026-07-15 at 17 05 38" src="https://github.com/user-attachments/assets/a62e3368-4b2c-4b65-8374-5a95e5032979" />


# 2048 MB Benchmark Data
<img width="676" height="952" alt="Screenshot 2026-07-15 at 17 39 25" src="https://github.com/user-attachments/assets/fc7ea445-bbab-4b0a-89f0-1c9a953ee18c" />
<img width="1308" height="1012" alt="2048MB_Screenshot 2026-07-15 at 14 01 13" src="https://github.com/user-attachments/assets/77f73406-ddc2-45b0-a1fe-ad566ccab82d" />


# 3008 MB Benchmark Data
<img width="675" height="951" alt="Screenshot 2026-07-15 at 17 39 46" src="https://github.com/user-attachments/assets/c5c4d1f0-76c0-4e7f-bec0-92b13172dccb" />
<img width="1302" height="912" alt="3008MB_Screenshot 2026-07-15 at 15 09 20" src="https://github.com/user-attachments/assets/f57cfe5a-03b6-4794-bc14-737941bbd1db" />


---
 
## Analysis
 
### 1. The curve has a knee, and it sits around 1024 MB
 
From 128 MB to 512 MB, average response time falls 48% (451 → 234 ms). Below 512 MB the
function is CPU-starved, and every doubling of memory buys meaningful compute.
 
Past 1024 MB the curve flattens hard. With all seven sizes measured, the marginal returns
decay cleanly and monotonically: **32.4% → 23.3% → 10.3% → 2.4% → 1.0% → −1.0%**.
 
The plateau is now unambiguous. From 1024 MB to 3008 MB — a 3× memory increase — end-to-end
latency reads **210 → 205 → 203 → 205 ms**. That is a 7 ms spread, or 3.3%, across four
memory sizes. Over that same range, cost rises **7.1×**.
 
The headline ratio: **23.5× the memory (128 → 3008 MB) bought 2.2× the speed.** Performance
plateaus. Cost does not.

<img width="2800" height="1800" alt="image" src="https://github.com/user-attachments/assets/a5c01aed-23ad-4260-973f-9fa641fc2033" />

 
### 2. The cost curve is U-shaped, and the right-hand wall is steep
 
This is the part the latency numbers alone hide:
 
- **256 MB is the cheapest** at $0.60 per million invocations — but it takes 141 ms.
- **1024 MB costs 1.46× more** ($0.87/M) and is **2.8× faster** (51 ms). That is the trade
  that pays: latency collapses, cost barely moves.
- **1536 MB** is fastest in isolation (46 ms) and has the **best tail of all seven sizes**
  (P99 271 ms, max 1,321 ms) — but end-to-end it is only 2.4% faster than 1024 MB for 34%
  more cost. Diminishing.
- **3008 MB costs 10.28×** the cheapest ($6.16/M) — and is **2.5× slower than 1024 MB**.
  You pay 7.1× more than 1024 MB to go *backwards*.
Under-provisioning (128 MB) is not even the cheapest option — at $0.81/M it costs 35% more
than 256 MB, because it runs long enough to erase its low per-ms rate. Both ends of the
curve are traps. The money is in the middle.
 
### 2b. The clearest pair in the dataset: 1536 MB vs 3008 MB
 
Two sizes produced an **identical 205 ms average**. Everything else about them differs:
 
| | 1536 MB | 3008 MB |
|---|---:|---:|
| Avg response time | 205 ms | 205 ms |
| P99 | **271 ms** | 282 ms |
| Slowest request | **1,321 ms** | 2,031 ms |
| Cost / 1M invocations | **$1.17** | $6.16 |
 
Same speed. **5.3× the cost. And a 54% worse tail.** There is no metric on which 3008 MB
wins. If a single row makes the argument for measuring rather than guessing, it is this one.
 
(The identical request count — 4,153 on both runs — is not a coincidence. In a closed-loop
test, equal latency produces equal throughput. See section 4.)
 
### 3. What over-provisioning actually costs
 
Lambda bills memory × duration, so once duration stops falling, every additional MB is
waste multiplied by every invocation:
 
| Monthly volume | 256 MB | 1024 MB | 3008 MB | Annual penalty of 3008 over 1024 |
|---|---:|---:|---:|---:|
| 10M invocations | $6.00/mo | $8.72/mo | $61.61/mo | **~$635/year** |
| 100M invocations | $59.95/mo | $87.25/mo | $616.10/mo | **~$6,346/year** |
 
At 100M invocations a month, "we set it to 3008 to be safe" costs roughly **$6,300 a year
more than 1024 MB, and delivers worse latency**. That is Lambda compute alone, for one
function.
 
Under-provisioning is visible — it is slow, someone complains, it gets fixed.
Over-provisioning is silent. It just shows up in the bill, every month, forever.
 
### 4. Throughput here is not independent evidence
 
A detail worth stating plainly, because it is easy to over-claim: this is a **closed-loop**
test. With a fixed pool of virtual users, throughput is mathematically bound to latency.
 
Multiplying throughput by average response time gives effective concurrency:
 
| Memory | Throughput × Avg latency | Effective concurrency |
|---|---|---:|
| 128 MB | 15.97 × 0.451 s | 7.20 |
| 256 MB | 23.42 × 0.305 s | 7.14 |
| 512 MB | 30.34 × 0.234 s | 7.10 |
| 1024 MB | 33.69 × 0.210 s | 7.07 |
| 1536 MB | 34.51 × 0.205 s | 7.07 |
| 2048 MB | 34.93 × 0.203 s | 7.09 |
| 3008 MB | 34.53 × 0.205 s | 7.08 |
 
Effective concurrency is essentially constant at ~7.1 across every run. The throughput
improvements are the arithmetic shadow of the latency improvements, not a separate result.
Reporting "throughput more than doubled!" alongside "latency more than halved!" would be
double-counting one finding.
 
This also means the test never saturated Lambda — it measured a fixed client concurrency
against a faster and faster backend. An open-model test would be needed to find the true
scaling ceiling.
 
### 5. The tail tells a different story from the average
 
Averages hide the shape of the distribution. Every run shows a max far above its P99:
 
- 128 MB: 451 ms average, **3,641 ms max** (~8× the average)
- 1024 MB: 210 ms average, **1,361 ms max** (~6.5× the average)
- 3008 MB: 205 ms average, **2,031 ms max** (~10× the average)
Those maxima are the cold-start signature. Note that 3008 MB has a *worse* max than
1024 MB despite a similar average — larger execution environments are not free to
initialise. Paying 7× more did not buy a better tail.
 
The customer who lands on that request does not experience your average. They experience
two seconds.
 
---
 
## Conclusion

Serverless removes server management, but it does not remove performance engineering.

A Lambda function should not be sized by default, guesswork or intuition.

It should be tuned using data.

The right Lambda configuration is not always the smallest or the largest. It is the configuration that best aligns cost, latency, throughput, reliability and business priority.

# Data Points
 
1. **The knee is around 1024 MB, and it is not arbitrary.** Single-threaded code saturates
   at one vCPU (1,769 MB). Below that, memory buys real speed.
   Above it, memory buys bill.
2. **Cheapest ≠ fastest ≠ best tail ≠ best value.** Cheapest is 256 MB. Fastest and best
   tail is 1536 MB. Best value is 1024 MB — 2.8× the speed of the cheapest for 1.46× the
   cost. Four different winners on four different axes.
3. **Default-and-forget is the expensive failure mode.** 3008 MB was the worst cost of
   every size tested (10.28× the cheapest) *and* slower than 1024 MB.
4. **Both ends of the curve are traps.** 128 MB is neither fast nor cheap.
5. **Measure the path, not just the function.** Power Tuning optimises the Lambda.
   Load testing shows what the caller actually experiences.

## Caveats and honest limits
 
- One Postman run per memory size. The four results above 1024 MB (210/205/203/205 ms) sit
  within a 3.3% band and should be treated as a plateau, not as a ranking — the ordering
  between them is inside plausible run-to-run variance. A production commitment should
  average several runs.
- Closed-loop load model with 10 VUs — this does not probe Lambda's concurrency ceiling.
- Cost figures are Lambda compute only, read from the Power Tuning chart and validated
  against AWS's published model. They exclude request charges, API Gateway and DynamoDB.
- The `list` operation is DynamoDB-I/O-bound as well as CPU-bound. A more CPU-heavy
  function would push the knee higher; a pure I/O-wait function would flatten it earlier.
- Power Tuning durations (function-only) and Postman durations (end-to-end) measure
  different things and are not directly comparable — they are reported separately above.

## Reproducing this
 
1. Deploy the serverless microservice (API Gateway → Lambda → DynamoDB).
2. Deploy `aws-lambda-power-tuning` from the AWS Serverless Application Repository.
3. Run the Step Functions state machine with
   `powerValues: [128, 256, 512, 1024, 1536, 2048, 3008]`.
4. For each memory size, run the Postman collection in Performance mode
   (10 VU, 2 min, ramp-up) and export the report.
5. Compare the exports. The knee is where your marginal improvement stops paying rent.
---
 
*Built and measured by Anu Agarwal — [linkedin.com/in/agarwalanu](https://www.linkedin.com/in/agarwalanu)*

<img width="732" height="56" alt="image" src="https://github.com/user-attachments/assets/6d6d2775-4fcf-45af-a872-aa3b19b7db72" />


