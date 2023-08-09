---
title: 파이썬에서의 진법 변환
date: 2023-03-23 20:52:00 +0900
categories: [Python]
math: True
---

## 배경

본 포스팅에는 파이썬에서의 내장 함수를 이용한 진법 변환 방법에 대하여 서술하였다.

## 10진수 $\rightarrow$ n진수

### bin(), oct(), hex()

파이썬의 내장 함수 중 `bin()`/`oct()`/`hex()`는 각각 주어진 정수를 `"0b"`/`"0o"`/`"0x"`로 시작하는 **이진수**/**8진수**/**16진수** 문자열로 변환한다.

```python
print(bin(10))
print(oct(10))
print(hex(10))

# 0b1010
# 0o12
# 0xa
```

### format()

내장 함수 `format()`으로도 10진수를 다른 진수로 변환할 수 있다. 첫번째 인자로는 변환할 10진수를, 그리고 두번째 인자로는 변환할 진법 정보를 전달한다.

위에서 설명했던 함수들과는 다르게 접두사가 붙지 않는다.

```python
print(format(10, 'b'))
print(format(10, 'o'))
print(format(10, 'x'))

# 1010
# 12
# a
```

두번째 인자의 앞에 `#`을 붙이게 되면 접두사를 붙일 수 있다.

```python
print(format(10, '#b'))
print(format(10, '#o'))
print(format(10, '#x'))

# 0b1010
# 0o12
# 0xa
```

## n진수 $\rightarrow$ 10진수

내장 함수 `int()`의 두번째 인자는 `base`로, 생략하면 기본값으로 **10**을 가지며, 변환할 진법 정보를 전달하게 되면 원하는 진수로 변환이 가능하다.

```python
print(int('1001', 2))
print(int('15', 8))
print(int('ab', 16))

# 9
# 13
# 171
```

2진수/8진수/16진수가 대상인 경우, 문자열 앞에 접두사 `"0b"`/`"0o"`/`"0x"`를 붙여 명시할 수 있다.

> 첫번째 인자가 숫자가 아니거나 `base`가 생략되지 않고 주어졌다면 첫번째 인자는 꼭 문자열, bytes, 또는 bytesarray 중 하나여야 한다.
{: .prompt-info }

## 참고문헌

[https://docs.python.org/ko/3/library/functions.html](https://docs.python.org/ko/3/library/functions.html)
