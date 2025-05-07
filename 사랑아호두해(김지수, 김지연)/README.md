# 🔍 AWS EC2 인스턴스 성능 비교 실험: 0-1 Knapsack 기반 분석


### 🧠 주제 선택 배경
EC2 인스턴스에 대해 학습하며, 클라우드 인스턴스 선택은 단순한 사양 비교를 넘어 실제 워크로드에 최적화된 성능 검증이 중요하다는 것을 알게 되었다.
그래서 <b>“같은 연산이라도, 어떤 인스턴스가 실제로 가장 빠르고 효율적인가?”</b>에 대한 답을 찾고자 했다.
<br><br>


### 📚 실험 배경 및 목적
이번 실험은 두 가지 아이디어에서 출발했다:

1. **우리가 아는 알고리즘 문제로 실험을 접근하면 창의적일 것이다!**  
   - 메모리와 CPU 모두를 많이 사용하는 DP 기반 문제로, 입력 크기에 따라 연산 시간이 급격히 증가하는 **0-1 Knapsack 문제**를 선택

2. **인스턴스 종류에 따라 성능 차이가 얼마나 발생할까?**  
   - Instance family 관점: `m6i.large`, `c6i.large`, `t3.large`  
   - Processor architecture 관점: `m6i.large`(Intel) vs `m6g.large`(Graviton2)

<h4> ✅ 목표 : 단순 스펙 비교가 아닌 실제 워크로드 기반의 실성능 평가 지표 제공 </h4>
<br>


### 🧪 실험 개요
- **문제 유형**: 0-1 Knapsack (DP 기반)
- **입력 조건**:
  - 아이템 수: 2,000개
  - 배낭 용량: 100,000
  - 반복 실행: 5회
- **측정 항목**:
  - 평균 계산 시간 (초)
  - 최대 메모리 사용량 (MB)
<br><br><br>


### ⚙️ 실험 방법

- Python 3.11 (Anaconda)
- `time`, `memory_profiler` 라이브러리를 활용해 성능 측정
- 멀티스레드 조건별(1, 2, 4, 8스레드)로 실행 시간 비교
<br><br><br>

-----
### 💻 사용한 코드
### 첫 번째 실험(knapsack_test.py) : Knapsack 알고리즘
```python
# 실험 실행
def run_knapsack_experiment(n_items, trials):
    times = []
    mem_usages = []

    for _ in range(trials):
        weights, values, capacity = generate_knapsack_data(n_items)
        start = time.time()
        result = knapsack(weights, values, capacity)
        end = time.time()
        times.append(end - start)
        mem_usages.append(get_memory_usage_mb())

    avg_time = sum(times) / trials
    max_mem = max(mem_usages)

    print(f"[✓] 실험 정보: 아이템 개수 = {n_items}, 배낭 용량 = {capacity}")
    print(f"    ▶ 평균 수행시간: {avg_time:.4f}초")
    print(f"    ▶ 최대 메모리 사용량: {max_mem:.2f}MB")

# 메인
if __name__ == "__main__":
    run_knapsack_experiment(n_items=2000, trials=5)
```
<br><br>

### 📈 실험 결과 예측
| 인스턴스     | 예상 결과 요약                                          |
|:---:|----------------------------------------------------------|
| `c6i.large`  | 고클럭 기반, 가장 빠른 처리 시간 예상                    |
| `m6i.large`  | 안정적 처리, 속도는 `c6i`보다 느리나 메모리 효율 우수   |
| `t3.large`   | 버스트형이므로 성능 예측이 어려움, 연속적 고부하 작업에서 가장 낮은 성능 예상 |
| `m6g.large`  | ARM 기반이지만 효율적, `t3`보다 우수하고 `m6i` 근접 예상 |

<br><br>

### 📊 실험 결과 그래프 : Knapsack 알고리즘 실행 시간 비교
<img src="https://github.com/user-attachments/assets/14abf05c-aba1-47fc-920c-e5d073f413ed" width=600><br>

| 인스턴스    | 평균 수행 시간 (초) | 최대 메모리 사용량 (MB) | 
|:-------------:|:---------------------:|:--------------------------:|
| m6i.large   | 18.3768             | 18.33                    | 
| c6i.large   | 18.4469             | 18.12                    | 
| t3.large    | 24.1489             | 18.25                    | 
| m6g.large   | 28.7074             | 17.27                    |

<br><br>


### 📝 실험 결과 요약
❗ **예상과 다른 결과 발생!** <br>
- `c6i.large`가 더 성능이 좋을 것으로 예상했으나, 실제로는 `m6i.large`가 **더 나은 평균 성능**을 보임<br>

🤔 **왜 이런 결과가 나왔을까?**
- `0-1 Knapsack` 문제는 **단일 스레드 기반 연산**이므로, CPU의 클럭 속도보다 **메모리 대역폭, 캐시 구조** 등이 더 큰 영향을 줄 수 있음
- `c6i`와 `m6i`의 구조적 차이는 **단일 스레드 환경에서 명확히 드러나지 않음**<br>

➕ **그래서 어떻게 했나?**
- `c6i.large`와 `m6i.large`의 차이를 **명확하게 드러내려면 멀티스레드 환경에서 CPU 병렬 처리 능력을 요구하는 문제**를 선택해야 함!
- CPU 병렬 처리 능력을 요구하는 **행렬곱 실험**을 추가로 진행
<br><br><br>

-----
### 💻 사용한 코드
### 두 번째 실험(mtx_thread_test.py) : 행렬곱 알고리즘

```python
import numpy as np
import threading
import time

# 행렬 곱셈 함수
def multiply_part(A, B, C, start_row, end_row):
    for i in range(start_row, end_row):
        C[i, :] = np.dot(A[i, :], B)

def threaded_matrix_multiplication(n=2048, num_threads=4):
    A = np.random.randint(0, 100, size=(n, n))
    B = np.random.randint(0, 100, size=(n, n))
    C = np.zeros((n, n), dtype=int)

    threads = []
    rows_per_thread = n // num_threads

    start = time.time()

    for i in range(num_threads):
        start_row = i * rows_per_thread
        end_row = (i + 1) * rows_per_thread if i < num_threads - 1 else n
        t = threading.Thread(target=multiply_part, args=(A, B, C, start_row, end_row))
        threads.append(t)
        t.start()

    for t in threads:
        t.join()

    end = time.time()
    print(f"[✓] 행렬 크기: {n}x{n}, 스레드 수: {num_threads}, 수행 시간: {end - start:.4f}초")

# 실행 예시
if __name__ == "__main__":
    for threads in [1, 2, 4, 8]:
        threaded_matrix_multiplication(n=2048, num_threads=threads)
```
<br><br>


### 📊 실험 결과 그래프 : 멀티 스레드 환경에서의 실행 시간 비교
<img src="https://github.com/user-attachments/assets/41df5f68-e518-4ea8-9c3b-77eb44e9e045" width=600><br>

| 인스턴스 | vCPU | 아키텍처 | 스레드 1 | 스레드 2 | 스레드 4 | 스레드 8 | 비고                                       |
|:-------------:|:------:|:------------------:|:----------:|:----------:|:----------:|:----------:|:--------------------------------------------:|
| m6i.large   | 2    | x86 (Intel)      | 50.49초  | 20.54초  | 39.71초  | 25.18초  | 성능 일관성 낮음 (4스레드↓)               |
| c6i.large   | 2    | x86 (Intel)      | 50.89초  | 17.83초  | 38.02초  | 26.24초  | m6i 대비 2스레드 성능 우위                 |
| t3.large    | 2    | x86 (Intel)     | 66.58초  | 24.15초  | 21.57초  | 21.84초  | 4~8스레드 시 오히려 향상                   |
| m6g.large   | 2    | ARM (Graviton2)  | 51.21초  | 32.16초  | 26.37초  | 27.28초  | ARM 기반, 멀티스레드 안정적               |

❗ **주요 관찰 포인트**
- 2스레드까지는 성능이 개선되는 모습을 보였지만, 4스레드와 8스레드에서는 오히려 성능이 떨어짐
- 이는 인스턴스의 vCPU 수를 초과한 스레드를 사용할 경우 오버헤드가 발생하기 때문

💬 **요점 정리**
- 4스레드와 8스레드 실험은 과도한 병렬 처리로 인한 오버헤드를 시각적으로 입증함
- 따라서, 인스턴스 선택 시 **코어 수 대비 병렬 처리 한계**를 고려해야 함
<br><br><br>


### 📊 실험 결과 : 스레드 수에 따른 성능 변화 분석
### 📌 m6i.large
- 2스레드까지는 좋은 성능 향상이 있음 (**2배 이상** 성능 개선)
- 하지만 4스레드에서 **성능이 급락** (오히려 더 느림)
- 8스레드는 일부 회복되지만 2스레드보다 나쁨 → 스레드 수가 CPU 코어 수를 넘어서면 오버헤드 발생

### 📌 c6i.large
- 전반적으로 m6i와 유사한 패턴.
- **2스레드에서 가장 뛰어난 성능(17.8초)**
- 4스레드와 8스레드는 성능 저하 → 역시 물리 코어 2개에서 더 이상 병렬화가 의미 없음

### 📌 t3.large
- 성능 예측이 어려운 **버스트형 인스턴스**
- 기본 성능은 떨어지는 결과를 보임 (1스레드 66.6초)
- 4, 8스레드에서 놀라운 성능 향상(21초대)을 보였는데, 일반적으로는 예외적인 결과로 신뢰도는 낮음
- 이는 버스트 크레딧 활용 중일 가능성 있음

### 📌 m6g.large (Graviton2, ARM)
- 1, 2, 4, 8스레드까지 **성능 변화가 점진적이며 일관됨**
- 스레드 수가 늘어날수록 점진적인 성능 향상이 있지만, 8스레드에서 소폭 악화
- **다중 스레드 병렬 처리에 강한 안정성** → Graviton2의 특성

→ 결론적으로는 .large의 경우에는 vCPU가 2개여서 스레드가 2개인 경우까지만 유의미한 성능 향상을 관찰할 수 있었음
<br><br><br>

-----
### 💡 실험을 통해 얻은 인사이트
- `c6i.large` 가 항상 우세할 거라고 예상했던 것과 달리 항상 더 빠르지는 않았음, 멀티스레드 환경에서는 우위였으나 범용적인 성능에서는 m6i가 더 안정적이었음
- `t3.large`는 크레딧 기반이므로 **지속적 연산에는 부적합**하지만, 간헐적 연산에는 비용 효율적인 대안이 될 수 있음
- `m6g.large`는 ARM 기반이지만 **일정한 병렬 성능**을 보여주며, 에너지 효율 측면에서도 충분히 고려할 수 있는 선택지

<br><br>


### 🧾 결론

실험을 통해, 어떤 인스턴스가 더 효율적인지는 **사용 목적과 연산 특성에 따라 달라진다**는 점을 확인할 수 있었다.

따라서, 단순히 vCPU, RAM, 비용만 보고 인스턴스를 선택하기보다는,  
**해당 인스턴스가 어떤 워크로드에 최적화되어 있는지를 실험적으로 확인**하는 것이 필요하다.

#### 👥 제안
> 머신러닝 학습, 영상 처리 등 고부하 병렬 연산에는 `c6i` 계열이 적합  
> 웹 서버, API 처리처럼 일관성과 안정성이 중요한 워크로드는 `m6i` 계열이 더 효율적일 수 있음  
> 비용 효율 중심이라면 `t3.large`는 간헐적 트래픽 대응에 유리  
> ARM 기반인 `m6g`는 코드 호환성이 확보된 경우 비용 대비 성능이 우수함

<br>
✅ 실제 성능 데이터를 바탕으로, <b>자신의 목적에 가장 적합한 인스턴스를 선택</b>하는 것이 가장 중요하다.
