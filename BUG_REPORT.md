# State-Related Bugs Found

## Critical Bugs

### 1. **CRITICAL: `atv_connected` Variable Duplication/Mismatch**

**Location**: `ws_remote.js:24` and `web_remote.js:485,501`

**Problem**:
- `atv_connected` is declared as a global variable in `ws_remote.js` (line 24)
- `web_remote.js` tries to use `atv_connected` but it's a different variable (not imported/exported)
- This creates TWO separate `atv_connected` variables that can be out of sync

**Code**:
```javascript
// ws_remote.js:24
var atv_connected = false;

// web_remote.js:485
function isConnected() {
    return atv_connected  // ❌ This references a different variable!
}

// web_remote.js:501
atv_connected = false;  // ❌ This sets a different variable!
```

**Impact**: High - Connection state can be incorrect, causing UI to show wrong status

**Fix**: Export `atv_connected` from `ws_remote.js` or use the event emitter properly

---

### 2. **State Not Reset in `connectToATV()` Error Path**

**Location**: `web_remote.js:697-710`

**Problem**:
```javascript
async function connectToATV() {
    if (connecting) return;
    connecting = true;
    // ...
    await ws_connect(atv_credentials);  // ❌ If this throws, connecting stays true forever
    // ...
    connecting = false;
}
```

**Impact**: Medium - If connection fails with exception, `connecting` flag stays `true` forever, preventing future connection attempts

**Fix**: Use try/finally
```javascript
async function connectToATV() {
    if (connecting) return;
    connecting = true;
    try {
        setStatus("Connecting to ATV...");
        // ... rest of code
    } finally {
        connecting = false;
    }
}
```

---

### 3. **`ws_connecting` State Not Reset on Connection Timeout**

**Location**: `ws_remote.js:209-238`

**Problem**:
```javascript
function ws_connect(creds) {
    if (ws_connecting) return;  // ❌ Guards against re-entry
    ws_start_tm = Date.now();
    ws_connecting = true;
    // ...
    const connectionTimeout = setTimeout(() => {
        ws_connecting = false;  // ✅ Reset here
        ws_start_tm = false;
        // ...
    }, 10000);

    sendMessage("connect", creds)
    atv_events.once("connected", ic => {
        clearTimeout(connectionTimeout);
        ws_connecting = false;  // ✅ Reset here
        // ...
    });
}
```

Actually this one looks OK on closer inspection - both paths reset it.

---

### 4. **`ws_timeout` Can Leak if Multiple Reconnects Triggered**

**Location**: `ws_remote.js:68-86`

**Problem**:
```javascript
function reconnect() {
    if (ws_timeout) return;  // ❌ Only checks if timeout exists
    ws_timeout = setTimeout(() => {
        ws_timeout = false;
        // ...
    }, ws_timeout_interval)
}
```

If reconnect() is called while a timeout is pending, it returns early and doesn't create a new timeout. This could be intentional, but the state could get stuck.

**Impact**: Low - Might prevent reconnections in some edge cases

---

### 5. **`scanWhenOpen` Not Reset After Scan**

**Location**: `ws_remote.js:189-195`

**Problem**:
```javascript
function ws_startScan() {
    connection_failure = false;
    if (ws_connected) sendMessage("scan");
    else {
        scanWhenOpen = true;  // ❌ Set to true but never reset
    }
}

// In startWebsocket():
ws.once('open', function open() {
    ws_connected = true;
    if (scanWhenOpen) ws_startScan();  // Sends scan but doesn't reset flag
    // ...
})
```

**Impact**: Low - Scan might be sent multiple times on subsequent connections

**Fix**: Reset the flag after using it
```javascript
if (scanWhenOpen) {
    scanWhenOpen = false;
    ws_startScan();
}
```

---

### 6. **`connection_failure` Flag Inconsistently Managed**

**Location**: Multiple files

**Problem**: `connection_failure` is set to false in many places but there's no clear ownership of this state:
- `ws_remote.js:23` - declared
- `ws_remote.js:136, 147, 181, 190, 219, 231, 242, 249, 255, 261` - modified

It's being treated as a global flag but there's no clear state machine for it.

**Impact**: Medium - Connection failure state might be incorrect

---

### 7. **`pending` Array Never Cleared on Connection Failure**

**Location**: `ws_remote.js:30-46`

**Problem**:
```javascript
var pending = []

function sendMessage(command, data) {
    if (!ws) {
        pending.push([command, data]);  // ❌ Commands queued
        return;
    }
    while (pending.length > 0) {  // Sent when ws reconnects
        var cmd_ar = pending.shift();
        ws.send(JSON.stringify({ cmd: cmd_ar[0], data: cmd_ar[1] }))
    }
    // ...
}
```

If websocket never connects, pending array grows forever.

**Impact**: Low-Medium - Memory leak if many commands sent while disconnected

**Fix**: Add a max size check or clear pending on permanent failure

---

### 8. **`pairDevice` and `ws_pairDevice` Confusion**

**Location**: Multiple

**Problem**:
- `web_remote.js:3` - `var pairDevice = ""`
- `ws_remote.js:25` - `var ws_pairDevice = ""`
- `ws_remote.js:131` - Uses `pairDevice` (from web_remote.js) instead of `ws_pairDevice`

```javascript
// ws_remote.js:131
localStorage.setItem('atvcreds', JSON.stringify(getCreds(pairDevice)));
                                                          // ^^^^^^^^^^
                                                          // Wrong variable!
```

This references the wrong variable from a different file.

**Impact**: High - Pairing credentials might be saved for wrong device

---

### 9. **`device` Object Never Reset**

**Location**: `web_remote.js:31`

**Problem**:
```javascript
var device = false;

function handleMessage(msg) {
    if (!device) {  // ✅ Good null check
        return;
    }
    device.lastMessages.push(...)  // Uses device
}
```

But `device` is never actually set or reset anywhere in the code. This looks like legacy code from an old implementation.

**Impact**: Low - Dead code, but confusing

---

### 10. **`playstate` Never Updated** ✅ FIXED

**Location**: `web_remote.js:33`

**Problem**:
```javascript
var playstate = false;
```

This variable is declared but never used or updated anywhere in the codebase.

**Impact**: None - Dead code

**Fix Applied**: Variable removed and replaced with `currentlyPlaying` state variable that properly tracks play/pause state and updates button icon

---

## Recommendations

### Immediate Fixes (High Priority):

1. **Fix `atv_connected` synchronization**
   - Export it properly from `ws_remote.js`
   - Import it in `web_remote.js`
   - OR: Use only the event emitter pattern

2. **Add try/finally to `connectToATV()`**
   - Ensures `connecting` flag is always reset

3. **Fix `pairDevice` vs `ws_pairDevice` confusion**
   - Use consistent variable naming
   - Pass credentials properly

### Medium Priority:

4. **Add max size to `pending` array**
5. **Reset `scanWhenOpen` after use**
6. **Clean up dead code** (`device`, `playstate`)

### Low Priority:

7. **Add state machine** for connection states instead of multiple boolean flags
8. **Add logging** for state transitions to help debugging

---

## Suggested State Machine

Instead of multiple boolean flags, consider:

```javascript
const ConnectionState = {
    DISCONNECTED: 'disconnected',
    CONNECTING: 'connecting',
    CONNECTED: 'connected',
    FAILED: 'failed',
    RECONNECTING: 'reconnecting'
};

var connectionState = ConnectionState.DISCONNECTED;
```

This makes state transitions clearer and prevents invalid state combinations.
