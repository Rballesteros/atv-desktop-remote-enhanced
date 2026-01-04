# Python Backend Bug Report

This document tracks bugs found in the Python WebSocket server (`build/wsserver.py`). These bugs should be addressed in a future release.

---

## CRITICAL BUGS

### Bug #1: Direct Dictionary Access Without Error Handling
**Severity:** CRITICAL
**Lines:** 281, 295-296, 364-365
**File:** build/wsserver.py

**Description:**
The code directly accesses dictionary keys without validation, which will crash with `KeyError` if a malformed WebSocket message is sent.

**Locations:**
- Line 281: `text = data["text"]` (settext command)
- Lines 295-296: `id = data["identifier"]`, `creds = data["credentials"]` (connect command)
- Lines 364-365: `key = data['key']`, `taction = InputAction[data['taction']]` (key command)

**Impact:** Application crash when receiving malformed messages

**Suggested Fix:**
```python
# Line 281
if cmd == "settext":
    if "text" not in data:
        await sendCommand(websocket, "error", "Missing 'text' field")
        return
    text = data["text"]
    # ... rest of code

# Lines 295-296
if cmd == "connect":
    if "identifier" not in data or "credentials" not in data:
        await sendCommand(websocket, "error", "Missing required fields")
        return
    id = data["identifier"]
    creds = data["credentials"]

# Lines 364-365
if not isinstance(data, str):
    if not isinstance(data, dict) or 'key' not in data or 'taction' not in data:
        await sendCommand(websocket, "error", "Invalid key command format")
        return
    key = data['key']
    try:
        taction = InputAction[data['taction']]
    except KeyError:
        await sendCommand(websocket, "error", f"Unknown action: {data['taction']}")
        return
```

---

### Bug #2: Missing Null Check Before Active Device Access
**Severity:** CRITICAL
**Lines:** 282, 288, 290
**File:** build/wsserver.py

**Description:**
The `settext` and `gettext` command handlers access `active_device.keyboard` without checking if `active_device` exists. Since `active_device` is initialized to `False`, this will crash with `AttributeError: 'bool' object has no attribute 'keyboard'`.

**Code:**
```python
if cmd == "settext":
    text = data["text"]
    if active_device.keyboard.text_focus_state != pyatv_const.KeyboardFocusState.Focused:  # active_device could be False!
        return
```

**Impact:** Application crash when commands are sent before device connection

**Suggested Fix:**
```python
if cmd == "settext":
    if not active_device:
        await sendCommand(websocket, "error", "Device not connected")
        return
    text = data.get("text")
    if not text:
        return
    if active_device.keyboard.text_focus_state != pyatv_const.KeyboardFocusState.Focused:
        return
    await active_device.keyboard.text_set(text)
```

---

## HIGH PRIORITY BUGS

### Bug #4: Bare Exception Handler
**Severity:** HIGH
**Line:** 136
**File:** build/wsserver.py

**Description:**
Bare `except:` clause swallows all exceptions silently, including `KeyboardInterrupt` and system exits, making debugging impossible.

**Code:**
```python
except:
    pass
```

**Impact:** Makes debugging connection issues extremely difficult

**Suggested Fix:**
```python
except Exception as e:
    print(f"Error closing device during reconnection: {e}")
    pass
```

---

### Bug #5: Logic Error in Command-Line Argument Parsing
**Severity:** HIGH
**Lines:** 468-470
**File:** build/wsserver.py

**Description:**
If user passes `-h` for help, the code prints help but then still tries `int("-h")` causing a `ValueError`.

**Code:**
```python
if len(args) > 0:
    if args[0] in ["-h", "--help", "-?", "/?"]:
        print ("Usage: %s (port_number)\n\n Port number by default is %d" % (my_name, default_port))
    port = int(args[0])  # BUG: Still tries to convert help flag to int!
```

**Impact:** Crash when user requests help

**Suggested Fix:**
```python
if len(args) > 0:
    if args[0] in ["-h", "--help", "-?", "/?"]:
        print ("Usage: %s (port_number)\n\n Port number by default is %d" % (my_name, default_port))
        return  # Exit early
    try:
        port = int(args[0])
    except ValueError:
        print(f"Error: Invalid port number '{args[0]}'")
        port = default_port
```

---

### Bug #7: Missing Pairing State Validation
**Severity:** HIGH
**Lines:** 254-256
**File:** build/wsserver.py

**Description:**
Code always accesses `pairing.service.credentials` even when `pairing.has_paired` is False, which may cause an `AttributeError`.

**Code:**
```python
if pairing.has_paired:
    print("Paired with device!")
    print("Credentials:", pairing.service.credentials)
else:
    print("Did not pair with device!")
pairing_creds["Companion"] = pairing.service.credentials  # BUG: Always executes!
```

**Impact:** Potential crash when pairing fails

**Suggested Fix:**
```python
if pairing.has_paired:
    print("Paired with device!")
    print("Credentials:", pairing.service.credentials)
    pairing_creds["Companion"] = pairing.service.credentials
    await sendCommand(websocket, "pairCredentials", pairing_creds)
else:
    print("Did not pair with device!")
    await sendCommand(websocket, "pairCredentials_failed", "Pairing unsuccessful")
```

---

## MEDIUM PRIORITY BUGS

### Bug #9: Incorrect Keyboard State Check
**Severity:** MEDIUM
**Line:** 363
**File:** build/wsserver.py

**Description:**
When `data` is a dict, the code doesn't validate that 'key' and 'taction' fields exist before accessing them.

**Code:**
```python
taction = False
key = data
if not isinstance(data, str):
    key = data['key']  # BUG: Could KeyError if 'key' not in data
    taction = InputAction[data['taction']]  # BUG: Could KeyError if 'taction' not in data
```

**Impact:** Crash on malformed key command

**Suggested Fix:**
See Bug #1 for the fix (same issue)

---

### Bug #10: Global State Modified Without Synchronization
**Severity:** MEDIUM
**Lines:** Throughout (37-42, 62-87, 118-168, 171-256)
**File:** build/wsserver.py

**Description:**
Multiple global variables (`active_device`, `active_remote`, `active_ws`, `pairing_atv`, `active_pairing`, `pairing_creds`) are modified in async/concurrent contexts without locks. If multiple WebSocket clients connect simultaneously, or if reconnection happens during pairing, state can become inconsistent.

**Example Race Condition:**
1. Connection handler starts at line 318: `device = await pyatv.connect(...)`
2. Simultaneously, reconnection handler at line 143: `device = await pyatv.connect(...)`
3. Both write to `active_device` variable simultaneously
4. One device object overwrites the other, causing resource leak

**Impact:** State corruption, resource leaks, connection failures

**Suggested Fix:**
```python
import asyncio

# At module level
device_lock = asyncio.Lock()

# In handler
async with device_lock:
    if active_device:
        try:
            await active_device.close()
        except:
            pass
    active_device = device
    active_remote = remote
```

---

### Bug #13: Missing Null Check for Pairing ATV
**Severity:** MEDIUM
**Lines:** 234, 270
**File:** build/wsserver.py

**Description:**
If pairing fails or is interrupted, `pairing_atv` could be `False`, causing `AttributeError` when accessing `pairing_atv.identifier`.

**Code:**
```python
id = pairing_atv.identifier  # BUG: pairing_atv could be False
```

**Impact:** Crash during pairing

**Suggested Fix:**
```python
if not pairing_atv:
    await sendCommand(websocket, "error", "No ATV selected for pairing")
    return
id = pairing_atv.identifier
```

---

### Bug #15: Unsafe Global Variable Assignment
**Severity:** MEDIUM
**Line:** 172
**File:** build/wsserver.py

**Description:**
Line 172 unconditionally sets `active_ws = websocket`. If multiple WebSocket clients connect, only the last one is remembered, breaking the ability to send messages to other clients.

**Code:**
```python
async def parseRequest(j, websocket):
    global scan_lookup, pairing_atv, active_pairing, active_device, active_remote, active_ws, pairing_creds
    active_ws = websocket  # BUG: Sets global without checking if already set
```

**Impact:** Lost ability to communicate with multiple clients

**Suggested Fix:**
```python
# Handle multiple concurrent WebSocket connections
connected_websockets = set()

async def parseRequest(j, websocket):
    global connected_websockets, active_ws
    connected_websockets.add(websocket)
    # ... handle request ...
```

---

## Summary

**Total Bugs Found:** 11 Python backend bugs

**By Severity:**
- Critical: 2 bugs (will crash the app)
- High: 3 bugs (major issues)
- Medium: 6 bugs (should be fixed)

**Recommendation:**
These bugs should be addressed in v1.5.0 release. The CRITICAL bugs (#1, #2) should be prioritized as they can cause application crashes with malformed input or incorrect call sequences.

**Note:** The Python WebSocket server is located in `build/wsserver.py`, which appears to be a compiled/packaged version. The source may be in a different location that should be fixed instead.
