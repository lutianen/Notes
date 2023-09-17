# Compile Qt 5.15.10 using mingw64 for Windows

## System requirements

> pen a command prompt and ensure that the following tools can be found in the path : `perl -v`、`gcc -v`、`python -V`、`ruby -v`
>
> - MinGW-builds gcc 4.9 or later
> - Perl version 5.12 or later
> - Python version 2.7 or later
> - Ruby version 1.9.3 or later

### 下载地址

- [MinGW](https://github.com/niXman/mingw-builds-binaries/releases/download/8.5.0-rt_v10-rev0/x86_64-8.5.0-release-posix-seh-rt_v10-rev0.7z)

  > ```powershell
  > # 测试 C++ compiler supporting the C++11 standard
  > F:\qt-everywhere-src-5.15.2> mingw32-make -v
  > 
  > GNU Make 4.2.1
  > Built for i686-w64-mingw32
  > Copyright (C) 1988-2016 Free Software Foundation, Inc.
  > License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
  > This is free software: you are free to change and redistribute it.
  > There is NO WARRANTY, to the extent permitted by law.
  > ```

- [Perl](https://www.perl.org/get.html)

  > ```powershell
  > # 测试
  > F:\qt-everywhere-src-5.15.2> perl -v
  > 
  > This is perl 5, version 32, subversion 1 (v5.32.1) built for MSWin32-x86-multi-thread-64int
  > 
  > Copyright 1987-2021, Larry Wall
  > 
  > Perl may be copied only under the terms of either the Artistic License or the
  > GNU General Public License, which may be found in the Perl 5 source kit.
  > 
  > Complete documentation for Perl, including FAQ lists, should be found on
  > this system using "man perl" or "perldoc perl".  If you have access to the
  > Internet, point your browser at http://www.perl.org/, the Perl Home Page
  > ```

- [Ruby](https://rubyinstaller.org/downloads/)

  > ```powershell
  > # 测试
  > C:\Users\Tianen>ruby -v
  > ruby 3.1.1p18 (2022-02-18 revision 53f5fc4236) [i386-mingw32]
  > ```

- [Python](https://www.python.org/ftp/python/3.8.10/python-3.8.10-amd64.exe)

  > ```powershell
  > # test
  > python
  > Python 3.8.10 (tags/v3.8.10:3d8993a, May  3 2021, 11:48:03) [MSC v.1928 64 bit (AMD64)] on win32
  > Type "help", "copyright", "credits" or "license" for more information.
  > >>> 
  > ```

## Build

- **Step 1. 设置临时环境变量**

  NOTE：
  
  - 一次复制一条并执行（不要一次复制多条，可能会报错）
  - `F:\Windows Kits\10\bin\10.0.19041.0\x64`，目录下需要包含 `fxc.exe` 文件，请根据实际情况调整，可能的路径`C:\Program Files (x86)\Windows Kits\8.1\bin\x64.`;

  ```powershell
  set path=F:\Python\Python38\;F:\Python\Python38\Scripts;C:\Windows;C:\Windows\system32;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH;F:\Strawberry\perl\site\bin;F:\Strawberry\perl\bin;F:\mingw64\bin;F:\Ruby31-x64\bin;F:\LLVM\bin;F:\cmake32\bin;F:\Windows Kits\10\bin\10.0.19041.0\x64;

  set LANG=en

  cmd /k
  ```

- **Step 2. mkdir build && configure**

  **NOTE 实际编译选项，根据情况进行调整**

  ```powershell
  mkdir build

  cd build

  ..\configure.bat -static -release -prefix F:\Qt\Qt5.15.10 -confirm-license -opensource -platform win32-g++ -qt-sqlite -qt-zlib -qt-libpng -qt-libjpeg -opengl desktop -qt-pcre -qt-freetype -nomake tests -nomake examples -skip qtwebengine

  # -prefix 指定安装将会部署的位置，根据自己情况修改
  # -debug-and-release 指示编译生成debug版和release版的Qt库
  # -platform win32-g++ 指明编译平台是windows，并使用mingw编译器
  # -opensource -confirm-license 是为了自动确认开源证书，免得到时暂停手动确认
  # -nomake tests 不需要编译测试工程
  # -skip qtwebengine 暂时先不编译webengine模块，因为太大了
  # -qt-zlib -ssl -icu 指示检测这些库，并在需要时使用
  # -opengl desktop 明确指示使用你windows上安装的opengl驱动来编译程序，但这样编译出的程序在别的电脑上运行时需要目标电脑上安装的opengl驱动能兼容你的程序
  ```

- **Step 3. make && install**

  ```powershell
  mingw32-make -j16    # 开启十六线程编译，根据自己电脑实际核心数调整

  mingw32-make install
  ```

## Qt 5.15.x 可能遇到的 BUG

### `qfilesystemengine_win.cpp:669:16: error: redefinition of 'struct _FILE_ID_INFO' typedef struct _FILE_ID_INFO {`

Solution:
  修改 `qtbase/src/corelib/io/qfilesystemengine_win.cpp:669` 源码：

  Change

  `#if defined(Q_CC_MINGW) && WINVER < 0x0602 //  Windows 8 onwards`

  to

  `#if defined(Q_CC_MINGW) && WINVER < 0x0602 && !(_WIN32_WINNT >= _WIN32_WINNT_WIN8) //  Windows 8 onwards`

### `'_uuidof' was not declared in this scope`

Solution:

  修改 `qtdeclarative/src/plugins/scenegraph/d3d12/qsgd3d12engine.cpp` 和 `qtdeclarative/src/plugins/scenegraph/d3d12/qsgd3d12engine_p_p.h` 源码：

  ```bash
  diff --git a/src/plugins/scenegraph/d3d12/qsgd3d12engine.cpp b/src/plugins/scenegraph/d3d12/qsgd3d12engine.cpp
  index 75bde2c66b..3594878eca 100644
  --- a/src/plugins/scenegraph/d3d12/qsgd3d12engine.cpp
  +++ b/src/plugins/scenegraph/d3d12/qsgd3d12engine.cpp
  @@ -221,7 +221,7 @@ static void getHardwareAdapter(IDXGIFactory1 *factory, IDXGIAdapter1 **outAdapte
          if (SUCCEEDED(factory->EnumAdapters1(adapterIndex, &adapter))) {
              adapter->GetDesc1(&desc);
              const QString name = QString::fromUtf16((char16_t *) desc.Description);
  -            HRESULT hr = D3D12CreateDevice(adapter.Get(), fl, _uuidof(ID3D12Device), nullptr);
  +            HRESULT hr = D3D12CreateDevice(adapter.Get(), fl, __uuidof(ID3D12Device), nullptr);
              if (SUCCEEDED(hr)) {
                  qCDebug(QSG_LOG_INFO_GENERAL, "Using requested adapter '%s'", qPrintable(name));
                  *outAdapter = adapter.Detach();
  @@ -238,7 +238,7 @@ static void getHardwareAdapter(IDXGIFactory1 *factory, IDXGIAdapter1 **outAdapte
          if (desc.Flags & DXGI_ADAPTER_FLAG_SOFTWARE)
              continue;

  -        if (SUCCEEDED(D3D12CreateDevice(adapter.Get(), fl, _uuidof(ID3D12Device), nullptr))) {
  +        if (SUCCEEDED(D3D12CreateDevice(adapter.Get(), fl, __uuidof(ID3D12Device), nullptr))) {
              const QString name = QString::fromUtf16((char16_t *) desc.Description);
              qCDebug(QSG_LOG_INFO_GENERAL, "Using adapter '%s'", qPrintable(name));
              break;
  diff --git a/src/plugins/scenegraph/d3d12/qsgd3d12engine_p_p.h b/src/plugins/scenegraph/d3d12/qsgd3d12engine_p_p.h
  index a95cbb1cbb..54a2c4dc8f 100644
  --- a/src/plugins/scenegraph/d3d12/qsgd3d12engine_p_p.h
  +++ b/src/plugins/scenegraph/d3d12/qsgd3d12engine_p_p.h
  @@ -55,6 +55,7 @@
  #include <QCache>

  #include <d3d12.h>
  +#include <d3d12sdklayers.h>
  #include <dxgi1_4.h>
  #include <dcomp.h>
  #include <wrl/client.h>
  @@ -263,8 +264,8 @@ private:
      void beginFrameDraw();
      void endDrawCalls(bool lastInFrame = false);

  -    static const int MAX_SWAP_CHAIN_BUFFER_COUNT = 4;
  -    static const int MAX_FRAME_IN_FLIGHT_COUNT = 4;
  +    static inline const int MAX_SWAP_CHAIN_BUFFER_COUNT = 4;
  +    static inline const int MAX_FRAME_IN_FLIGHT_COUNT = 4;

      bool initialized = false;
      bool inFrame = false;
  ```
