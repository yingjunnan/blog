---
title: 苹果 M1 到 M5 全系列芯片性能对比
published: 2026-03-09
description: '基于 Apple 官方技术规格和官方测试口径，对比 M1 到 M5 在 CPU、GPU、神经网络引擎和功耗效率上的差异，并给出选购建议。'
image: ''
tags: ['Apple', 'M1', 'M2', 'M3', 'M4', 'M5', '芯片', '性能对比']
category: '硬件'
draft: false
lang: 'zh'
---

# 苹果 M1 到 M5 全系列芯片性能对比

本文所有参数均来自 Apple 官方 Newsroom 或 Apple Support 技术规格页面；性能对比均按 Apple 官方 “up to” 测试口径整理。

## 一图看懂：核心参数总表（M1-M5）

| 芯片 | 首发日期 | 制程 | 晶体管数量 | CPU | GPU | 神经网络引擎 | 统一内存带宽 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| M1 | 2020-11-10 | 5nm | 160 亿 | 8 核（4P+4E） | 7/8 核 | 16 核，11 TOPS | 官方未在 M1 页面给出明确 GB/s |
| M2 | 2022-06-06 | 第二代 5nm | 200 亿 | 8 核（4P+4E） | 最多 10 核 | 16 核，15.8 TOPS | 100 GB/s |
| M3 | 2023-10-30 | 3nm | 250 亿 | 8 核（4P+4E） | 10 核 | 16 核（官方称较 M1 快最多 60%） | 100 GB/s |
| M4 | 2024-05-07 | 第二代 3nm | 280 亿 | 最多 10 核（4P+6E） | 10 核 | 16 核，最高 38 TOPS | 120 GB/s |
| M5 | 2025-10-15 | 第三代 3nm | 官方未披露 | 最多 10 核（4P+6E） | 10 核（含 Neural Accelerators） | 16 核（官方未公布 TOPS） | 153 GB/s |

## Geekbench 6 跑分对比（单核 / 多核 / Metal）

| 芯片 | 代表机型（Geekbench Browser） | 单核分数 | 多核分数 | Metal GPU 分数 |
| --- | --- | --- | --- | --- |
| M1 | [MacBook Air (Late 2020)](https://browser.geekbench.com/macs/macbook-air-late-2020) | 2347 | 8342 | 33156 |
| M2 | [MacBook Air (2022)](https://browser.geekbench.com/macs/macbook-air-2022) | 2587 | 9669 | 42206 |
| M3 | [MacBook Pro (14-inch, Nov 2023, M3)](https://browser.geekbench.com/macs/macbook-pro-14-inch-nov-2023-8c-cpu-10c-gpu) | 3077 | 11537 | 48161 |
| M4 | [MacBook Pro (14-inch, 2024, M4)](https://browser.geekbench.com/macs/macbook-pro-14-inch-2024-10c-cpu) | 3753 | 14912 | 57386 |
| M5 | [MacBook Pro (14-inch, 2025, M5)](https://browser.geekbench.com/macs/macbook-pro-14-inch-2025) | 4228 | 17458 | 75673 |

> 说明：以上分数为 Geekbench Browser 的 Geekbench 6 用户提交平均分（抓取日期：2026-03-09）；不同机型散热、系统版本和功耗策略会带来一定波动。

## CPU 性能对比（官方口径）

| 芯片 | 官方 CPU 性能信息 |
| --- | --- |
| M1 | 相比上一代 Mac，CPU 最高可达 3.5x。 |
| M2 | 多核 CPU 性能较 M1 提升最高 18%。 |
| M3 | CPU 性能较 M1 提升最高 35%。 |
| M4 | 相比 iPad Pro 的 M2，CPU 性能最高 1.5x。 |
| M5 | 相比 M4，多线程 CPU 性能最高提升 15%。 |

配图说明：Apple 官方 M4 CPU 核心架构图（4P+6E）。
图片链接：[Apple M4 CPU 架构图](https://www.apple.com/newsroom/images/2024/05/apple-introduces-m4-chip/article/Apple-M4-chip-new-CPU-240507_big.jpg.large.jpg)

## GPU 性能对比（官方口径）

| 芯片 | 官方 GPU 性能信息 |
| --- | --- |
| M1 | 相比上一代 Mac，GPU 最高可达 6x；峰值吞吐量 2.6 TFLOPS。 |
| M2 | 相比 M1，GPU 在同功耗下最高快 25%，满功耗最高快 35%。 |
| M3 | 图形性能较 M1 最高快 65%；同等性能下接近 M1 一半功耗。 |
| M4 | 相比 M2，Octane 等专业渲染场景最高可达 4x。 |
| M5 | 相比 M4，图形性能最高快 30%；光追场景最高快 45%；AI 相关峰值 GPU 计算最高超 4x（对比 M4）。 |

配图说明：Apple 官方 M4 GPU 架构图（10 核 GPU，含 Dynamic Caching / Mesh Shading / Ray Tracing）。
图片链接：[Apple M4 GPU 架构图](https://www.apple.com/newsroom/images/2024/05/apple-introduces-m4-chip/article/Apple-M4-chip-10-core-CPU-240507_big.jpg.large.jpg)

## 神经网络引擎（NPU/AI）对比

| 芯片 | 神经网络引擎对比 |
| --- | --- |
| M1 | 16 核 Neural Engine，11 TOPS。 |
| M2 | 16 核 Neural Engine，15.8 TOPS（较 M1 提升超 40%）。 |
| M3 | 16 核 Neural Engine，官方称较 M1 家族最高快 60%。 |
| M4 | 16 核 Neural Engine，最高 38 TOPS。 |
| M5 | 更快的 16 核 Neural Engine + 每个 GPU 核心内置 Neural Accelerator，官方未给出 TOPS 数值。 |

配图说明：Apple 官方 M4 Neural Engine 架构图（16 核 Neural Engine）。
图片链接：[Apple M4 Neural Engine 架构图](https://www.apple.com/newsroom/images/2024/05/apple-introduces-m4-chip/article/Apple-M4-chip-Neural-Engine-240507_big.jpg.large.jpg)

## 功耗效率对比（官方口径）

| 芯片 | 官方能效信息 |
| --- | --- |
| M1 | 高能效核心可在约十分之一功耗下提供接近当时双核 MacBook Air 的性能。 |
| M2 | CPU 可在 1/4 功耗下达到竞品峰值性能；GPU 可在 1/5 功耗下达到竞品峰值性能。 |
| M3 | CPU 在“与 M1 相同多线程性能”下可用约 1/2 功耗；GPU 在“与 M1 相同性能”下接近 1/2 功耗。 |
| M4 | 相同性能下相比 M2 约 1/2 功耗；相比当时轻薄本 PC 芯片约 1/4 功耗。 |
| M5 | 官方强调“industry-leading power-efficient performance”，但未公开具体功耗倍率。 |

配图说明：Apple 官方 M1 CPU performance vs. power 图（能效曲线示意）。
图片链接：[Apple M1 CPU Performance vs. Power 图](https://www.apple.com/newsroom/images/product/mac/standard/Apple_m1-chip-cpu-power-chart_11102020_big.jpg.large.jpg)

## 适用场景推荐

| 芯片 | 更推荐的人群与场景 |
| --- | --- |
| M1 | 日常办公、轻度开发、轻量修图剪辑、学生党入门。 |
| M2 | 中轻度创作（PS/LR/1080P-4K 入门剪辑）、多任务办公。 |
| M3 | 需要硬件光追、图形工作负载更重、同时兼顾续航的移动办公。 |
| M4 | 更高强度创作与 AI 推理（本地模型、视频渲染、复杂多任务）。 |
| M5 | 重度本地 AI 工作流、3D/视频专业流程、希望更长生命周期的高性能用户。 |

## 选购结论（2026 版）

1. 预算敏感且以日常生产力为主，M1/M2 仍然够用。
2. 追求“性能/功耗平衡”，M3 仍是非常稳妥的一代。
3. 以 AI 与专业图形渲染为核心，优先看 M4/M5。
4. 若你最在意未来 3-5 年余量，M5 的内存带宽（153GB/s）和 GPU AI 架构升级更有前瞻性。

## 参考来源（官方）

1. Apple Newsroom: [Apple unleashes M1](https://www.apple.com/newsroom/2020/11/apple-unleashes-m1/)
2. Apple Newsroom: [Apple unveils M2](https://www.apple.com/newsroom/2022/06/apple-unveils-m2-with-breakthrough-performance-and-capabilities/)
3. Apple Newsroom: [Apple unveils M3, M3 Pro, and M3 Max](https://www.apple.com/newsroom/2023/10/apple-unveils-m3-m3-pro-and-m3-max-the-most-advanced-chips-for-a-personal-computer/)
4. Apple Newsroom: [Apple introduces M4 chip](https://www.apple.com/newsroom/2024/05/apple-introduces-m4-chip/)
5. Apple Newsroom: [Apple unleashes M5](https://www.apple.com/ml/newsroom/2025/10/apple-unleashes-m5-the-next-big-leap-in-ai-performance-for-apple-silicon/)
6. Apple Support: [MacBook Air (M2, 2022) - Tech Specs](https://support.apple.com/en-us/111867)
7. Apple Support: [MacBook Pro (14-inch, M3, Nov 2023) - Tech Specs](https://support.apple.com/en-mide/117735)
8. Apple Support: [iPad Pro 11-inch (M4) - Tech Specs](https://support.apple.com/en-us/119892)
9. Apple Support: [iPad Pro 11-inch (M5) - Tech Specs](https://support.apple.com/en-afri/125406)
10. Apple Support: [MacBook Pro (13-inch, M1, 2020) - Technical Specifications](https://support.apple.com/en-us/111893)
11. Geekbench Browser: [MacBook Air (Late 2020) Benchmarks (Apple M1)](https://browser.geekbench.com/macs/macbook-air-late-2020)
12. Geekbench Browser: [MacBook Air (2022) Benchmarks (Apple M2)](https://browser.geekbench.com/macs/macbook-air-2022)
13. Geekbench Browser: [MacBook Pro (14-inch, Nov 2023, Apple M3) Benchmarks](https://browser.geekbench.com/macs/macbook-pro-14-inch-nov-2023-8c-cpu-10c-gpu)
14. Geekbench Browser: [MacBook Pro (14-inch, 2024, Apple M4) Benchmarks](https://browser.geekbench.com/macs/macbook-pro-14-inch-2024-10c-cpu)
15. Geekbench Browser: [MacBook Pro (14-inch, 2025, Apple M5) Benchmarks](https://browser.geekbench.com/macs/macbook-pro-14-inch-2025)

> 说明：不同代际官方“up to”数据的测试机型和软件版本并不完全一致，适合看趋势和代际差异，不建议机械换算为统一绝对跑分。
