---
title: 파이썬에서 수행 시간을 측정하는 방법
date: 2022-11-19 12:45:00 +0900
categories: [Algorithm]
tags: [time]
---

알고리즘 풀이 중에는 **수행 시간**을 측정하는 일을 종종 하게된다.  

파이썬에서는 **time** 모듈을 이용하면 그것이 가능하다.

```python
import time
start_time = time.time()

end_time = time.time()
print("Elapsed time : ", end_time - start_time)
```

본 프로그램의 수행 전과 후에 *time.time()* 을 호출하고 연산을 통해 수행 시간을 얻을 수 있다.
