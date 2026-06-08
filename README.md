# Лабораторная работа №6

## CMake

В CMakeLists ничего существенно нового, кроме переменных MAJOR, MINOR, PATCH и TWEAK, которые отвечают за версию релиза, если более подробнее, то
изменения, вносящиеся в проект(итоговый пакет) должны помечаться, для этого созданы так называемые тэги, удобный маркер, по которому можно сказать что 
какого рода изменения были внесены в проект. Так же директива install указывает, какие файлы нужно включить в пакет — в нашем случае это исполняемый файл solver, который копируется в директорию bin.

MAJOR - Крупные изменения(полная переработка)
MINOR - Новые возможности(добавление нового функционала с совместимостью например)
PATCH - Исправления(пофиксили баги и т.д. без изменения функционала)
TWEAK - Номер сборки(при незначительных изменениях, например документацию дополнили)

```cmake
cmake_minimum_required(VERSION 3.10)
project(lab06)

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

install(TARGETS solver DESTINATION bin COMPONENT applications)

include(CPackConfig.cmake)
```

## CPackConfig

В конце подключается файл конфигурации CPack, который мы прописаваем отдельно в CPackConfig.
Сначала настраиваем доп информацию: версия пакета, контакт разработчика, описание, потом настраиваются параметры для RPM и DEB пакетов. 
В конце используется условная компиляция — для каждой ОС выбирается свой генератор пакетов: WIX для Windows, DragNDrop для macOS, DEB/RPM/TGZ/ZIP для Linux.

```cmake
include(InstallRequiredSystemLibraries)

set(CPACK_RESOURCE_FILE_README ${CMAKE_SOURCE_DIR}/README.md)

set(CPACK_PACKAGE_CONTACT serezapovisok@gmail.com)
set(CPACK_PACKAGE_VERSION_MAJOR ${PRINT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PRINT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PRINT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_TWEAK ${PRINT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION ${PRINT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "solver library")

set(CPACK_RPM_PACKAGE_NAME "solver")
set(CPACK_RPM_PACKAGE_GROUP "solver")
set(CPACK_RPM_PACKAGE_RELEASE 1)
set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")

set(CPACK_DEBIAN_PACKAGE_NAME "libsolver-dev")
set(CPACK_DEBIAN_PACKAGE_PREDEPENDS "cmake >= 3.0")
set(CPACK_DEBIAN_PACKAGE_RELEASE 1)

if(WIN32)
    set(CPACK_GENERATOR "WIX")
    set(CPACK_WIX_ARCHITECTURE "x64")
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

## ci.yaml

Вообще Workflow состоит из трёх этапов:
1) Build — собирает проект на Ubuntu и загружает артефакты
2) Test — скачивает артефакты и запускает базовые тесты
3) Create binary packages — использует матрицу для параллельной сборки на трёх ОС (Ubuntu, macOS, Windows). Для каждой системы устанавливаются нужные зависимости (rpm для Ubuntu, .NET и WIX для Windows), затем собирается проект и создаются пакеты под каждую OS. В конце все пакеты загружаются в GitHub Releases через action softprops/action-gh-release.

Собственно, workflow запускается вручную или при пуше тега в удаленный репо.


```yaml
name: CI + CPACK
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
        uses: actions/checkout@v4
      - name: Configure build directory
        run: cmake -H. -B build
      - name: Build project
        run: cmake --build build
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
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
        uses: actions/download-artifact@v4
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
        uses: actions/checkout@v4
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
      - name: Configure build directory on Ubuntu and MacOS
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
      - name: Upload packages
        uses: softprops/action-gh-release@v2
        with:
          files: |
            build/*.deb
            build/*.rpm
            build/*.dmg
            build/*.msi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
