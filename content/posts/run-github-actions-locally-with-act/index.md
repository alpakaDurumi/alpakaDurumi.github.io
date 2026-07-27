---
date: '2026-07-28T02:18:00+09:00'
draft: false
title: 'GitHub Actions를 로컬에서: act'
---

## 개요

오픈소스에 기여할 때 PR을 날리면 CI가 자동으로 돌아가는데, 생각보다 상당한 시간이 걸리는 것을 볼 수 있다. 특히 workflow 파일 자체를 수정하는 작업이라면 이 사이클이 더 귀찮아질 수 있다.

이번에 ThorVG의 바이너리 사이즈 측정 워크플로우인 `binary_size.yml`을 수정하는 PR을 작업하면서, 로컬에서 GitHub Actions를 돌릴 수 있는 도구가 없나 찾다가 발견한 도구가 바로 [act](https://github.com/nektos/act)였다.

`act`를 사용한 환경은 Windows 11 + WSL이다.

## act 소개

`act`는 GitHub Actions 워크플로우를 **로컬 컴퓨터에서 그대로 실행**해볼 수 있게 해주는 툴이다. Docker 컨테이너를 이용해서 실제 GitHub Actions 러너와 최대한 비슷한 환경을 로컬에 만들고, 그 안에서 `.github/workflows/` 안의 workflow를 실행시켜 준다.

즉, **push 하기 전에** 워크플로우가 제대로 도는지 내 컴퓨터에서 미리 확인할 수 있다.

사용법에 대해서는 [act user guide](https://nektosact.com/)를 참고하자.

## 왜 필요했나

[이번에 작업한 PR](https://github.com/thorvg/thorvg/pull/4603)은 바이너리 사이즈 측정 워크플로우가 `wg`(WebGPU) 엔진 관련 변경사항을 제대로 반영하지 못하고 있는 문제를 고치는 작업이었다. 워크플로우가 `-Dengines="cpu, gl"`로만 빌드하고 있어서 wg 엔진 소스는 애초에 컴파일 대상이 아니었고, 그래서 wg를 건드리는 PR은 항상 delta 0으로 보고되고 있었다.

`.yml` 파일을 수정하는 것 자체는 어렵지 않았다. 어려운 쪽은 '고쳐졌다는 것을 어떻게 증명하느냐'였다. 그래서 wg 엔진 코드에 크기를 미리 아는 probe를 추가하고 실제로 워크플로우가 delta를 잘 출력하는지 검증하는 과정이 필요했는데, 매번 PR을 넣어서 이걸 테스트하기에는 상당히 번거롭고 시간이 많이 걸릴 것이라고 판단했다.

## 사용

우선 wg 엔진 코드 중 하나인 `tvgWgRenderer.cpp`에 아래와 같이 4KB 크기의 배열을 추가하였다.

```cpp
extern "C" const char tvg_size_probe[4096] = "probe";
```

그리고 아래와 같은 명령을 통해 지정된 워크플로우를 실행한다.

```bash
rm -rf /tmp/art-x86_64-gcc
act pull_request -W .github/workflows/binary_size.yml -j build \
  --matrix key:x86_64-gcc \
  -P ubuntu-24.04=catthehacker/ubuntu:act-24.04 \
  --artifact-server-path /tmp/art-x86_64-gcc
```

이때 `x86_64-gcc` 부분은 어떤 config를 사용하느냐에 따라 달라진다. 그리고 앞의 `rm -rf`는 이전 실행에서 남은 아티팩트가 섞이지 않게 하기 위함이다.

### --matrix

matrix에 정의된 조합 중 어떤 것을 실행할지 `필드이름:값` 형태로 지정한다. `binary_size.yml`의 각 matrix 항목이 `key: x86_64-gcc`처럼 `key` 필드로 구분되고 있기 때문에 `key:`가 붙는다.

이 옵션이 없으면 matrix에 정의된 4개 config가 전부 돌아간다. 굳이 `--matrix`를 지정하여 따로 돌리는 이유는 `arm64` config 때문이다. (아래에서 설명)

### -P

`.yml` 파일 내에서 `runs-on`으로 명시된 플랫폼을 `act`에서 지원하는 어떤 플랫폼에 대응시킬지 결정한다.

난 `arm64` config를 사용하는 경우에 대해서만 `-P` 이외에 `--container-architecture linux/arm64`을 추가로 명시해야 했다. `catthehacker/ubuntu:act-24.04`가 multi-arch 이미지라 `arm64` variant도 함께 갖고 있긴 하지만, `x86_64` 머신에서 그냥 받아오면 도커가 호스트 아키텍처에 맞는 `x86_64` variant를 골라오기 때문이다.

### --artifact-server-path

GitHub Actions에서 잡·워크플로우 간에 아티팩트를 주고받는 방식을 흉내내기 위한 것.

본래 `binary_size.yml`은 분석 결과를 아래 부분을 통해 업로드하며:

```yml
    - uses: actions/upload-artifact@v4
      if: always()
      with:
        name: size-${{ matrix.key }}
        path: ci-report/
```

또 다른 워크플로우인 `binary_size_report.yml`에서 다운로드해 정리하여 출력하는 방식이다.

```yml
    - name: Download size artifacts
      uses: actions/download-artifact@v4
      with:
        path: ci-report
        pattern: size-*
        merge-multiple: true
        run-id: ${{ github.event.workflow_run.id }}
        github-token: ${{ secrets.GITHUB_TOKEN }}
```

`act`는 `--artifact-server-path` 옵션으로 지정한 디렉토리에 결과 `.zip` 파일을 저장하도록 동작을 흉내낼 수 있다.

## 검증 결과

PR을 넣기 위해서는 두 가지를 확인해야 했다.

1. `binary_size.yml`만 수정했을 때 delta가 0이 맞는가?
2. probe를 추가했을 때 wg engine을 사용하는 3개의 config에서 4KB만큼 크기가 증가하는가?

1번은 워크플로우가 출력한 delta가 네 config에 대해 모두 0인 것을 확인했다. 실제 PR에 달린 Binary Size Report도 같다.

2번은 probe를 넣기 전(즉, 1번)과 후로 각각 한 바퀴씩 돌려 비교했다.

| Config | base text | base data | probe text | probe data | Delta |
|--------|----------:|----------:|-----------:|-----------:|------:|
| arm64-clang | 960,799 | 25,216 | 964,895 | 25,216 | +4096 (+0.42%) |
| x86-gcc | 899,065 | 11,620 | 899,065 | 11,620 | 0 (0.00%) |
| x86_64-clang | 975,686 | 23,568 | 979,798 | 23,568 | +4112 (+0.41%) |
| x86_64-gcc | 982,463 | 23,432 | 986,591 | 23,432 | +4128 (+0.41%) |

base text/data는 `binary_size.yml`만 수정한 결과이며, probe text/data는 거기에 probe까지 추가한 결과다. wg 엔진이 켜진 세 config에서만 4KB 남짓 크기가 증가하는 것을 볼 수 있다.

> 32비트 config만 0으로 남은 것은 `wgpu-native`가 32비트 리눅스 빌드를 제공하지 않아 이 config에서는 wg 엔진을 의도적으로 제외했기 때문인데, 의도한 대로 동작하고 있다는 것까지 로컬에서 확인할 수 있었다.

다만 실제로 GitHub Actions의 결과와 비교했을 땐 `arm64`만 4바이트가 컸는데, 이에 대해서는 명확한 원인을 확인하지 못했다. 정황상 `act` 사용 시 `x86_64`에서 `arm64`를 에뮬레이션하기 위해 QEMU를 거치는 것이 이유가 아닌가 추측되지만, 확실하지는 않다.

## 좋았던 점

- PR을 넣지 않더라도 로컬에서 파일 수정 후 빠르게 결과를 확인해볼 수 있다.
- PR에 검증 근거를 남기기에 좋다.
- CI는 공유 자원이므로, 로컬에서 최대한 문제를 걸러내고 제출한다면 프로젝트에 도움이 된다.

## 아쉬운 점

물론 완벽한 도구는 아니다. Docker 기반이라 완전히 GitHub 호스팅 러너와 동일한 환경은 아니며, 테스트해보니 x86_64 머신에서 `act`를 사용하여 arm64를 테스트하고자 하는 경우 QEMU 에뮬레이션으로 돌려야 해서 속도가 느렸다.

## 마무리

`act`가 GitHub Actions와 완전히 같은 환경을 제공해주지는 않는다. 그래도 "일단 로컬에서 큰 틀이 맞는지 확인하고 push한다"는 흐름만으로도 충분히 값어치가 있었다. 워크플로우 수정, 인프라 관련 작업 등을 할 계획이 있다면 한 번 써보시는 것을 추천드린다.
