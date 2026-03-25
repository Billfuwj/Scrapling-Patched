# Scrapling Patched Version - Changelog

> **Fork of [D4Vinci/Scrapling](https://github.com/D4Vinci/Scrapling)** (BSD-3-Clause License)
> Original Author: Karim Shoair (karim.shoair@pm.me)

## Patches Applied

### P0 Fix: Infinite Loop Protection

**Problem**: Original code contains multiple `while True` / `while <condition>` loops without exit conditions, which can cause the program to hang indefinitely.

**Files Modified**:

#### 1. `scrapling/engines/toolbelt/convertor.py`
- `_get_page_content()`: `while True` → `for attempt in range(20)` (max ~10s)
- `_get_async_page_content()`: Same fix for async version

#### 2. `scrapling/engines/_browsers/_stealth.py`
- `_cloudflare_solver()` sync version: 3 `while` loops added max attempt limits (60/40/20)
- `_cloudflare_solver()` async version: Same fixes applied

**Total**: 4 infinite loops fixed, all now have timeout/max-retry guards with logging.
