---
date: '2026-06-19T20:40:44+09:00'
draft: false
title: 'FXC to DXC'
---

## 동기

D3D12 엔진 개발 중, 정적으로 Root CBV에 하나만 바인딩해서 사용하던 기존 머티리얼 시스템을 확장하여 여러 머티리얼을 Root Descriptor Table로 바인딩 후 인덱싱할 수 있도록 작업을 진행했다.

```hlsl
// ps.hlsl
struct MaterialConstants
{
    float3 materialAmbient;
    float3 materialSpecular;
    float shininess;
    uint4 textureIndices;
};
ConstantBuffer<MaterialConstants> MaterialConstantBuffers[] : register(b0, space1);
```

이때 `textureIndices`의 x, y, z 컴포넌트에는 각각 albedo, normal map, height map으로 사용할 텍스처의 인덱스를 저장했다. `main` 함수에서 아래와 같이 변수를 선언했고, 이전에 구현해둔 POM (Parallax Occlusion Mapping) 함수에 `heightMapIdx`를 넘기는 형식이었다.

```hlsl
float4 main(PSInput input) : SV_TARGET
{
    uint4 textureIndices = MaterialConstantBuffers[materialIdx].textureIndices;
    uint albedoIdx = textureIndices[0];
    uint normalMapIdx = textureIndices[1];
    uint heightMapIdx = textureIndices[2];
    
    // ...
    
    float2 texCoord = ParallaxMapping(input.texCoord, toCameraTangent, heightMapIdx); 
```

그런데 이때 예상하지 못한 문제가 발생했다. `ps.hlsl`에서는 이번 커밋 이전에도 Unbounded array를 사용한 동적 인덱싱을 잘 사용해왔는데, 이번에 작업한 `ParallaxMapping` 함수 내부의 아래 line을 FXC가 처리하지 못하고 빌드 타임 에러를 발생시키는 것이었다. (사실 이 부분이 문제라는 것도 여러 시행착오 이후에 발견했다)

```hlsl
Texture2D g_textures[] : register(t0, space0);

// ...

float currentHeightMapValue = 1.0f - g_textures[heightMapIdx].SampleGrad(g_samplers[samplerIdx], currentTexCoord, dx, dy).r;
// error MSB6006: "fxc.exe" exited with code -1073741819.
```

Debug에서는 빌드/실행이 정상적으로 이루어지고 **Release에서만 빌드 시 에러가 발생**했기에, FXC의 **최적화**와 관련해서 뭔가 문제가 있으리라 추측했다.

> 종료 코드 `-1073741819`를 16진수로 변환하는 디코딩을 거치면 `0xC0000005`이 되고, 이는 [NTSTATUS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/596a1078-e883-4972-9bbc-49e60bebca55)에서 `STATUS_ACCESS_VIOLATION`를 의미한다.

그런데 `g_textures[2]`와 같이 **리터럴 상수**를 직접 인덱싱에 사용하는 경우에는 에러가 발생하지 않았다. 이는 `g_textures`에서 리소스를 동적 인덱스로 인덱싱하는 것이 트리거라는 것을 의미했다. 다만 정말 이상한 점은 `ps.hlsl`의 다른 부분에서 Unbounded array를 동적 인덱스로 인덱싱하는 다른 케이스가 있음에도 불구하고, 에러는 `g_textures`를 동적으로 인덱싱하는 경우에만 발생했다.

```hlsl
// 예외 1: g_directionalShadowMaps 인덱싱
shadowFactor += g_directionalShadowMaps[idxInArray].SampleCmpLevelZero(g_comparisonSampler0, float3(texCoord + offset, float(csmIdx)), compareValue);

// 예외 2: g_samplers 인덱싱
float3 texColor = g_textures[0].Sample(g_samplers[samplerIdx], texCoord).rgb;
```

그래서 가설을 더 좁혀서 "동적 인덱싱 + gradient 해석이 필요한 샘플링"이 문제인지 의심해보았고, LOD를 직접 지정하는 `SampleLevel`로 코드를 대체해 보았지만, 여전히 에러가 발생했다.

FXC를 직접 실행해서 컴파일해 봐도 뭔가 의미있는 로그를 뱉지않고 컴파일러가 바로 죽어버리는 상황이었기 때문에, 일단은 임시방편으로 문제가 되는 부분은 리터럴 상수를 이용해서 실행이 되도록 하였으며([해당 커밋](https://github.com/alpakaDurumi/D3D12Renderer/commit/38a7a43b47bca32f24d8b8c8060fbc4352f2dc99#diff-760ec037b6fa87d26437ffe0a3ecab2b2e6536f4e44fc552e71a4d3e5546976c)), 구식 컴파일러인 FXC 대신 d3d12에서 권장되는 DXC를 사용하면 해결이 되지 않을까 해서 마이그레이션을 진행하게 되었다.

아마 최근에 유지보수가 잘 되지 않는 FXC였기 때문에 이런 문제가 발생했던 것으로 보인다.

## FXC를 사용하던 방식

FXC에서 DXC로 마이그레이션하기 위해서는 먼저 현재 사용중인 FXC의 출처와 사용 방식에 대해 알아야 했다.

Windows SDK에서 제공되는 `d3dcompiler.lib`을 링킹하고 `D3Dcompiler.h`에 선언된 `D3DCompileFromFile(...)`로 `.hlsl` 파일을 매 실행마다 런타임에 DXBC로 컴파일하여 `ComPtr<ID3DBlob>`에 보관하여 사용한다.

```cpp
std::unordered_map<ShaderKey, ComPtr<ID3DBlob>> m_shaderBlobs;

// ...

for (const ShaderKey& key : shaderKeys)
{
    auto it = m_shaderBlobs.emplace(key, nullptr).first;
    std::vector<D3D_SHADER_MACRO> shaderMacros;
    for (const std::string& define : key.defines)
    {
        shaderMacros.push_back({ define.c_str(), NULL });
    }
    shaderMacros.push_back({ NULL, NULL });

    ComPtr<ID3DBlob> errorBlob;
    // 여기서 컴파일
    HRESULT hr = D3DCompileFromFile(key.fileName.c_str(), shaderMacros.data(), D3D_COMPILE_STANDARD_FILE_INCLUDE, "main", key.target.c_str(), compileFlags, 0, &it->second, &errorBlob);
    if (FAILED(hr))
    {
        OutputDebugStringA(static_cast<const char*>(errorBlob->GetBufferPointer()));
        throw std::exception();
    }
}
```

> - `d3dcompiler.lib`은 *import library*로, 보통 내부에 `d3dcompiler_47.dll`을 로드하라는 정보가 들어있다.
> - `d3dcompiler_47.dll`의 숫자 47은 버전을 의미함. 현재 FXC의 버전은 사실상 47에서 멈춘 상태로, Windows SDK에는 이 버전이 배포된다.

이 방식에는 또 다른 문제점이 있는데, `vcxproj` 파일에 `.hlsl` 파일이 `<FxCompile Include="PointLightShadowPS.hlsl">` 이런 식으로 작성되어 있어 런타임에 진행하는 위 단계와 별개로 MSBuild가 빌드 타임에 `fxc.exe`로 중복으로 컴파일해서 `.cso`를 생성하고 있었다.

## DXC로의 전환

FXC를 제거하기 전에, DXC를 가져올 출처와 사용할 방식에 대해 결정해야했다. Windows SDK에 DXC가 제공되긴 하지만, 최신 릴리즈와는 버전 차이가 존재했기 때문에 Nuget으로 명확한 버전의 DXC를 사용하기로 결정했다.

[이 커밋](https://github.com/alpakaDurumi/D3D12Renderer/commit/ed1b66d62cc83e826c2de19d5a0a4df34ba3fd6f)에서는 Nuget에 DXC 패키지를 추가하고 `vcxproj`를 수정, 그리고 `targets` 파일을 추가해 기존 MSBuild에서 `.hlsl` 파일에 대해 수행하던 FXC 사용을 DXC 사용으로 교체했다. `CustomBuild` 내에서 `Outputs`를 지정해 incremental build를 하게끔 설정했다.

기존 런타임 FXC 사용 경로는 건드리지 않았기 때문에, 여전히 실행에는 문제가 없다. 또한 빌드 과정에서 발생하는 산출물들을 지정된 디렉토리에 출력하도록 정리했다.

다음으로는 빌드 타임에 DXC가 컴파일한 결과인 DXIL을 런타임에 읽어서 사용하도록 로직을 교체했으며, 기존에 FXC를 사용하기 위한 설정들을 제거했다. ([해당 커밋](https://github.com/alpakaDurumi/D3D12Renderer/commit/ad0952a247772c5f39cbba5ca106c724b5c28437))

```cpp
// 자료구조 타입도 char로 교체
std::unordered_map<ShaderKey, std::vector<char>> m_shaderBlobs;

// ...

// Read shaders
// 이제 런타임 컴파일 없이, 빌드 타임에 컴파일해둔 DXIL을 읽기만 하면 된다.
{
    std::wstring vsName = L"vs.hlsl";
    std::wstring psName0 = L"ps.hlsl";
    std::wstring psName1 = L"PointLightShadowPS.hlsl";

    std::vector<std::wstring> csoNames;

    // VS
    for (UINT i = 0; i < static_cast<UINT>(MeshType::NUM_MESH_TYPES); ++i)
    {
        for (UINT j = 0; j < static_cast<UINT>(PassType::NUM_PASS_TYPES); ++j)
        {
            std::wstring temp = Utility::RemoveFileExtension(vsName);
            if (i == 1)
                temp += L"_instanced";
            if (j == 1)
                temp += L"_depth_only";
            temp += L".cso";

            csoNames.push_back(temp);
        }
    }

    // PS
    csoNames.push_back(Utility::RemoveFileExtension(psName0) + L".cso");
    csoNames.push_back(Utility::RemoveFileExtension(psName1) + L".cso");

    for (const auto& name : csoNames)
    {
        std::ifstream file(std::filesystem::path(L"shaders") / name, std::ios::binary);
        if (!file)
        {
            throw std::runtime_error("Failed to open .cso file.");
        }

        m_shaderBlobs.try_emplace(ShaderKey{ name }, std::istreambuf_iterator<char>(file), std::istreambuf_iterator<char>());
    }
}
```

마지막으로, DXC로의 전환을 하게 된 계기인 쉐이더 내 인덱싱 부분을 다시 복구했고, 정상 동작을 확인했다: [해당 커밋](https://github.com/alpakaDurumi/D3D12Renderer/commit/aa84a599806e637550679ef34b404648c639d4b5)

## 결과

d3d12에서 권장되는 DXC로 전환함으로써 원인을 명확히 알 수 없었던 동적 인덱싱 에러를 해결하였다. 또한 빌드 타임과 런타임에 중복으로 컴파일하던 과정을 간소화했다.

기존에는 hlsl 컴파일 시 target을 Unbounded array 사용에 필요한 조건이었던 shader model 5.1로 설정했으나, DXC에서는 사용 기능과 무관하게 최소 shader model 6.0으로 설정해야 한다는 차이가 있다.

쉐이더를 사용하기 위한 두 단계 중 첫 번째 단계(HLSL -> DXIL)은 갖춰졌지만, 두 번째 단계(DXIL -> GPU ISA)는 캐싱이 구현되지 않은 상태다. 두 번째 단계도 사실 두 갈래로 나뉜다. GPU vendor 또는 드라이버가 자동으로 수행하는 캐싱이 있고, 앱 개발자가 파이프라인을 구현해놓는 것이 있다. (둘 중 하나를 선택하는 개념이 아니며, 앱에서 구현 시 드라이버 캐시를 보완하는 형태). 엔진을 더욱 고도화 시키고 나서 이 방법을 도입하는 것도 괜찮을 것 같다.
