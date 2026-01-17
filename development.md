# UNav-Sim Development on Ubuntu 24.04 (peregrine)

## Fixing Compatibility Issues

To make **UNav-Sim** operational on **Ubuntu 24.04**, several critical modifications were required beyond the standard instructions in the README. These changes were necessary to resolve compatibility issues between the older AirSim/UNav-Sim codebase and the modern Ubuntu toolchain (Clang 15 and glibc 2.38).

### 1. Source Code Patches (Missing Headers)

Modern compilers are stricter about explicit header inclusions. You had to manually add `#include <cstring>` and `#include <unistd.h>` to several files to resolve "undeclared identifier" errors for functions like `memcpy`, `memset`, and `getpid`.

* **MavLinkCom Patches**:
* `MavLinkCom/src/serial_com/SerialPort.cpp`: Added `#include <cstring>`
* `MavLinkCom/src/serial_com/TcpClientPort.cpp`: Added `#include <cstring>`
* `MavLinkCom/src/serial_com/UdpClientPort.cpp`: Added `#include <cstring>`
* `MavLinkCom/src/serial_com/SocketInit.cpp`: Added `#include <cstring>`
* `MavLinkCom/src/impl/linux/Utils.cpp`: Added `#include <unistd.h>` (to fix the `getpid` error)
* `MavLinkCom/src/impl/linux/AdHocConnectionImpl.cpp`: Added `#include <unistd.h>`

* **AirLib Patches**:
* `MavLinkCom/include/MavLinkConnection.hpp`: Added `#include <memory>`

### 2. The Ubuntu 24.04 Linker Fix (ISO C23 Shim)

The most significant hurdle was a linker error: `undefined symbol: __isoc23_strtol`. This occurs because Ubuntu 24.04 uses a new version of `glibc` that renames standard functions, which the Unreal Engine linker (built for older Ubuntu versions) does not recognize.

* **File**: `external/rpclib/rpclib-2.3.0/lib/rpc/server.cc`
* **Fix**: Added a compatibility shim at the top of the file to redirect the new symbol back to the standard version:

```cpp
#include <cstdlib>
extern "C" long __isoc23_strtol(const char* nptr, char** endptr, int base) {
    return strtol(nptr, endptr, base);
}

```

### 3. Environment & Permission Adjustments

Since you are working in `/media/storage`, standard file permissions and script paths needed manual intervention:

* **Ownership**: Ran `sudo chown -R saswata:saswata /media/storage/saswata/Dev_Underwater` to ensure the build process had write access without needing `sudo`.
* **Executable Permissions**: Manually set `chmod +x` on various shell scripts (like `clean.sh` and `Build.sh`) that the main `build.sh` tries to trigger internally.

### 4. Correct Build & Launch Workflow

The standard "Generate Project Files" right-click method often fails on Linux. The reliable sequence for Ubuntu 24.04 is:

1. **Build UNav-Sim**: Run `./build.sh` from the root.
2. **Generate UE Project**: Use the terminal script:
`./GenerateProjectFiles.sh -project="[PATH_TO_BLOCKS].uproject" -game -engine`
3. **Compile via Command Line**: Use the Unreal `Build.sh` script to compile the "BlocksEditor" target before opening the GUI to avoid the "cannot compile while engine is running" conflict.
4. **Launch via Terminal**: Launch the editor specifically pointing to the project file to ensure all plugins load correctly.

### Summary Checklist for Future Reinstalls

1. Apply the **7 header fixes** (cstring/unistd).
2. Apply the **__isoc23_strtol shim** in `server.cc`.
3. Ensure **ownership** of the storage directory.
4. **Build from terminal** entirely before attempting to open the Unreal GUI.
