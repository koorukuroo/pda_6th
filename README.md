# 🖥️ AWS EC2 인스턴스 성능 리포트 - 프디아 6기

![image](https://github.com/user-attachments/assets/0c3eefe6-d014-4d14-9644-0496e21b3b3c)

# EC2 Arm 인스턴스 벤치마크 (Graviton 2 vs 3 vs c7g)

| 팀 | 말하는감자(김혜윤,김태헌) |
|----|----------------------|

> 동일 코드·트래픽 조건에서 **m6g.medium → m7g.medium → c7g.medium**  
> 성능 + 비용 효율을 비교해 “가장 가성비 좋은 API 서버 타입”을 찾는다.

---

## 1 . 테스트 환경

| 항목 | 설정 |
|------|------|
| **AMI** | Amazon Linux 2 (Arm64) + Flask 2.3 + Python 3.7 |
| **리전 / AZ** | ap‑northeast‑2a (서울) |
| **부하 발생기** | t3.large (같은 AZ) &nbsp;·&nbsp; JMeter 5.6.3 |
| **부하 시나리오** | 100 Virtual Users × 100 loop = **10 000 req**<br>동시 100 연결·워밍업 없음 |
| **측정 지표** | Avg / p95 Latency, Throughput(RPS), Error %, RPS per USD |

---

## 2 . 대상 인스턴스

| Instance | vCPU | RAM | $/h (Seoul) |
|-----------|-----:|----:|-----------:|
| **m6g.medium** | 1 | 4 GiB | \$0.0385 |
| **m7g.medium** | 1 | 4 GiB | \$0.0347 |
| **c7g.medium** | 1 | 2 GiB | \$0.0315 |

---

## 3 . 결과

| Instance | Avg ms | p95 ms | **RPS** | Err % | **RPS / \$** |
|----------|-------:|-------:|--------:|------:|-------------:|
| m6g.medium | 70 | 170 | 1 240 | 0.00 | 32 208 |
| m7g.medium | 51 | 168 | 1 466 | 0.00 | 42 301 |
| c7g.medium | 52 | 208 | 1 470 | 0.00 | **46 667** |

### 3‑1 Throughput & 가성비

![RPS_vs_RPSUSD](➊charts/rps_combo.png)

### 3‑2 Latency 분포

![Latency_Box](➋charts/latency_box.png)

> ※ 그래프는 `/charts` 폴더의 PNG 이미지를 참조하도록 돼 있습니다.

---

## 4 . 분석

1. **세대 효과**    
   Graviton‑3(m7g) → G2(m6g) 로 교체 시 **Avg –27 % / RPS +18 %**.
2. **컴퓨팅 최적화(c7g)**    
   최대 처리량 1 470 RPS 로 1위, 다만 p95 Tail +23 %(208 ms) → 레이턴시 민감 서비스는 m7g 권장.
3. **가성비**    
   시간당 \$0.0315 인 **c7g.medium** 이 RPS / USD 46 k 로 최상.
4. **x86 (t3.medium) 대비** *(참고)*    
   동일 가격대 Arm 전환 시 RPS ≈ +30 %, 비용 ≈ –25 % 절감 가능.

---

## 5 . 결론 & 권장 사항

| 사용 시나리오 | 추천 타입 | 근거 |
|---------------|-----------|------|
| **기본 API 서버** | **m7g.medium** | 지연·안정·비용 균형 최적 |
| **CPU‑집중 배치** | c7g.medium | 최대 처리량 & 가성비 1위 |
| **레거시 x86 필요** | m6i.large | 이번 실험 범위 밖, 참고용 |
| **비‑핵심·저예산** | m6g.medium | 가장 저렴하지만 지연 ↑ |

---

## 6 . 재현 방법

```bash
# 1. 부하 발생기 설치 (JMeter 5.6.3)
wget https://downloads.apache.org/jmeter/binaries/apache-jmeter-5.6.3.zip && unzip *.zip
export JM=~/apache-jmeter-5.6.3/bin/jmeter

# 2. 테스트 실행 (100 VU × 100 loop)
for host in 15.165.18.211 43.203.125.62 3.38.141.96; do
  $JM -n -t ping-test.jmx -Jhost=$host -Jusers=100 -Jiter=100 \
      -l results-$host.jtl -f -e -o report-$host
done
