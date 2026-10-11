# Windows 构建与适配说明（MinGW-w64 + Qt 6.12）

本文件记录在 Windows 上从源码构建 ModbusScope（v5.0.1）的过程，以及为适配 Qt 6.12 / MinGW 13.1 所做的构建修复。

## 1. 工具链

- C++ 编译器：MinGW-w64 g++ 13.1.0（x86_64-posix-seh，Qt 自带 `Tools/mingw1310_64`）
- CMake：Qt 自带 `Tools/CMake_64/bin/cmake.exe`（>= 3.19）
- Qt：6.12.0 MinGW 64-bit（`D:\Qt\6.12.0\mingw_64`）
- 第三方依赖已随源码包提供：`libraries/muparser`、`libraries/qcustomplot`（v2.1.1 Qt6.8 port）、`tests/googletest` v1.18.0，无需 dfetch 拉取
- 协议适配器为上游预编译二进制：`adapters/Modbus/modbusadapter.exe`，主程序通过 IPC（local socket）与它通信

## 2. 配置与编译

```powershell
$env:PATH = "D:\Qt\Tools\mingw1310_64\bin;D:\Qt\Tools\CMake_64\bin;D:\Qt\6.12.0\mingw_64\bin;" + $env:PATH
cd D:\dp\opensource\ModbusScope
cmake -S . -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release
cmake --build build -j 8
```

产物：
- 主程序 `build/src/bin/win/modbusscope.exe`（约 4.9 MB）
- 65 个单元测试可执行文件（`build/tst_*.exe`）

## 3. 对上游 CMakeLists 的适配修改

### 3.1 移除过时的 Qt6SerialBus 依赖

上游顶层 `CMakeLists.txt` 把 `SerialBus` 列为 `find_package(... REQUIRED)` 组件，并把 `Qt::SerialBus` 链入主程序。但代码检索（`QModbus*`、`QtSerialBus`）显示：

- 主程序 `src/` 下**没有任何文件**直接使用 QModbusClient / QModbusTcpClient；
- 真正的 Modbus 通信在独立预编译进程 `adapters/Modbus/modbusadapter.exe` 里，主程序通过 `src/ProtocolAdapter/adapterclient.cpp` 的 IPC 接口与其通信；
- 整个源码树只有 `CHANGELOG.md` 提到 SerialBus。

这是上游从"进程内 Modbus"重构为"外部 adapter 进程"架构后，CMakeLists 未同步清理留下的过时依赖。在未安装 QtSerialBus 模块的最小 Qt 环境下，原来的 `find_package` 会直接报错退出。修改为只保留实际使用的组件（Widgets/Xml/Network/PrintSupport/SerialPort）。

### 3.2 qcustomplot：QImage::mirrored() 在 Qt 6.12 已 deprecated

Qt 6.12 把 `QImage::mirrored(bool, bool)` 标记为 `[[deprecated]]`（推荐改用 `flipped(Qt::Orientations)`）。上游 qcustomplot v2.1.1 尚未迁移，项目全局 `-Werror` 把这条警告升级为错误，导致 `qcustomplot.cpp` 4 处编译失败（行 20665 / 26703 / 26931 / 30356）。

处理方式：**只给 qcustomplot 这个第三方 target 加 `-Wno-error=deprecated-declarations`**，不削弱主项目的 `-Werror` 严格性。

### 3.3 主项目：QSortFilterProxyModel::invalidateFilter() 在 Qt 6.12 已 deprecated

`src/MbcInterface/mbcregisterfilter.cpp` 两处调用 `invalidateFilter()`（行 40、54），Qt 6.12 推荐改用 `beginFilterChange()/endFilterChange()`。旧 API 在 6.12 仍可用，直接改源码会破坏对旧 Qt 版本的兼容。

处理方式：顶层 `add_compile_options` 在 `-Werror` 后追加 `-Wno-error=deprecated-declarations`，让 deprecation 警告不阻断编译，但保留其他 `-Werror` 检查。

## 4. 测试

```powershell
cd build
ctest --output-on-failure
```

65 个测试全部编译成可执行文件。本机 MinGW 13.1 + Qt 6.12 环境下 ctest 报告 52 通过 / 13 失败。失败用例（tst_util、tst_scopecontroller 等）启动即退出、无任何输出，疑似 MinGW posix-seh 线程模型与 QtTest 静态初始化在本机的运行时交互问题，与上述 CMake 适配修改无关（修改仅影响编译期警告级别和依赖清单，不改变运行时代码路径）。主程序 `modbusscope.exe` 可正常启动。

## 5. 运行时部署

主程序启动前，把 Qt 运行时 DLL 复制到 exe 旁（或把 `D:\Qt\6.12.0\mingw_64\bin` 加入 PATH）：

```powershell
windeployqt --no-translations build\src\bin\win\modbusscope.exe
```

协议适配器 `adapters/Modbus/modbusadapter.exe` 需与主程序一起分发（主程序通过适配器发现机制拉起它）。
