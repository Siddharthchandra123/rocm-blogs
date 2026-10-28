# A 192 GiB GPU, a 0 GB Dashboard, and One Wrong Divisor

*An AMD SMI integration bug I encountered while building AI Factory Mission Control*

I've been developing **AMD AI Factory Mission Control**, an independent platform for GPU inference scheduling, workload orchestration, and observability.

The system combines FastAPI, Redis, PostgreSQL, ROCm, vLLM, AMD SMI, Prometheus, and Grafana into a GPU inference control plane.

I initially developed it using simulated AMD Instinct workers, then connected it to a real **AMD Instinct MI300X** to validate inference execution.

During development, I discovered something interesting: even when the inference pipeline worked correctly, the monitoring layer could still report completely incorrect hardware information.

And the cause was just one divisor.

## The problem

Someone reviewing my public repository noticed something in `agent/amdsmi_probe.py`.

I was retrieving GPU VRAM information using AMD SMI's `amdsmi_get_gpu_vram_usage()` API.

But my implementation contained:

```python
'memory_total_gb': round(total / 2**30, 3),
'memory_used_gb': round(used / 2**30, 3),
```

That looks reasonable if `total` and `used` represent bytes.

But they don't.

The AMD SMI underlying data structure documents these VRAM fields in **MB**.

My code was effectively converting a value that was already expressed in megabytes as though it were bytes.

## Why this matters

Consider an illustrative raw memory value of 196,608, representing approximately 192 GiB in the convention used by my application.

With my original code:

`196608 / 2**30 = 0.000183`

After rounding to three decimal places:

**0.000 GB**

The dashboard could confidently suggest that a GPU with enormous memory capacity had effectively no memory.

The corrected calculation is:

`196608 / 1024 = 192`

This illustrates why telemetry normalization matters.

A monitoring system can successfully connect to hardware, receive valid API responses, and render beautiful dashboards while still showing fundamentally incorrect information.

## The correction

I updated the AMD SMI VRAM conversion to:

```python
'memory_total_gb': round(total / 1024, 3)
    if total is not None else None,
'memory_used_gb': round(used / 1024, 3)
    if used is not None else None,
```

I also reviewed the test fixtures because mocking the API with byte-based values can reproduce the same misunderstanding inside the tests.

The local automated test suite now passes.

However, the real MI300X instance is currently offline, so I haven't yet repeated the corrected telemetry measurement against physical hardware.

**This was an application integration bug, not a confirmed ROCm or AMD SMI defect.**

## What else did I learn while building Mission Control?

The more interesting challenge was not just reading GPU metrics. It was understanding the complete inference request lifecycle.

In one real MI300X benchmark, all 16 inference jobs completed successfully.

During an earlier eight-job comparison, I also observed the following after optimizing the orchestration pipeline:

| Metric | Before | After |
|---|---:|---:|
| p95 queue waiting | 11.283 s | 4.675 s |
| p95 end-to-end latency | 12.737 s | 5.826 s |
| Client-observed throughput | 49.607 tok/s | 69.661 tok/s |

That's approximately a **54.3% observed reduction in p95 end-to-end latency**.

The improvement came from optimizing my own scheduling pipeline, not from the AMD SMI conversion fix.

These were limited before-and-after measurements on my own system, not proof of superiority over established GPU schedulers.

## Five engineering lessons

1. **Hardware telemetry needs explicit units.** Every metric should define its raw units, normalized units, and expected range.

2. **Tests can preserve the same mistake as production code.** If mocks assume the wrong unit, passing tests can create false confidence.

3. **Observability is only as trustworthy as its data pipeline.** A valid dashboard does not guarantee valid measurements.

4. **Scheduling and monitoring should be decoupled.** Telemetry polling should not block inference dispatch.

5. **Sharing code exposes problems that benchmarks alone might miss.** The feedback came from someone who inspected the implementation instead of relying on reported results.

## What's next?

I'm currently exploring adaptive GPU inference concurrency without Kubernetes.

Instead of using a fixed number of active inference slots, I want Mission Control to adjust concurrency using signals such as:

- vLLM waiting and running requests
- KV-cache utilization
- Controller queue pressure
- Inference latency
- Worker capacity and health

I'll first validate the controller using simulated telemetry, then compare its behavior on actual GPU infrastructure when available.

I'm also interested in contributing a focused ROCm/vLLM observability example or an AMD SMI documentation improvement to the AMD developer ecosystem.

The biggest takeaway from this experience is simple:

**A sophisticated scheduling system can still make poor decisions if its basic hardware metrics are wrong.**

And sometimes the most valuable contribution to a project isn't a new algorithm. It's someone taking the time to read the code and point out a mistake.

---

**References**

- [AMD SMI Python API Reference](https://rocm.docs.amd.com/projects/amdsmi/en/latest/reference/amdsmi-py-api.html)
- [AMD SMI source header](https://github.com/ROCm/amdsmi/blob/amd-mainline/include/amd_smi/amdsmi.h)
- **Project repository:** [Insert your GitHub repository URL]

*AMD AI Factory Mission Control is an independent engineering project, not an official AMD product.*
