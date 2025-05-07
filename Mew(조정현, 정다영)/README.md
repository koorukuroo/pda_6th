# EC2 인스턴스 성능 비교 리포트

## 목적

---

이 성능 테스트의 목적은 다양한 인스턴스의 성능을 분석하고, 비용 대비 효율적인 인스턴스를 평가하는 것이다.

실험에서는 CPU와 메모리 성능 평가뿐만 아니라 100만 개의 데이터를 삽입하고, 단일 스레드 및 다수 스레드를 활용한 부하 테스트를 통해 데이터베이스 성능 또한 평가하였다.

성능 분석은 다음 세 가지 주요 기준을 바탕으로 진행하였다:

1. micro ~ xlarge 따른 인스턴스 성능 차이
2. 인스턴스 패밀리 간(t, m, r) 성능 차이
    1. m과 r의 경우 AWS RDS에서 기본적으로 사용하는 패밀리
3. 운영체제(Ubuntu, AWS Linux 2)에 따른 성능 차이

## 사용한 인스턴스 목록

---

### ubuntu

- t4g.micro
- t4g.small
- t4g.medium
- t4g.large
- t4g.xlarge

### AWS Linux 2

>스토리지 gp3 통일

- t4g.micro
- t4g.small
- t4g.medium
- t4g.large
- t4g.xlarge
- m6g.large
- m6g.xlarge
- r7g.large
- r7g.xlarge

## 테스트 사전 설치

---

- sysbench
    - 설치 방법: [https://velog.io/@cisxo/Mariadb-성능-테스트-with-sysbench-cisxo](https://velog.io/@cisxo/Mariadb-%EC%84%B1%EB%8A%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8-with-sysbench-cisxo)
- mariadb
    - 10.5.25-MariaDB

## 테스트 방법

---

### 인스턴스 정량 측정

cpu 성능 측정

```jsx
sysbench cpu --cpu-max-prime=10000 run
```

memory  성능 측정

```jsx
sysbench memory run
```

디스크 쓰기 측정 (1GB 쓰기 테스트)

```jsx
sysbench fileio --file-total-size=1G prepare
```

### 테스트 데이터 베이스 생성

```jsx
CREATE DATABASE sbtest;
```

### sysbench를 통한 mariadb database 테스트
| 옵션 | 값 | 설명 |
| --- | --- | --- |
| `sysbench` | - | 성능 테스트를 실행하는 명령어 도구. CPU, 메모리, 디스크, 데이터베이스 등의 성능을 측정할 수 있음. |
| `--db-driver` | `mysql` | 사용할 데이터베이스 드라이버 지정. 여기서는 MySQL을 대상으로 테스트함. |
| `--mysql-host` | `localhost` | MySQL이 실행 중인 호스트 주소. 로컬 머신에서 실행 중이므로 `localhost`로 설정됨. |
| `--mysql-user` | `root` | DB에 접근할 사용자 계정. `root`는 MySQL의 기본 관리자 계정. |
| `--mysql-password` | `1234` | DB 접속을 위한 비밀번호. 테스트 목적이므로 단순한 비밀번호 사용. |
| `--mysql-db` | `sbtest` | 벤치마크용 테스트 대상 데이터베이스 이름. 사전에 생성되어 있어야 함. |
| `--table-size` | 10000 | 테스트용 테이블에 삽입될 행(row)의 개수. 총 100만 건의 데이터를 대상으로 부하 테스트 수행. |
| `--threads` | 100 | 동시 실행할 클라이언트(스레드) 수. 다중 사용자 시나리오를 시뮬레이션하기 위해 10개의 스레드로 병렬 처리함. |
| `oltp_read_write` | - | 테스트 타입 지정. `OLTP(Online Transaction Processing)` 환경에서 읽기 + 쓰기 복합 부하 시나리오를 테스트함. |
| `run` | - | 테스트를 실행하는 명령. 위 설정으로 실제 부하를 수행함. |




테스트 데이터 준비 (100만개 데이터 삽입)

```jsx
sysbench --db-driver=mysql --mysql-host=localhost --mysql-user=root --mysql-password=1234 --mysql-db=sbtest --table-size=1000000 oltp_read_write prepare

```

스레드 100개

```jsx
sysbench --db-driver=mysql --mysql-host=localhost --mysql-user=root --mysql-password=1234 --mysql-db=sbtest --table-size=10000 --threads=100 oltp_read_write run
```

# 테스트 데이터

---

## Ubuntu (arm)

| 추출 목록 | cpu 처리량(events/sec) | memory | memory 쓰기속도 (1gb) | Transactions (초당 처리된 트랜잭션 수), TPS | Queries/sec (쿼리 초당 처리 속도) QPS | Latency(min) | Latency(max) | latency(95%) | 시간당 비용 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| t4g.micro | 2804.27 | 3967891.95 per second | 7.23 | 514.1 | 11648.13 per sec. | 4.06 | 1309.48 |  502.20 | 0.0104 |
| t4g.small | 2805.50 | 3966219.63 per second | 7.22 | 543.9 | 12338.23 per sec. | 4.38 | 1216.01 | 475.79 | 0.0208 |
| t4g.medium | 2711.99 | 3768032.35 | 7.22 | 555.1 | 12494.18 per sec. | 5.98 | 1375.99 | 467.30 | 0.0416 |
| t4g.large | 2788.82 | 3975151.36 per second | 7.22 | 540.8 | 12254.82 per sec | 4.15 | 1346.43 | 475.79 | 0.0832 |
| t4g.xlarge | 2611.86 | 3960290.94 per second | 7.22 | 1051.8 | 24076.12 per sec. | 3.45 | 630.15 | 248.83 | 0.1664 |

## AWS Linux 2 (arm)

| 추출 목록 | cpu 처리량 | memory | momory 쓰기속도 (1gb) | Transactions (초당 처리된 트랜잭션 수), TPS | Queries/sec (쿼리 초당 처리 속도) QPS | Latency(min) | Latency(max) | latency(95%) | 비용 시간당 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| t4g.micro | 2961.65(events/s) | 4239412.3777 MiB/s | 7.20 | 857.6(10s)  | 171520 | 1.89ms | 680.39ms | 267.41ms | 0.0104USD |
| t4g.small | 2961.83 | 4398998.5224 | 7.20 | 895.1 | 179020 | 1.99 | 620.96 | 253.35 | 0.0208USD |
| t4g.medium | 2954.71 | 4295833.2110 | 7.20 | 952.0 | 190400 | 2.29 | 605.63 | 240.02 | 0.0416USD |
| t4g.large | 2959.85 | 4367606.2917 | 7.20 | 922.5 | 184516 | 2.18 | 555.66 | 235.74 | 0.0832USD |
| t4g.xlarge | 2921.46 | 4353271.4946 | 7.20 | 1671.2 | 334240 | 2.30 | 382.89 | 139.85 | 0.1664USD |
| m6g.large | 2961.45 | 4396761.4023 | 7.19 | 976.0 | 195200 | 1.94 | 667.05 | 240.02 | 0.094USD |
| m6g.xlarge | 2962.07 |  4414761.4830 | 7.20 | 1895.2 | 379040 | 1.76 | 402.22 | 125.52 | 0.188USD |
| r7g.large | 3178.87 | 5006897.4311 | 7.20 | 1223.8 | 244760 | 1.51 | 733.64 | 189.93 | 0.1292USD |
| r7g.xlarge | 3182.19 | 5029881.8079 | 7.20 | 2280.5 | 456100 | 1.67 | 508.47 | 108.68 | 0.2584USD |

# 테스트 결과

## 1. micro ~ xlarge에서의 성능 차이와 비용 대비 효율

![Image](https://github.com/user-attachments/assets/68cc3c5b-a30d-47fa-b925-3fde001cfac3)

- CPU 처리량, 메모리 처리량, 메모리 쓰기 속도는 t4g.micro ~ t4g.xlarge까지 큰 차이를 보이지 않음.
- TPS와 QPS는 t4g.large에서 t4g.xlarge로 갈 때 급격히 늘어나고, Latency 또한 t4g.large에서 t4g.xlarge로 갈 때 급격히 줄어들었음. t4g.micro ~ t4g.large까지는 변화폭이 크지 않음.

**→ 리소스량이 대략 2배씩 늘어나는 건 똑같은데, 왜 t4g.large와 t4g.xlarge의 성능차이가 확연하게 드러날까?**

t4g.large는 2개의 vCPU를, t4g.xlarge는 4개의 vCPU를 사용한다. 이 시점에서 우연히도 시스템 성능을 제한하던 병목 현상(CPU 사용량이 100%이고, 작업 대기 큐가 쌓여 있는 상태)이 해소되면서 성능이 급격하게 높아졌을 확률이 높다.

---


![Image](https://github.com/user-attachments/assets/8d7a402f-e4e6-4027-9a58-f2ed064d0fdb)

- 비용 대비 성능 차이를 비교분석해봤을 때, t4g.micro가 가장 높은 가성비를 보임.
- 비용은 2배씩 차이나지만, 성능 개선폭은 그에 비례하지 않기 때문임.
- 가성비를 따진다면 t4g.micro가 가장 좋은 선택지가 될 것이나, t4g.large → t4g.xlarge 구간에 한해서는 2배 가격 차이만큼의 급격한 성능 개선폭을 보이므로 고성능이 요구되는 환경에서는 추기 비용 대비 효과적인 투자가 될 수 있음.

## 2. 인스턴스 패밀리끼리의 성능 차이와 비용 대비 효율

---


![Image](https://github.com/user-attachments/assets/217ba23d-9dce-4b45-a056-9c8c78e87c1c)

- CPU 처리량과 메모리 처리량 부분에서는 r7g 시리즈들이 성능이 제일 좋았음.
- 메모리 쓰기 속도는 t4g, m6g, r7g 셋 다 큰 차이가 없음.
- TPS와 QPS 부분에서는 r7g, m6g, t4g 순으로 성능이 좋음.
- Latency 부분에서는 t4g, m6g, r7g 순으로 성능이 좋음.

---


![Image](https://github.com/user-attachments/assets/ce777477-efeb-4d33-a6eb-9cdaebac0947)

- 대체적으로 성능은 r7g, m6g, t4g 순으로 좋지만, 가성비는 그와 반대되는 경향성을 보임.

## 3. 운영체제에 따른 성능 차이

---


![Image](https://github.com/user-attachments/assets/e8f6a189-93a2-4d4f-9cf5-5c5e31cd008b)

- CPU 속도 : Amazon Linux 2가 미세하게 초당 이벤트 처리 수가 많음.
- 메모리 처리 속도 : Amazon Linux 2가 미세하게 메모리 처리 속도가 빠름.
- 메모리 쓰기 속도 : Amazon Linux 2가 아주 미세하게 1GB를 쓰는 데 시간이 적게 걸림(0.02초 정도)
- 트랜잭션 처리량 : Amazon Linux 2가 더 많음.
- QPS : Amazon Linux 2가 눈에 띄게 많음.
- Latency(95%) : Amazon Linux 2가 낮음.

→ **왜 모든 지표에서 Amazon Linux 2가 Ubuntu보다 성능이 우수할까?**

Amazon Linux 2는 AWS 전용 OS로 AWS 환경에서의 성능을 극대화하도록 설계된 OS이기 때문에 AWS에 한해서 전반적인 성능이 다른 OS보다 우수할 수 있음.

가격은 Ubuntu와 Amazon Linux 2가 동일하기 때문에, AWS 환경에서 실행할 것이라면 Amazon Linux 2를 선택하는 것이 유리함.

## 그래서 어떤 모델을 선정해야 하는가?

---

<img width="701" alt="Image" src="https://github.com/user-attachments/assets/3eacc637-0302-460b-9d8f-a467ffd1f2da" />

- t4g.micro와 t4g.medium이 성능 대비 가격이 매우 낮음(2배씩 증가)
- 또한 AWS RDS에서 사용하는 m모델과 r모델


<img width="665" alt="Image" src="https://github.com/user-attachments/assets/638bd9f4-a278-4f8a-81fa-48c10656c809" />


- 이 지표를 종합적으로 고려했을 때, 가격 대비 성능 측면에서 t4g.xlarge와 m6g.xlarge 모델이 효율적이라 볼 수 있습니다.
- 인스턴스 모델을 선정할 때는 시스템 환경과 요구 사항을 충분히 고려한 후, 실제로 우리가 측정한 테스트 데이터를 바탕으로 sysbench와 같은 도구를 이용해 성능을 평가하고 비교하는 것이 가장 바람직합니다. 따라서, 이미 실험을 통해 얻은 데이터를 참고하여 최적의 모델을 선택하는 것이 성능과 효율성에 도움이 될 것입니다.
