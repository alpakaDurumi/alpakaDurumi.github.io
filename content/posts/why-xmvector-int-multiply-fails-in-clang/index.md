---
date: '2026-06-29T21:01:17+09:00'
draft: false
title: 'XMVECTOR와 int 간 곱셈이 clang에서 실패하는 이유'
---

## 증상

프로젝트의 빌드 시스템을 Visual Studio 종속에서 CMake로 마이그레이션하며 컴파일러도 이 참에 더 엄격한 clang-cl로 바꾸기로 했다. 그러던 와중 기존 MSVC에서는 잘 동작하던 코드에서 문제가 발생했다.

아래 코드는 카메라를 Panning 시키는 함수로, `DirectXMath`의 `XMVECTOR` 타입을 사용한다.

```cpp
// Camera.cpp
void Camera::Pan(XMINT2 mouseMove)
{
    constexpr float panSensitivity = 0.01f;

    MoveRight(mouseMove.x * panSensitivity);

    XMVECTOR pos = XMLoadFloat3(&m_currPosition);
    XMVECTOR rot = XMLoadFloat4(&m_rotation);
    XMVECTOR localUp = XMVector3Rotate(XMVectorSet(0.0f, 1.0f, 0.0f, 0.0f), rot);

    pos += -localUp * mouseMove.y * panSensitivity;    // 이 부분에서 문제 발생

    XMStoreFloat3(&m_currPosition, pos);
}
```

```text
error: cannot convert between scalar type 'int32_t' (aka 'int') and
       vector type 'XMVECTOR' (aka '__m128') as implicit conversion would cause truncation
```

이는 clang-cl과 clang++ 양쪽 모두에서 발생했으며, `localUp`(`XMVECTOR`)과 `mouseMove.y`(`int`) 간 `*` 연산자가 예상한 대로 동작하지 않는다는 신호였다.

## 근본 원인: `__m128` 정의가 컴파일러마다 다름

`XMVECTOR` 타입은 `_XM_SSE_INTRINSICS_` 매크로가 정의되어있는 경우 `__m128` 타입에 대한 aliasing이다.

```cpp
    //------------------------------------------------------------------------------
    // Vector intrinsic: Four 32 bit floating point components aligned on a 16 byte
    // boundary and mapped to hardware vector registers
#if defined(_XM_SSE_INTRINSICS_) && !defined(_XM_NO_INTRINSICS_)
    using XMVECTOR = __m128;
#elif defined(_XM_ARM_NEON_INTRINSICS_) && !defined(_XM_NO_INTRINSICS_)
    using XMVECTOR = float32x4_t;
#else
    using XMVECTOR = __vector4;
#endif
```

`_XM_SSE_INTRINSICS_` 매크로는 Windows의 경우 기본적으로 활성화된다. (ARM 프로세서를 쓰는 경우 등 제외)

<details>
<summary>자세한 판정 로직</summary>

```cpp
// DirectXMath.h
#if !defined(_XM_ARM_NEON_INTRINSICS_) && !defined(_XM_SSE_INTRINSICS_) && !defined(_XM_NO_INTRINSICS_)
#if (defined(_M_IX86) || defined(_M_X64) || __i386__ || __x86_64__) && !defined(_M_HYBRID_X86_ARM64) && !defined(_M_ARM64EC)
#define _XM_SSE_INTRINSICS_
#elif defined(_M_ARM) || defined(_M_ARM64) || defined(_M_HYBRID_X86_ARM64) || defined(_M_ARM64EC) || __arm__ || __aarch64__
#define _XM_ARM_NEON_INTRINSICS_
#elif !defined(_XM_NO_INTRINSICS_)
#error DirectX Math does not support this target
#endif
#endif // !_XM_ARM_NEON_INTRINSICS_ && !_XM_SSE_INTRINSICS_ && !_XM_NO_INTRINSICS_
```

</details>
<br>

이 `__m128` 타입의 구현은 **컴파일러에 따라 다르다.**

### MSVC

```cpp
// xmmintrin.h (MSVC에서 제공되는 버전)
typedef union __declspec(intrin_type) __declspec(align(16)) __m128 {
     float               m128_f32[4];
     unsigned __int64    m128_u64[2];
     __int8              m128_i8[16];
     __int16             m128_i16[8];
     __int32             m128_i32[4];
     __int64             m128_i64[2];
     unsigned __int8     m128_u8[16];
     unsigned __int16    m128_u16[8];
     unsigned __int32    m128_u32[4];
 } __m128;
```

타입이 `union`으로, **class type**이다. 그래서

```cpp
// DirectXMathVector.inl
inline XMVECTOR XM_CALLCONV operator*
(
    FXMVECTOR      V,
    const float    S
) noexcept
{
    return XMVectorScale(V, S);
}
```

이런 식으로 연산자 오버로딩이 정의될 수 있고, `XMVECTOR`와 `int`의 곱셈은 문제가 되지 않는다.

### clang

```cpp
// xmmintrin.h (clang에서 제공되는 버전)
typedef float __m128 __attribute__((__vector_size__(16), __aligned__(16)));
```

반면 clang에서는 class type이 아니라 그 자체로 raw vector를 의미한다. (GCC에서도 동일한 것으로 알고있다)

C++ 규칙상 비멤버 operator는 **최소 한 인자가 user-defined type** 이어야 하기 때문에, clang에서는 오버로딩 자체가 불가능하다.

## DirectXMath의 대응

그래서 DirectXMath는 **clang/GCC에서 연산자 오버로딩을 통째로 끈다**:

```cpp
// DirectXMath.h
#if !defined(_XM_NO_XMVECTOR_OVERLOADS_) && (defined(__clang__) || defined(__GNUC__)) && !defined(_XM_NO_INTRINSICS_)
#define _XM_NO_XMVECTOR_OVERLOADS_
#endif

// ...

#ifndef _XM_NO_XMVECTOR_OVERLOADS_
    XMVECTOR    XM_CALLCONV     operator+ (FXMVECTOR V) noexcept;
    XMVECTOR    XM_CALLCONV     operator- (FXMVECTOR V) noexcept;

    XMVECTOR&   XM_CALLCONV     operator+= (XMVECTOR& V1, FXMVECTOR V2) noexcept;
    XMVECTOR&   XM_CALLCONV     operator-= (XMVECTOR& V1, FXMVECTOR V2) noexcept;
    XMVECTOR&   XM_CALLCONV     operator*= (XMVECTOR& V1, FXMVECTOR V2) noexcept;
    XMVECTOR&   XM_CALLCONV     operator/= (XMVECTOR& V1, FXMVECTOR V2) noexcept;

    XMVECTOR& operator*= (XMVECTOR& V, float S) noexcept;
    XMVECTOR& operator/= (XMVECTOR& V, float S) noexcept;

    XMVECTOR    XM_CALLCONV     operator+ (FXMVECTOR V1, FXMVECTOR V2) noexcept;
    XMVECTOR    XM_CALLCONV     operator- (FXMVECTOR V1, FXMVECTOR V2) noexcept;
    XMVECTOR    XM_CALLCONV     operator* (FXMVECTOR V1, FXMVECTOR V2) noexcept;
    XMVECTOR    XM_CALLCONV     operator/ (FXMVECTOR V1, FXMVECTOR V2) noexcept;
    XMVECTOR    XM_CALLCONV     operator* (FXMVECTOR V, float S) noexcept;
    XMVECTOR    XM_CALLCONV     operator* (float S, FXMVECTOR V) noexcept;
    XMVECTOR    XM_CALLCONV     operator/ (FXMVECTOR V, float S) noexcept;
#endif /* !_XM_NO_XMVECTOR_OVERLOADS_ */

using XMVECTOR = __m128;
```

즉 `operator* → XMVectorScale` 경로는 **MSVC 전용**이고, clang 빌드에는 그런 경로가 존재하지 않는다. clang에서는 `operator*`를 built-in vector 산술 연산으로만 인식하게 되고, 해당 연산은 `XMVECTOR`와 `float`를 명시적으로 요구한다. 이때 `int`를 전달하게 되면 clang에서 에러를 내뿜는 것.

## 해결

두 가지 방법이 존재한다. 난 2번 방법을 택했다: [해당 커밋](https://github.com/alpakaDurumi/D3D12Renderer/commit/0f81fc2214f86cfeedd482cd9c64f8874248c238)

### 1

정수 스칼라를 **float**로 만들어 `XMVECTOR * float`가 되게 한다.

```cpp
pos += -localUp * (mouseMove.y * panSensitivity);   // 스칼라끼리 먼저 → float
// 또는 static_cast<float>(mouseMove.y)
```

### 2

또는 `XMVectorScale`같은 함수를 명시적으로 사용하면 된다. 

```cpp
pos += XMVectorScale(localUp, -mouseMove.y * panSensitivity);
```

`-mouseMove.y * panSensitivity`가 먼저 `float` 값으로 평가되기 때문에 정상적인 경로로 실행된다.

> `XMVectorScale` 내부 동작을 따라가보면 `_mm_set_ps1`과 `_mm_mul_ps` 인트린직을 사용하는데, `_mm_set_ps1`는 하나의 float를 4개 슬롯에 복사하는 동작이며 `_mm_mul_ps`이 곱셈이다. `_mm_mul_ps`은 내부적으로 clang의 빌트인 벡터 산술 연산을 호출한다.

## 마무리

이 문제를 만나기 전에는 단순히 '오버로딩이 정의되어있으면 굳이 긴 함수 이름 타이핑 칠 필요가 없구나' 라고 생각했었지만, 사실 그건 MSVC에 국한된 이야기였다. 이래서 여러 시도를 해 봐야하나 보다.
