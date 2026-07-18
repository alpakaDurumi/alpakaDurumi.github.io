---
date: '2026-07-18T18:32:07+09:00'
draft: false
title: 'clangd가 #include를 제대로 처리 못하는 경우'
---

## 증상

MSYS2 CLANG64 환경에서 [ThorVG Example](https://github.com/thorvg/thorvg.example) 프로젝트를 살펴보던 중, **clangd**가 제대로 작동하지 않는 문제를 발견했다. 빌드와 실행 모두 잘 되지만, clangd가 `#include <thorvg-1/thorvg.h>`, `#include <SDL2/SDL.h>` 등의 `#include` 문을 제대로 처리하지 못하고 있었다.

## 원인

알고보니 당시 사용하던 clangd가 MSYS2의 것이 아닌, vscode-clangd 확장이 자동으로 설치한 번들 버전이었다. 현재 내 환경인 MSYS2 CLANG64에서 사용해야 할 clangd의 target은 `x86_64-w64-windows-gnu`지만, 사용중인 번들 clangd는 target이 `x86_64-pc-windows-msvc`로 서로 달랐다.

```bash
# MSYS2 CLANG64에서 clangd --version 을 실행했을 때
clangd version 22.1.8 (https://github.com/msys2/MINGW-packages 0e49d538be8f77422f9f2fbff025394903669d26)
Features: windows
Platform: x86_64-w64-windows-gnu
```

```bash
# vscode-clangd의 번들 clangd.exe에 대해 --version을 실행했을 때
clangd version 22.1.8 (https://github.com/llvm/llvm-project ca7933e47d3a3451d81e72ac174dcb5aa28b59d1)
Features: windows
Platform: x86_64-pc-windows-msvc
```

Target triple의 마지막 부분인 environment는 환경 및 ABI를 나타내며, 이것이 달라지면 참조해야 하는 헤더 파일과 라이브러리 파일의 경로가 변경된다. 예를 들어:

- `x86_64-w64-windows-gnu`은 sysroot를 기준으로 `include` 디렉토리를 찾는다. MSYS2 CLANG64라면 `.../msys64/clang64/include`를 include directories에 추가한다.
- `x86_64-pc-windows-msvc`는 Visual Studio와 Windows SDK의 경로를 추가한다.

> 현재 컴파일러에서 헤더 파일을 찾는 경로가 궁금하다면, `<compiler> -E -x c++ -v /dev/null`를 실행해보자.
> 
> - `-E`: 전처리만 수행하고 컴파일은 하지 않음
> - `-x c++`: 입력 파일을 cpp 소스 파일로 간주
> - `-v`: verbose
> - `/dev/null`: 내용이 없는 빈 입력 파일

종합하면: 난 MSYS2 CLANG64에 환경을 구성했기 때문에 `x86_64-w64-windows-gnu`를 대상으로 하는 clangd를 사용해야 했지만, `x86_64-pc-windows-msvc`를 대상으로 동작하는 clangd를 사용하여 `.../msys64/clang64/include`에 위치한 헤더 파일을 찾지 못한 것.

## 해결

vscode-clangd 확장의 설정에서 사용할 clangd의 경로를 설정해주면 된다. 각 프로젝트가 어떤 target 대상으로 빌드되느냐에 따라 적절하게 지정하자. 아래는 내 컴퓨터에서 설정한 예시.

### `x86_64-w64-windows-gnu`

```json
{
    "clangd.path": "C:/msys64/clang64/bin/clangd.exe"
}
```

### `x86_64-pc-windows-msvc`

```json
{
    "clangd.path": "C:/Program Files/LLVM/bin/clangd.exe"
}
```
