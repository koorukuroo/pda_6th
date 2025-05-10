# 📈 IPO 상황을 가정한 최적의 인스턴스 활용 전략

<br>

![ezgif-7e9fbf1f89e985](https://github.com/user-attachments/assets/bf9bc2bb-ab56-4f45-a837-02585eafcd93)

<br>


## 🧩 목적

실제 **트래픽 폭주**가 발생할 수 있는 IPO(기업공개) 상황을 가정하여,

**Auto Scaling 기반 서버 증설 전략**의 효율성과 비용 효과를 실험함

> 목표: 인스턴스 활용 방식의 성능 및 비용 효율성을 비교해 최적 전략을 도출하는 것
> 
> - ✅ **소형 인스턴스를 다수 확장 (수평 확장)**
> - ✅ **대형 인스턴스를 적게 활용 (수직 확장)**

---

## 📦 실험 대상 인스턴스

| 인스턴스 유형 | vCPU | RAM | 시간당 요금 (USD) |
| --- | --- | --- | --- |
| `t4g.small` | 2 | 2 GB | $0.0208 |
| `t4g.medium` | 2 | 4 GB | $0.0416 |

---

## 🛠 실험 구성

- **테스트 도구**: [k6](https://k6.io/) (부하 테스트 오픈소스 툴)
- **시나리오 조건**:
    - 동시 접속자 수(VUs): 1500
    - 요청 수: 약 22,653건
    - 테스트 시간: 2분
- **측정 지표**:
    - 평균 응답 시간 (`http_req_duration`)
    - 오류율 (`checks_failed`)
    - 처리량 (`http_reqs`)
    - 네트워크 사용량 (`data_received`, `data_sent`)
    - 비용 대비 효율성

---

# AWS EC2 인스턴스 성능 비교 (K6 부하 테스트)

`k6`를 사용해 세 가지 EC2 인스턴스 타입의 웹 서버 성능 실험을 통한 최저비용의 효율적 인스턴스를 찾음

테스트 대상 인스턴스:

- `t3.small`
- `t4g.small`
- `t3a.small`

---

## 📊 테스트 요약

| 항목 | t3.small | t4g.small | t3a.small |
| --- | --- | --- | --- |
| 평균 응답 시간 (ms) | 887.77 | **518.18** | 919.01 |
| 초당 요청 처리수 | 276.116/s | **326.504/s** | 257.768/s |
| 실행 시간 | 2분 | 2분 | 2분 |
| 시간당 비용 ($) | 0.0208 | **0.0168** | 0.0188 |
| 요청당 비용 ($) | 6.28e-7 | **4.24e-7** | 5.98e-7 |

✅ 모든 인스턴스에서 HTTP 응답 상태는 100% 성공 (`status is 200`)

### 테스트코드
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 1300,              // 동시 사용자 수
  duration: '1m',      // 테스트 지속 시간
};

export default function () {
  const res = http.get('http://<IP>:3000');

  check(res, {
    'status is 200': (r) => r.status === 200,
  });

  sleep(1);  // 다음 요청까지 대기 시간 (1초)
}
```
---

## 📈 시각화 그래프

![image.png](https://isyoudwn.notion.site/image/attachment%3A07db7e6b-9923-4804-9a4d-69a8fc46159c%3Aimage.png?table=block&id=1ec63371-3678-8054-b40a-fe93752b125f&spaceId=ea7f5970-fe44-4506-aa2b-500766ca5b0d&width=1420&userId=&cache=v2)

---

## ✅ 결론

- **t4g.small** 인스턴스는 모든 면에서 가장 뛰어난 성능을 보였으며, 비용 효율성도 가장 우수
- ARM 아키텍처 기반의 Graviton 인스턴스인 `t4g.small`은 웹 서버 부하 처리에 가장 적합한 선택

---

## 🛠 테스트 환경

- 툴: [k6](https://k6.io/)
- 사용자 수: 500 VUs
- 시나리오: 2분 동안 반복 요청
- 대상: Node.js 기반 웹 API 서버

t4g-small

![image.png](https://isyoudwn.notion.site/image/attachment%3A6b151a3a-a3aa-4415-b549-49be49603233%3Aimage.png?table=block&id=1ec63371-3678-8095-b496-ecce45b87f0e&spaceId=ea7f5970-fe44-4506-aa2b-500766ca5b0d&width=1420&userId=&cache=v2)

t3a-small

![image.png](https://isyoudwn.notion.site/image/attachment%3A90e9cad1-452b-4028-a780-aef7b7d50804%3Aimage.png?table=block&id=1ec63371-3678-80d0-bc76-d34ba06042b5&spaceId=ea7f5970-fe44-4506-aa2b-500766ca5b0d&width=1420&userId=&cache=v2)

t3.small

![image.png](https://isyoudwn.notion.site/image/attachment%3Ae90b058d-add0-41da-be64-c74707221fb1%3Aimage.png?table=block&id=1ec63371-3678-80c4-a3aa-cba0a3ba6d4c&spaceId=ea7f5970-fe44-4506-aa2b-500766ca5b0d&width=1420&userId=&cache=v2)
