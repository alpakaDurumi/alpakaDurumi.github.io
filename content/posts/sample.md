---
title: "샘플 글: 렌더 확인용"
date: 2026-06-17
draft: false
tags: ["test"]
---

이 글은 Hugo + PaperMod 렌더링이 제대로 도는지 확인하기 위한 샘플입니다. 발행 전 삭제 예정.

## 코드 하이라이트 테스트 (HLSL)

```hlsl
ConstantBuffer<MaterialData> materials[] : register(b0, space1);

float4 main(PSInput input) : SV_Target {
    uint idx = input.materialIndex; // 변수 인덱싱
    return materials[idx].baseColor;
}
```

## 한국어 + 영어 혼용 테스트

이 문단은 한국어와 English를 섞어서 `hasCJKLanguage` 글자수 계산이 맞는지 확인합니다.
목차(ToC), 읽는 시간, 코드 복사 버튼이 잘 뜨는지도 같이 봅니다.
