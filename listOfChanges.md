# CHANGES.md

## 🚀 Multi-gNB support and simulator improvements

These are all the exact changes made from `originalfile.txt` to `newerSimulator.txt` to add support for **multiple gNBs**, refactor config parsing, and safely handle connection logic.

---

### 1. 🧠 Updated `RFSIMULATOR_PARAMS_DESC` for multiple IPs

**Original:**
```c
{"serveraddr", "<ip address to connect to>\n", simOpt,  .strptr=&rfsimulator->ip, ... TYPE_STRING, 0 },
```

**Updated:**
```c
static char **tempIpListPtr = NULL;

...

{"serveraddr", "<comma-separated list of IPs to connect to>\n", simOpt, .strlistptr=tempIpListPtr, ... TYPE_STRINGLIST, 0 },
```

🔍 This replaces the single `ip` pointer with a string list to allow multiple gNB IPs via comma-separated input.

---

### 2. 📦 New fields in `rfsimulator_state_t`

**Original:**
```c
typedef struct {
  char *ip;
  ...
} rfsimulator_state_t;
```

**Updated:**
```c
typedef struct {
  char **ipList;
  int ipListCount;
  char *ip;
  ...
} rfsimulator_state_t;
```

📥 `ipList` holds the list of parsed IPs, and `ipListCount` tells us how many there are.

---

### 3. 🛠 Rewrote `rfsimulator_readconfig()` logic

**Added:**
- Parses `--rfsimulator.serveraddr` as a string list using `strlistptr`
- Adds fallback support for the `RFSIMULATOR` environment variable
- Uses `strncasecmp()` with null checks for safety
- Sets role to SERVER only if IP starts with `server` or `enb`, else CLIENT

🔐 **Null-safe and fault-tolerant.** Now it doesn’t crash if IP list is missing.

---

### 4. 🔁 Rewrote `startClient()` to loop over IP list

**Original:**
```c
addr.sin_addr.s_addr = inet_addr(t->ip);
connect(sock, (struct sockaddr *)&addr, sizeof(addr));
allocCirBuf(t, sock);
```

**Updated:**
```c
for (int i = 0; i < t->ipListCount; i++) {
  addr.sin_addr.s_addr = inet_addr(t->ipList[i]);
  connect(sock, ...);
  allocCirBuf(t, sock);
}
```

🚀 Now the client connects to *every* gNB IP in the list, allocating buffers per socket.

---

### 5. 🧪 Added Debug Logs for Multi-gNB Mode

New logs like:
```c
LOG_I(HW, "[ASHAY] ipList[%d] = %s\n", i, t->ipList[i]);
LOG_E(HW, "[ASHAY] ipList[%d] is NULL! Skipping...\n", i);
LOG_I(HW, "[ASHAY] Trying to connect to gNB #%d at %s:%d\n", i, t->ipList[i], t->port);
```

🔎 Helps track exactly which IPs are being used and where connection fails.

---

### 6. 🔧 Minor Fixes and Quality Improvements

- 💣 Segfault-prone checks in `rfsimulator_readconfig()` now safely fall back if `ipList` is null or empty.
- 🧹 Changed config macro: `strlistptr = tempIpListPtr` (no `&` used).
- 🛡 `char *env = getenv(...)` moved inside function scope to reduce global leakage.

---

### ✅ Final Touches

You’ve made the simulator robust, scalable for multi-gNB setups, and safe against invalid inputs. Very solid engineering, Ashay! 💪
