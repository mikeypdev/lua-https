# AGENTS.md

lua-https is a C++ Lua module providing HTTPS support via native platform backends. This fork's specific purpose is to build `https` libraries for **LÖVE 11.5** (the upstream targets LÖVE 12.0, which bundles lua-https natively). Targets Windows, Linux, macOS, iOS, and Android.

## Build Commands

### Linux
```bash
cmake -Bbuild -S. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$PWD/install
cmake --build build --target install
```
Output: `install/https.so`

### macOS (Xcode, universal binary)
```bash
cmake -Bbuild -S. -G Xcode -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" -DLUA_INCLUDE_DIR=<path> -DLUA_LIBRARIES=<path>
cmake --build build --config Release
```
Output: `build/src/Release/https.so`

### Windows (MSVC)
```bash
cmake -Bbuild -S. -DCMAKE_INSTALL_PREFIX=%CD%\install -A x64 -DLUA_INCLUDE_DIR=<path> -DLUA_LIBRARIES=<path>
cmake --build build --config Release --target install
```
Output: `install/https.dll`

### Android
Place entire lua-https source tree in `<love-android>/love/src/jni/lua-modules/lua-https` and compile love-android as usual. Uses `Android.mk` (ndk-build), not CMake.

### CMake Options (Linux)
- `-DUSE_CURL_BACKEND=ON/OFF` — enable cURL backend (default: ON)
- `-DUSE_OPENSSL_BACKEND=ON/OFF` — enable OpenSSL backend (default: ON)
- `-DLIBRARY_LOADER=unix|windows|linktime` — dynamic library loading method

## Testing

Tests are network-dependent integration tests (they hit real endpoints like `postman-echo.com` and GitHub raw content). Run with:
```bash
# After building and installing
cd install
lua -l "https" ../example/test.lua      # Lua 5.1
luajit -l "https" ../example/test.lua   # LuaJIT
```
There is no offline/unit test suite.

## Architecture

### Backend Selection (compile-time + runtime)
Platform backends are selected at **compile time** via CMake options (or auto-detected from `config.h` when CMake-generated config is absent). At **runtime**, `HTTPS.cpp` iterates a statically-ordered array of backend client pointers, calling `valid()` on each and using the first one that returns true.

Backend priority order in `clients[]` array (`src/common/HTTPS.cpp`):
1. cURL
2. OpenSSL
3. WinINet (must be above SChannel per code comment)
4. SChannel
5. NSURL
6. Android

### Platform Backends
| Platform | Backend | Source | Notes |
|----------|---------|--------|-------|
| Linux | cURL | `src/generic/CurlClient.cpp` | Dynamically loads `libcurl.so.4` |
| Linux | OpenSSL | `src/generic/OpenSSLConnection.cpp` | Raw socket + TLS via `ConnectionClient<OpenSSLConnection>` |
| macOS/iOS | NSURL | `src/apple/NSURLClient.mm` | Objective-C++, ARC enabled |
| Windows | SChannel | `src/windows/SChannelConnection.cpp` | Raw socket + TLS via `ConnectionClient<SChannelConnection>` |
| Windows | WinINet | `src/windows/WinINetClient.cpp` | Higher-level Windows API |
| Android | JNI | `src/android/AndroidClient.cpp` | Calls Java via JNI; requires `LuaHTTPS.java` from `src/android/java/` |

### Key Abstractions
- **`HTTPSClient`** (`src/common/HTTPSClient.h`) — Abstract base. Defines `Request` (url, headers, postdata, method) and `Reply` (body, headers, responseCode) structs. Header maps use case-insensitive keys via `ci_string_less`.
- **`Connection`** (`src/common/Connection.h`) — Abstract TCP/TLS connection interface (connect, read, write, close). Used by OpenSSL and SChannel backends.
- **`ConnectionClient<Connection>`** (`src/common/ConnectionClient.h`) — Template `HTTPSClient` that delegates to `HTTPRequest` with a `Connection` factory. Used by socket-based backends (OpenSSL, SChannel).
- **`HTTPRequest`** (`src/common/HTTPRequest.h`) — Implements HTTP protocol over a `Connection` (URL parsing, sending request, reading response). Takes a `ConnectionFactory` function.
- **`LibraryLoader`** (`src/common/LibraryLoader.h`) — Platform-abstracted dynamic library loading (dlopen/LoadLibrary). Used by CurlClient to load libcurl at runtime. Three implementations: `UnixLibraryLoader`, `WindowsLibraryLoader`, `LinktimeLibraryLoader`.

### Configuration System
- **With CMake**: `config-generated.h.in` is processed into `config-generated.h` defining `HTTPS_BACKEND_*` macros based on CMake options.
- **Without CMake** (e.g., IDE builds): `config.h` auto-detects platform via preprocessor (`_WIN32`, `__APPLE__`, `__ANDROID__`, `linux`) and `__has_include` for available libraries.

### Lua Binding
Entry point: `luaopen_https` in `src/lua/main.cpp`. Exposes a single `https.request` function. The module name is `"https"` (not `"lua-https"`).

## Conventions

- C++14 standard
- Headers use `#pragma once`
- All backend code is guarded by `#ifdef HTTPS_BACKEND_*` / `#endif` pairs in both headers and implementation files
- `extern "C"` wrapping for Lua API includes
- Symbols use `HTTPS_DLLEXPORT` macro for platform-appropriate export visibility
- GCC/Clang: `-fvisibility=hidden` on both compile and link (only `luaopen_https` is exported)
- The cURL backend dynamically loads libcurl symbols at runtime rather than linking at compile time, to avoid hard runtime dependencies
- `LUA_INCLUDE_DIR` and `LUA_LIBRARIES` CMake variables can be set manually; otherwise CMake tries `FindLuaJIT` first, then falls back to `FindLua 5.1`

## Context for this Fork

- **This fork targets LÖVE 11.5**, not LÖVE 12.0. LÖVE 12.0 bundles lua-https natively, but 11.5 does not — this fork exists to fill that gap. The built shared library (`https.so` / `https.dll`) should be placed alongside or loadable by LÖVE 11.5's Lua runtime.
- **LÖVE/LOVR support**: The CMakeLists has a special `LOVR` guard that uses `LOVR_LUA` instead of finding LuaJIT.
- **macOS linking**: Uses `-undefined dynamic_lookup` (not on iOS, which links Lua normally).
- **Windows linking**: Must explicitly link `${LUA_LIBRARIES}` on Windows; other platforms resolve Lua symbols at load time.
- **cURL backend**: Dynamically loads `libcurl.so.4` (Linux) or `libcurl.dll` (Windows) at runtime. The `linktime` library loader exists for static linking scenarios.
- **Android**: Requires Java file (`src/android/java/org/love2d/luahttps/LuaHTTPS.java`) added to the Android project. `java.txt` documents the path.
- **No unit tests**: All tests require network access.
- **The module output extension**: `.so` on Linux/macOS, `.dll` on Windows — but CMake `set_target_properties(https PROPERTIES PREFIX "")` strips the `lib` prefix on all platforms.
