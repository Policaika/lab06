# Лабораторная работа №6

Цель данной лабораторной работы — ознакомление с инструментарием для создания пакетов (на примере CPack).

Для создания релиза проекта нужны файлы CMake и файл CI.

### CMake

Псмотрим на главный `CMakeLists.txt` всего проекта:
```cmake
cmake_minimum_required(VERSION 3.10)
project(examples)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(PRINT_VERSION_MAJOR 1)
set(PRINT_VERSION_MINOR 0)
set(PRINT_VERSION_PATCH 0)
set(PRINT_VERSION_TWEAK 0)
set(PRINT_VERSION ${PRINT_VERSION_MAJOR}.${PRINT_VERSION_MINOR}.${PRINT_VERSION_PATCH}.${PRINT_VERSION_TWEAK})
set(PRINT_VERSION_STRING "v${PRINT_VERSION}")

add_subdirectory(formatter_lib)
add_subdirectory(formatter_ex_lib)
add_subdirectory(solver_lib)
add_subdirectory(hello_world_application)
add_subdirectory(solver_application)

install(TARGETS solver
        DESTINATION bin
        COMPONENT applications)

include(CPackConfig.cmake)
```

водится задание группы переменных, хранящих информацию о текущей версии проекта, добавляется инструкция `install`, копирующая исполняемый файл `solver` в каталог `bin` (необходимо для функционирования CPack), а также подключается конфигурационный файл для пакетов, создаваемых CPack. В этом файле прописано следующее:
```cmake
include(InstallRequiredSystemLibraries)

set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_SOURCE_DIR}/LICENSE)
set(CPACK_RESOURCE_FILE_README ${CMAKE_SOURCE_DIR}/README.md)

set(CPACK_PACKAGE_CONTACT rchekr@yandex.ru)
set(CPACK_PACKAGE_VERSION_MAJOR ${PRINT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PRINT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PRINT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_TWEAK ${PRINT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION ${PRINT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "solver library")

set(CPACK_RPM_PACKAGE_NAME "solver")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_PACKAGE_GROUP "solver")
set(CPACK_RPM_PACKAGE_RELEASE 1)
set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")

set(CPACK_DEBIAN_PACKAGE_NAME "libsolver-dev")
set(CPACK_DEBIAN_PACKAGE_PREDEPENDS "cmake >= 3.0")
set(CPACK_DEBIAN_PACKAGE_RELEASE 1)

if(WIN32)
    set(CPACK_GENERATOR "WIX")
    set(CPACK_WIX_ARCHITECTURE "x64")
    set(CPACK_WIX_PRODUCT_GUID "05b23be7-c85f-40fb-942c-7a678101c21a")
    set(CPACK_WIX_LICENSE_RTF "${CMAKE_SOURCE_DIR}/LICENSE.rtf")
    set(CPACK_WIX_PROGRAM_MENU_FOLDER "out")
    set(CPACK_WIX_VERSION "4")
    set(CPACK_COMPONENTS_ALL applications)
elseif(APPLE)
    set(CPACK_GENERATOR "DragNDrop")
    set(CPACK_DMG_VOLUME_NAME "solver")
    set(CPACK_DMG_FORMAT "UDBZ")
else()
    set(CPACK_GENERATOR "DEB;RPM;TGZ;ZIP")
endif()

include(CPack)
```

Здесь сперва определяются глобальные параметры Cpack: версия проекта, его имя, пути и прочие общие настройки. После этого для каждой операционной системы индивидуально прописываются параметры соответствующих форматов архивов (например, для `.deb`, `.rpm`, `.dmg`, `.msi` и т.д.).

В файл CI.yaml запишем следующее:
```yaml
name: 1
on:
  push:
    tags: [ "v*" ]
  workflow_dispatch:


permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Get repo files
        uses: actions/checkout@v6
      - name: Configure build directory
        run: cmake -H. -B build
      - name: Build project
        run: cmake --build build
      - name: Upload artifacts
        uses: actions/upload-artifact@v6
        with:
          name: Applications
          path: |
            ./build/hello_world_application/hello_world
            ./build/solver_application/solver

  test:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v6
        with:
          name: Applications
          path: ./apps
      - name: Fix permissions
        run: |
          chmod +x ./apps/hello_world_application/hello_world
          chmod +x ./apps/solver_application/solver
      - name: Test hello_world
        run: ./apps/hello_world_application/hello_world
      - name: Test solver
        run: echo "1 5 -6" | ./apps/solver_application/solver

  create-binary-packages:
    needs: test
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]

    steps:
      - name: Get repo files
        uses: actions/checkout@v6
      - name: Install required dependencies Ubuntu
        if: matrix.os == 'ubuntu-latest'
        run: |
          sudo apt-get update
          sudo apt-get install -y rpm
      - name: Set up DotNet Windows
        if: matrix.os == 'windows-latest'
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.x'
      - name: Install required dependencies Windows
        if: matrix.os == 'windows-latest'
        run: |
          dotnet tool install --global wix --version 4.0.0
          wix extension add --global WixToolset.UI.wixext/4.0.4
      - name: Configure build directory Windows
        if: matrix.os == 'windows-latest'
        run: |
          mkdir build
          cmake -B build
      - name: Configure build directory Ubuntu and MacOS
        if: matrix.os != 'windows-latest'
        run: cmake -H. -B build
      - name: Build project
        run: cmake --build build --config Release
      - name: Create packages DEB RPM
        if: matrix.os == 'ubuntu-latest'
        run: |
          cd build
          cpack -G DEB -C Release
          cpack -G RPM -C Release
      - name: Create packages DMG
        if: matrix.os == 'macos-latest'
        run: |
          cd build
          cpack -G DragNDrop -C Release
      - name: Create packages MSI
        if: matrix.os == 'windows-latest'
        run: |
          cd build
          cpack -G WIX -C Release
      - name: Upload binary packages
        uses: softprops/action-gh-release@v1
        with:
          files: |
            build/*.deb
            build/*.rpm
            build/*.dmg
            build/*.msi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Этот скрипт состоит из двух этапов: предварительная сборка с микро-тестированием и сборка бинарных пакетов. Сначала задаётся условие успешного выполнения джоба `test` и создаётся матрица с тремя операционными системами, так как часть кода повторяется. Затем для каждой ОС индивидуально устанавливаются необходимые зависимости и выполняется сборка программы с учётом особенностей системы. После этого генерируются артефакты программы в различных форматах: `.deb` и `.rpm` для Linux, `.dmg` для `macOS`, `.msi` для Windows. Далее с помощью действия `action-gh-release` формируется релиз из полученных архивов. На этом этапе автоматически создаются также архивы исходного кода в форматах `.zip` и `.tar.gz`, поэтому дополнительных действий для них не требуется.
