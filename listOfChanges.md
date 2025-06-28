
# CHANGES.md

## Multi-gNB support in `simulator.c`

These are all the exact changes made from `originalfile.txt` to the final working version of `simulator.c` that supports multiple gNBs in simulation mode.

---

### 1. 📌 Modified `RFSIMULATOR_PARAMS_DESC`

**Original:**
```c
{"serveraddr", "<ip address to connect to>\n", simOpt, .strptr=&rfsimulator->ip, .defstrval="127.0.0.1", TYPE_STRING, 0 },
```

**Updated:**
```c
{"serveraddr", "<comma-separated list of IPs to connect to>\n", simOpt, .strlistptr=NULL, .defstrlistval=NULL, TYPE_STRINGLIST, 0 },
```

🔧 This allows the RF simulator to accept multiple `--rfsimulator.serveraddr` values instead of just one. We also changed the field from `strptr` to `strlistptr`, and `TYPE_STRING` to `TYPE_STRINGLIST`. The actual pointer is assigned later in `rfsimulator_readconfig()`.

---

### 2. 📌 Added `ipList` and `ipListCount` to `rfsimulator_state_t`

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
  char *ip;
  char **ipList;     // NEW: list of parsed IPs
  int ipListCount;   // NEW: number of IPs
  ...
} rfsimulator_state_t;
```

💡 These fields are used to store the parsed string list and its length from the command-line `--rfsimulator.serveraddr`. The simulator can now loop over each IP and attempt a connection.

---

### 3. 📌 Added static `tempIpListPtr` for config parser to write into

**Added globally (top of file):**
```c
static char **tempIpListPtr = NULL;
```

📥 Because the config system initializes statically before `rfsimulator` exists, we need a temporary global variable to hold the parsed IP list before copying it into the struct.

---

### 4. 📌 Modified `rfsimulator_readconfig()` to use `ipList`

**Changes inside `rfsimulator_readconfig()`:**
```c
int serverAddrIdx = config_paramidx_fromname(rfsimu_params, sizeofArray(rfsimu_params), "serveraddr");
rfsimu_params[serverAddrIdx].strlistptr = &tempIpListPtr;

...

rfsimulator->ipList = tempIpListPtr;
rfsimulator->ipListCount = rfsimu_params[serverAddrIdx].numelt;
```

🧠 This sets up the config parser to store the IP list into `tempIpListPtr`, then transfers it to the actual `rfsimulator` struct after parsing.

---

### 5. 📌 Rewrote `startClient()` to loop over multiple IPs

**Original:**
```c
addr.sin_addr.s_addr = inet_addr(t->ip);
...
connect(sock, (struct sockaddr *)&addr, sizeof(addr));
```

**Updated:**
```c
for (int i = 0; i < t->ipListCount; i++) {
  if (t->ipList[i] == NULL) continue;

  int sock = socket(AF_INET, SOCK_STREAM, 0);
  ...
  addr.sin_addr.s_addr = inet_addr(t->ipList[i]);

  while (!connected) {
    connect(sock, (struct sockaddr *)&addr, sizeof(addr));
    ...
  }

  setblocking(sock, notBlocking);
  allocCirBuf(t, sock);
}
```

🔄 This loops through all the IPs from the command line, connects to each gNB one by one, and allocates a circular buffer for each socket.

---

### 6. 📌 Added Debug Logs

**New log examples:**
```c
LOG_I(HW, "[ASHAY] ipListCount = %d\n", t->ipListCount);
LOG_I(HW, "[ASHAY] Trying to connect to gNB #%d at %s:%d\n", i, t->ipList[i], t->port);
LOG_E(HW, "[ASHAY] ipList[%d] is NULL! Skipping...\n", i);
```

🧪 These logs appear during container boot to confirm that multi-gNB parsing and connection logic are working.
