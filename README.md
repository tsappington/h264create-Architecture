# H264Create: Enterprise Video Conversion Pipeline

- **Role:** Lead Software Architect
- **Status:** Proprietary / Closed Source

---

## System Overview

H264Create is a production-grade video conversion pipeline designed for 24/7 broadcast and media operations. The system monitors incoming video streams from professional video recorders (PVRs), performs automated H.264 transcoding via FFmpeg, and extracts closed captions for accessibility compliance. Deployed across 15+ production instances with zero-downtime requirements, the platform has maintained 99.9%+ uptime over three years of continuous operation, processing thousands of video files daily.

---

## Tech Stack

| Layer | Technologies |
|-------|--------------|
| **Core Runtime** | Python 3.11+ |
| **Video Processing** | FFmpeg, CCExtractor |
| **File System Monitoring** | watchdog (cross-platform Observer pattern) |
| **Configuration** | Pydantic v2 (validation), TOML/INI (dual-format) |
| **Concurrency** | concurrent.futures.ThreadPoolExecutor |
| **Platform Integration** | Windows COM API (win32com), ctypes |
| **Observability** | Custom metrics, TimedRotatingFileHandler |
| **Data Analysis** | pandas, numpy |
| **Packaging** | PyInstaller (Windows executable distribution) |
| **Testing** | pytest, pytest-mock |

---

## Key Challenges Solved

### 1. Race-Condition-Free File Processing in Multi-Writer Environments

Video recorders continuously write to monitored directories while the system attempts to process completed files. Implemented a multi-layered file readiness detection system combining Windows COM API file-lock queries, file size stability checks, and modification time grace periods. This eliminated premature processing of incomplete files—a critical failure mode that previously caused truncated output and data loss.

### 2. Thread-Safe State Management Across 40+ Mutable Globals

The original architecture relied on 40+ global variables modified by concurrent threads, creating subtle race conditions and making the codebase difficult to reason about. Designed and implemented an `ApplicationState` container pattern using reentrant locks (RLock) with atomic operations, providing a single source of truth for mutable state while enabling gradual migration from the legacy architecture.

### 3. Zero-Downtime Configuration Migration (INI → TOML)

Migrated a fleet of 15 production instances from legacy INI configuration to type-safe TOML with Pydantic validation—without any service interruption. Implemented a dual-format loader with automatic fallback, SHA256 hash verification for migration integrity, and a 10-day phased rollout strategy with instant rollback capability (<60 seconds). The migration preserved full backward compatibility while adding 150+ validation rules.

### 4. Fault-Tolerant Alert System with Dead-Letter Queue

SMTP failures could silently drop critical operator alerts. Implemented a dead-letter queue pattern that persists failed alerts to disk, ensuring no notification is lost even during network outages. Combined with local logging as a first-class citizen, operators maintain full visibility into system health regardless of email infrastructure status.

---

## Architecture

### Data Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SOURCE DIRECTORIES                           │
│                    (1-2 monitored paths per instance)               │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     FILE SYSTEM OBSERVER                            │
│              (watchdog + FileSystemEventHandler)                    │
│                                                                     │
│   • Detects new/modified video files                                │
│   • Filters by extension (.ts, .wtv, .mp4)                          │
│   • Queues candidates for validation                                │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     VALIDATION LAYER                                │
│                                                                     │
│   • File-in-use detection (platform-specific)                       │
│   • Duration extraction and validation                              │
│   • Size stability verification                                     │
│   • Corruption detection via tolerance thresholds                   │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  PARALLEL PROCESSING ENGINE                         │
│               (ThreadPoolExecutor, configurable workers)            │
│                                                                     │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│   │  Worker 1   │    │  Worker 2   │    │  Worker N   │             │
│   │  (FFmpeg)   │    │  (FFmpeg)   │    │  (FFmpeg)   │             │
│   └─────────────┘    └─────────────┘    └─────────────┘             │
│                                                                     │
│   • H.264 transcoding with configurable output parameters           │
│   • Progress tracking via segment dot-files                         │
│   • Error capture and retry logic                                   │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 OPTIONAL: CAPTION EXTRACTION                        │
│                      (CCExtractor subprocess)                       │
│                                                                     │
│   • Closed caption extraction to .bin format                        │
│   • Process lifecycle management (zombie prevention)                │
│   • Exponential backoff retry on failures                           │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      OUTPUT DIRECTORY                               │
│                                                                     │
│   • H.264 MP4 files                                                 │
│   • Optional .bin caption files                                     │
│   • Progress dot-files for external monitoring                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Control Plane

```
┌─────────────────────────────────────────────────────────────────────┐
│                     MONITORING SUBSYSTEM                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   DISK SPACE MONITOR          SINGLETON IPC           ALERTING      │
│   ┌─────────────────┐        ┌──────────────┐       ┌────────────┐  │
│   │ Threshold-based │        │ Socket-based │       │ HTML Email │  │
│   │ capacity alerts │        │ external     │       │ Dead-letter│  │
│   │ (configurable)  │        │ control API  │       │ queue      │  │
│   └─────────────────┘        └──────────────┘       └────────────┘  │
│                                                                     │
│   Commands: ping | pause | resume | stats | errors                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Design Patterns Employed

| Pattern | Application |
|---------|-------------|
| **Observer** | File system monitoring with event-driven processing |
| **Factory Method** | Configuration loading with warning collection |
| **Singleton** | Port-based IPC for single-instance enforcement |
| **Thread Pool** | Bounded parallelism for resource management |
| **State Container** | Centralized, thread-safe mutable state |
| **Dead-Letter Queue** | Fault-tolerant alert delivery |
| **Strangler Fig** | Incremental monolith-to-modular migration |
| **Feature Flag** | Gradual rollout of new subsystems |

---

## Deployment Strategy

The system employs a risk-aware deployment methodology suitable for 24/7 critical infrastructure:

- **Canary Deployment:** Single low-risk instance validates changes for 48 hours
- **Staged Rollout:** Progressive deployment (25% → 50% → 75% → 100%) over 10 days
- **Shadow Mode Testing:** New code paths execute in parallel with production for behavioral equivalence validation
- **Instant Rollback:** Automated scripts restore previous state in under 60 seconds
- **Hash Verification:** SHA256 checksums prevent configuration drift between deployment phases

---

*Note: This repository serves as an architectural overview. The source code is proprietary and not publicly available.*
