
# 0) （可选）清理旧痕迹，避免版本冲突

**Steps**

```bash
# 如果你之前装过 apt 版 DART，建议先卸掉（我们要用 /usr/local 的源码版）
sudo apt remove 'libdart*'
sudo apt autoremove
# 检查系统里还剩哪些 dart 的 cmake 配置
sudo updatedb && locate DARTConfig.cmake
```

**Requirements**：可用 sudo。
**Debug**

* 后面 `cmake` 时务必指向 **/usr/local/share/dart/cmake**，否则会不小心又捡到 `/usr/include` 里的旧版本。

---

# 1) 建 Conda 环境

**Steps**

```bash
conda create -n mass python=3.10 -y
conda activate mass
```

**Requirements**：Miniconda/Anaconda 已安装。
**Debug**

* 如果用 defaults 渠道弹 TOS，先执行：

  ```bash
  conda tos accept --channel https://repo.anaconda.com/pkgs/main
  conda tos accept --channel https://repo.anaconda.com/pkgs/r
  ```
* 或者改用 conda-forge：

  ```bash
  conda config --remove channels defaults
  conda config --add channels conda-forge
  conda config --set channel_priority strict
  ```

---

# 2) 装 Python 侧依赖（给 CMake 用的 pybind11 等）

**Steps**

```bash
conda install -y -c conda-forge cmake pkg-config pybind11 numpy matplotlib ipython
```

**Requirements**：已激活 `mass` 环境。
**Debug**

* 后面 `cmake` 找不到 pybind11 时，传：

  ```bash
  -Dpybind11_DIR="$(python -c 'import pybind11; print(pybind11.get_cmake_dir())')"
  ```

---

# 3) 装系统库（编 DART/MASS 必需）

**Steps**

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential git \
  freeglut3-dev libassimp-dev libeigen3-dev libbullet-dev \
  libccd-dev libfcl-dev liburdfdom-dev libtinyxml-dev \
  libxi-dev libxmu-dev libgl1-mesa-dev libglu1-mesa-dev
```

**Requirements**：Ubuntu 24.04，sudo。
**Debug**

* 缺 OpenGL / GLUT 相关符号、`glut*` 未声明：确认 `freeglut3-dev` 在。
* 后续 MASS 报 `dart/gui/...` 头缺失，多半是 DART 包问题，见第 4 步。

---

# 4) 从源码安装 DART 6.13（带 GUI + Bullet）

> 这是关键：apt 版 6.13 在 24.04 上常见 **GUI 头（glut）缺失**；源码安装最省心。

**Steps**

```bash
git clone https://github.com/dartsim/dart.git
cd dart
git checkout tags/v6.13.2 -b v6.13.2-local

mkdir build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  -DDART_BUILD_GUI=ON \
  -DDART_BUILD_COLLISION_BULLET=ON \
  -DDART_BUILD_EXAMPLES=OFF \
  -DDART_BUILD_TESTS=OFF \
  -DDART_BUILD_TUTORIALS=OFF
make -j"$(nproc)"
sudo make install
```

**Requirements**：第 3 步的系统库已装齐。
**Debug**

* **找不到 Bullet**：`sudo apt-get install -y libbullet-dev`，然后重配。
* **找不到 urdfdom / tinyxml**：确认第 3 步依赖齐。
* 安装后自检：

  ```bash
  ls /usr/local/include/dart/gui/glut/Win3D.hpp
  ls /usr/local/share/dart/cmake/DARTConfig.cmake
  ```

  两者都在就 OK。

---

# 5) 配置并编译 MASS（确保用 C++17、指向正确的 DART & Python）

**Steps**

```bash
# 到 MASS 仓库根目录
cd ~/Desktop/MASS

# 强烈建议：把 C++17“兜底”塞到编译器标志，防止被旧 CMake 改回 11/14
export CXXFLAGS="-std=gnu++17 ${CXXFLAGS:-}"

# 干净构建
rm -rf build
mkdir build && cd build

cmake .. \
  -DDART_DIR=/usr/local/share/dart/cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DPYTHON_EXECUTABLE="$(which python)" \
  -Dpybind11_DIR="$(python -c 'import pybind11; print(pybind11.get_cmake_dir())')" \
  -DCMAKE_PREFIX_PATH="$CONDA_PREFIX"

make -j"$(nproc)"
```

**Requirements**：

* 第 4 步已经把 DART 安到 `/usr/local`
* 仍在 `conda activate mass` 的环境里
  **Debug**
* **出现 `std::invoke_result_t / is_same_v` 相关错误**：说明仍在用 C++11/14。

  * 查命令行标准：`grep -o -- '-std=[^ "]\+' compile_commands.json | sort -u`。
  * 如果还看到 `-std=gnu++11`，在对应目标上**追加**（保证出现在命令行最后）：

    * `core/CMakeLists.txt` 里（`add_library(mss ...)` 后）加：

      ```cmake
      target_compile_features(mss PUBLIC cxx_std_17)
      target_compile_options(mss PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-std=gnu++17>)
      ```
    * `render/CMakeLists.txt` 里（`add_executable(render ...)` 后）加同样两行。
    * `python/CMakeLists.txt` 里（`pybind11_add_module(pymss ...)` 后）加同样两行。
  * 再 `rm -rf build && cmake .. && make`
* **`dart/gui/glut/glut.hpp` 找不到**：你没用 `/usr/local` 的源码版 DART。

  * CMake 输出应显示 `Found DART: /usr/local/include ...`，不是 `/usr/include`。
  * 传 `-DDART_DIR=/usr/local/share/dart/cmake`，必要时加 `export CMAKE_PREFIX_PATH=/usr/local:$CMAKE_PREFIX_PATH`。
* **`pybind11Config.cmake` 找不到**：确保在 `mass` 环境里安装了 pybind11，并传 `-Dpybind11_DIR=...`。
* **`collision-bullet` 缺失**：重编 DART 时开启 `-DDART_BUILD_COLLISION_BULLET=ON`。

---

# 6) 运行

**渲染 UI（无权重）**

```bash
./build/render/render ./data/metadata.txt
```

**训练（会生成 ./nn 权重）**

```bash
conda activate mass
cd python
python main.py -d ../data/metadata.txt
```

**渲染已训练模型**

```bash
# 肌肉模型
./build/render/render ./data/metadata.txt ./nn/xxx.pt ./nn/xxx_muscle.pt
# 力矩模型
./build/render/render ./data/metadata.txt ./nn/xxx.pt
```

**Debug**

* 远程/无显示环境报 OpenGL/GLX 错：需要本地 X 会话，或用 `ssh -X`，或 `xvfb-run` 启动渲染（性能有限）。
* 运行时报 `libpython*.so` 找不到：说明 `pymss.so` 链到的不是当前 Python。重配第 5 步，确保用 `-DPYTHON_EXECUTABLE="$(which python)"` 指向 conda 的 python。

---

# 常见坑位速查表（遇到就对号入座）

| 报错/现象                                          | 根因                          | 解决                                                                                                                                          |
| ---------------------------------------------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Could NOT find Boost (regex/system/filesystem) | 缺 Boost                     | `sudo apt-get install -y libboost-all-dev` 或 `conda install -c conda-forge boost` 并 `-DBOOST_ROOT=$CONDA_PREFIX -DBoost_NO_SYSTEM_PATHS=ON` |
| `parsePose(...) is ambiguous`                  | DART 6.8 + urdfdom 4.0 冲突   | 用 DART 6.13（推荐）；或在 DART 源里把 `parsePose(...)` 调用改成 `urdf::parsePose(...)`                                                                    |
| `missing: collision-bullet`                    | 没有 Bullet 碰撞模块              | 源码编 DART 时加 `-DDART_BUILD_COLLISION_BULLET=ON`；apt 方案需装 `libdart-collision-bullet*`                                                         |
| `ikfast.h: No such file`                       | 缺 ikfast 头                  | 源码装 DART 自带；apt 方案需要 `libdart-external-ikfast-dev`                                                                                          |
| `std::invoke_result_t / is_same_v` 未定义         | 编译标准不是 C++17                | 见第 5 步 Debug，强制 `-std=gnu++17`，必要时在各目标追加 `target_compile_options(... -std=gnu++17)`                                                         |
| `dart/gui/glut/glut.hpp` 缺                     | apt 包缺 GUI 头                | 源码安装 DART（第 4 步）；或临时把 `dart/gui/glut` 目录从源码拷到 `/usr/local/include/dart/gui/`                                                                |
| `glutInit` 等未声明                                | 没包含 `<GL/glut.h>` 或没链接 GLUT | 确保 `#include <GL/glut.h>`（若你改了 include 路径）；并链接 `GLUT::GLUT`，且装了 `freeglut3-dev`                                                             |
| `pybind11Config.cmake` 找不到                     | 没在 conda 环境 / 没传路径          | 激活环境；传 `-Dpybind11_DIR="$(python -c 'import pybind11; print(pybind11.get_cmake_dir())')"`                                                   |
| DART 仍指向 `/usr/include`                        | 找到了 apt 版                   | 指定 `-DDART_DIR=/usr/local/share/dart/cmake`，必要时 `export CMAKE_PREFIX_PATH=/usr/local:$CMAKE_PREFIX_PATH`                                    |



# 🔧 补充 1：先装 apt 版 DART → 发现问题 → 直接改为源码编译

**你实际的过程（建议也这样写进笔记里）：**

* **先用 apt 安装了系统自带的 DART 6.13**（以及 Boost/Assimp/GL 等依赖），随后按需补装了 DART 的组件：

  * `libdart-collision-bullet6.13` / `libdart-collision-bullet-dev`（Bullet 碰撞）
  * `libdart-utils-urdf-dev`（URDF）
  * `libdart-external-ikfast-dev`（ikfast 头文件）
  * 还有 `libboost-all-dev` 等

* 这样可以先把 **核心库** 和 **Python 扩展（pymss）** 编译通过；
  但 **render** 阶段报 `dart/gui/glut/*.hpp` 缺失（apt 包常见缺头问题），即使 FreeGLUT 在，也会卡。

* **未继续纠结 apt 版**，直接切换到 **源码编译 DART 6.13（带 GUI+Bullet）** 并安装到 `/usr/local`：

  ```bash
  git clone https://github.com/dartsim/dart.git
  cd dart && git checkout v6.13.2 -b v6.13.2-local
  mkdir build && cd build
  cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local \
           -DDART_BUILD_GUI=ON -DDART_BUILD_COLLISION_BULLET=ON \
           -DDART_BUILD_EXAMPLES=OFF -DDART_BUILD_TESTS=OFF -DDART_BUILD_TUTORIALS=OFF
  make -j"$(nproc)"
  sudo make install
  ```

* **回到 MASS** 时，**一定要指向** `/usr/local` 这套：

  ```bash
  cmake .. -DDART_DIR=/usr/local/share/dart/cmake  ...
  ```

  这样 `render` 也能顺利编过（因为 `/usr/local/include/dart/gui/glut/*.hpp` 都齐）。

> 小备注：你写的“补充编译了 plot 成功”应是笔误，实际是 **补齐了 Bullet/ikfast 等组件**，但没再验证 apt 版 render，**直接改为源码 DART**（这是对的，省时省心）。

---

# 🧪 补充 2：检查标准版本 & 三个 CMakeLists 都要加 C++17

**先让 CMake 生成编译数据库**（方便排查 `-std=`）：

```bash
cmake .. -DCMAKE_EXPORT_COMPILE_COMMANDS=ON  ...
```

**检查实际使用的 C++ 标准**（看是否混用 11/17）：

```bash
grep -o -- '-std=[^ "]\+' build/compile_commands.json | sort -u
# 理想只剩：-std=gnu++17
# 你当时看到的是同时有 -std=gnu++11 和 -std=gnu++17
```

**为避免某些目标仍被旧设置覆盖，分别在三个 CMakeLists 里强制 C++17：**

1. `core/CMakeLists.txt`（`add_library(mss ...)` 之后）：

```cmake
target_compile_features(mss PUBLIC cxx_std_17)
target_compile_options(mss PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-std=gnu++17>)
```

2. `render/CMakeLists.txt`（`add_executable(render ...)` 之后）：

```cmake
target_compile_features(render PUBLIC cxx_std_17)
target_compile_options(render PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-std=gnu++17>)
```

3. `python/CMakeLists.txt`（`pybind11_add_module(pymss ...)` 之后）：

```cmake
target_compile_features(pymss PUBLIC cxx_std_17)
target_compile_options(pymss PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-std=gnu++17>)
```

> 说明：
>
> * `target_compile_features(... cxx_std_17)` 让 CMake 知道要 17；
> * `target_compile_options(... -std=gnu++17)` **把 `-std=gnu++17` 追加到命令行最后**，覆盖掉任何早先遗留的 `-std=gnu++11/14`。
> * 你也可以在顶层 `CMakeLists.txt` 兜底：
>
>   ```cmake
>   set(CMAKE_CXX_STANDARD 17)
>   set(CMAKE_CXX_STANDARD_REQUIRED ON)
>   set(CMAKE_CXX_EXTENSIONS ON)  # 给 GCC 走 gnu++17；如果想纯 c++17 改成 OFF
>   ```

**再次验证：**

```bash
rm -rf build && mkdir build && cd build
cmake .. -DDART_DIR=/usr/local/share/dart/cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ...
grep -o -- '-std=[^ "]\+' compile_commands.json | sort -u
# 只应剩下 -std=gnu++17
```

你现在这两行输出已经把“谜底”都给了：

```
readelf … RUNPATH: […:/usr/local/lib:/home/.../miniconda3/envs/mass/lib]
ldd … 
  libpython3.8.so.1.0 => not found
  libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6  ✅（系统的）
```

# 发生了什么（核心原因）

* **构建阶段**：CMake 确实按你的 conda Python 来**链接**了（所以二进制里 NEEDED 写的是 `libpython3.8.so.1.0`）。
* **运行阶段**：动态链接器去 `RUNPATH` 里的路径找 `libpython3.8.so.1.0`。你的 `RUNPATH` 里包含了 conda 的 `…/envs/mass/lib`，但它**没找到同名文件**，于是 `=> not found`。
* 同时 `libstdc++` 现在走的是**系统**那份 `/lib/x86_64-linux-gnu`（好事，说明我们已经避免了 conda 的旧 libstdc++ 抢位），所以之前 GLIBCXX 的锅已经没了。

> 换句话说：**不是“用了两个 Python”**，DART 也不吃 Python。现在就差**让运行时能找到 `libpython3.8.so.1.0`**。

# 为什么找不到？

多半是 **你的 conda 环境里没有 `libpython3.8.so.1.0` 这个文件名**（只有 `libpython3.8.so`）。conda 的打包有时不带 `.so.1.0` 这个 SONAME 软链接。

# 快速修法（二选一）

## 方案 A：确认并创建软链接（最简单）

1. 看看 conda 里到底有哪些文件：

   ```bash
   ls -l $CONDA_PREFIX/lib/libpython3.8.so*
   ```

   * 如果只有 `libpython3.8.so` 而**没有** `.so.1.0`，就补一个软链接：

     ```bash
     ln -s "$CONDA_PREFIX/lib/libpython3.8.so" \
           "$CONDA_PREFIX/lib/libpython3.8.so.1.0"
     ```
2. 再次检查并运行：

   ```bash
   ldd ./render/render | egrep -i 'python|stdc\+\+'
   ./render/render ./data/metadata.txt
   ```

## 方案 B：重配时让 CMake 直接链接到实际存在的 `.so`

如果你不想做软链接，就把 `PYTHON_LIBRARY` 指到**实际存在**的那个文件（通常是没有版本号的 `.so`）：

```bash
conda activate mass
cd ~/Desktop/MASS
rm -rf build && mkdir build && cd build

PYVER=$(python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')
cmake .. \
  -DDART_DIR=/usr/local/share/dart/cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DPYTHON_EXECUTABLE="$(which python)" \
  -DPYTHON_LIBRARY="$CONDA_PREFIX/lib/libpython${PYVER}.so" \
  -DPYTHON_INCLUDE_DIR="$CONDA_PREFIX/include/python${PYVER}" \
  -Dpybind11_DIR="$(python -c 'import pybind11; print(pybind11.get_cmake_dir())')" \
  -DCMAKE_SKIP_RPATH=ON -DCMAKE_BUILD_RPATH="" -DCMAKE_INSTALL_RPATH=""
make -j"$(nproc)"
ldd ./render/render | egrep -i 'python|stdc\+\+'
```

> 上面我关了 RPATH，避免把 conda 的 `lib/`写进可执行文件里，防止又把 libstdc++ 搞乱。现在 `ldd` 里应显示：
> `libpython3.8.so => /home/.../miniconda3/envs/mass/lib/libpython3.8.so`
> `libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6`

# 你问的三个问题，逐条回答

1. **“CMake 真的用了 conda 的 Python？”**
   是的。链接时我们指定了 `PYTHON_EXECUTABLE/PYTHON_LIBRARY`，可执行文件的 NEEDED 里写了 `libpython3.8.so.1.0`。

2. **“make 出来的东西是什么？DART 用系统自带的 python？”**
   DART **不依赖 Python**。依赖 Python 的是 MASS 的 `render`（嵌入 Python）和 `pymss` 模块。DART 只依赖它自己的 C++ 库（和 OpenGL/GLUT 等），跟 Python 没关系。

3. **“为什么同时会用两个呢？”**
   不是同时用两个 Python。之前是：

   * 你编译时链接到了系统的 `libpython3.12`（那次错了）→ 运行时和 conda 3.8 不一致 → 崩。
   * 现在改成链接 conda 3.8 了，但**运行时找不到** `libpython3.8.so.1.0` 这个具体文件名 → `not found`。
     修好文件名（软链或重配）即可。


## 方案 B（更优雅）：把“只含 libpython 的目录”写进二进制的 RUNPATH

这样以后运行就不用再带环境变量。

```bash
cd ~/Desktop/MASS/build

# 1) 准备目录（同上）
mkdir -p ./render/pyshim
cp "$CONDA_PREFIX/lib/libpython3.8.so"* ./render/pyshim/

# 2) 写入 RUNPATH：只指向 $ORIGIN/pyshim，再加系统库目录
sudo apt-get install -y patchelf
patchelf --set-rpath '$ORIGIN/pyshim:/usr/local/lib:/usr/lib/x86_64-linux-gnu' ./render/render

# 3) 验证
readelf -d ./render/render | egrep -i 'runpath|rpath'
ldd ./render/render | egrep -i 'python|stdc\+\+'

# 4) 直接运行（无需再设 LD_LIBRARY_PATH）
./render/render ../data/metadata.txt
```
(mass) jz7785@ENG-HS5894-UBUNTU:~/Desktop/MASS/build$ ldd ./render/render | egrep -i 'python|stdc\+\+'
	libpython3.8.so.1.0 => /home/jz7785/Desktop/MASS/build/./render/pyshim/libpython3.8.so.1.0 (0x00007b2b7be00000)
	libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007b2b7a800000)


---






