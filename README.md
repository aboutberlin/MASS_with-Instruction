
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

---

需要我把你当前仓库的 `CMakeLists.txt` 做一次最小改动的 patch（只加 C++17 的两行）吗？你把三个 CMakeLists（根、core、render、python）贴上来，我给你打一个干净的补丁，后续拉别人代码也能直接套。
