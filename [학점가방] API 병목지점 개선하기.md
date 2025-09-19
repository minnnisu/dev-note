# [학점가방] API 병목지점 개선하기

저희 [학점가방](https://github.com/Dongguk-unibag) 프로젝트를 배포 했을 당시 많은 유저들이 서비스의 기능을 이용해줬고 그 중에서 과제 조회를 위한 API를 주로 호출한다는 사실을 로그를 통해 확인했습니다.
이런 상황에서 과제 조회 API가 트래픽 부하가 발생한 경우 어떤 지점에서 병목이 발생하고, 개선할 수 있을지 고민해보고 싶어 부하테스트를 진행하였습니다.


> **실험 결과 요약**

## 실험 환경
부하 테스트를 위해 다음과 같은 환경에서 테스트를 진행하였습니다.

| <img width="500" height="9891" alt="성능 측정 환경" src="https://github.com/user-attachments/assets/4d26e226-5987-4d2b-ac8c-c43a3aae76b3" /> |
|:---:|
| 성능 측정을 위한 아키텍쳐 |


### 서버 환경
- 하드웨어: Odroid (4 Core CPU, 8GB RAM)
- 운영체제: Ubuntu 20.04.6 LTS
- 웹 서버: Nginx 1.27.0
- 웹 애플리케이션: Spring Boot 3.3.5 (Java 17)
- 데이터베이스:
  - MySQL 8.2
  - Redis 7.2.5
- 모니터링:
  - Prometheus
  - Grafana
  - Loki

### 테스트 환경
- 부하 발생 도구: k6
- 테스트 클라이언트: MacBook Pro M3

## 1차 성능 테스트
앞서 설정한 환경을 기반으로 기존에 작성했던 API에 부하테스트를 진행하였습니다.

부하테스트 진행에 앞서 DB에는 약 5000명의 유저 정보와 과제 5만건을 미리 저장해두었습니다.

| <img width="280" alt="스크린샷 2025-09-19 오후 12 31 36" src="https://github.com/user-attachments/assets/2cf9abba-2794-4a74-beb7-dea938d7a30f" /> | <img width="280" alt="스크린샷 2025-09-19 오후 12 31 24" src="https://github.com/user-attachments/assets/048e54c8-654e-44a3-8ea2-b3aa77680221" /> |
|:---:|:---:|
| 과제 테이블 Inspector | 유저 테이블 Inspector |

### 테스트 시나리오
최대 500명의 동시 사용자가 Access Token을 이용해 `/api/assigment` API를 호출하는 상황을 점진적으로 늘리고 줄이면서 테스트합니다. 

또한 각 사용자는 1초 간격으로 요청을 보내고, 응답이 정상적으로 200 OK 인지 확인합니다.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { SharedArray } from 'k6/data';

// CSV 파일 불러오기
const tokens = new SharedArray('tokens', function () {
  // 각 줄을 배열로 반환
  return open('./access_tokens.csv')
    .split('\n')
    .slice(1) // 첫 줄(header) 제거
    .map(line => line.trim())
    .filter(line => line.length > 0);
});

export let options = {
  scenarios: {
    ramping_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 20 },  // 2분 동안 20명까지 증가
        { duration: '2m', target: 100 },  
        { duration: '2m', target: 300 },  
        { duration: '2m', target: 500 },  
        { duration: '2m', target: 300 },  
        { duration: '1m', target: 100 },  // 점진적으로 종료
      ],
    },
  }
};

export default function () {
  // __VU = 현재 실행 중인 가상 유저 번호 (1부터 시작)
  let tokenIndex = (__VU - 1) % tokens.length; // 2000명 이상일 경우 안전하게 나눠 사용
  let accessToken = tokens[tokenIndex];

  let res = http.get('https://unibag.kro.kr/api/assigment', {
    headers: { Authorization: `Bearer ${accessToken}` },
  });

  check(res, {
    'status was 200': (r) => r.status === 200,
  });

  sleep(1);
}
```

### 테스트 결과
| <img width="1000" alt="Frame 197" src="https://github.com/user-attachments/assets/fa997bf6-6f81-4470-8caf-ae8539b52aed" /> | <img width="876" alt="image" src="https://github.com/user-attachments/assets/85572357-6bc7-4504-a752-11f66e2240da" /> |
|:---:|:---:|
| Summary | VUs & Throughput |

K6 Report를 통해 다양한 성능 지표를 확인할 수 있었으며 그 중에서 저는 **지연시간(Latency), 처리량(throughput), 실패율** 3가지 지표에 집중했습니다.

지연시간은 95% 지점에서 13s로 상당한 느린 결과를 확인할 수 있었습니다.

처리량은 30.85/s으로 나타났으며 다른 지표와 함께 비교를 해본 결과 유저 수가 약 50명이 되는 지점부터 병목이 발생하는 것을 확인할 수 있었습니다.

실패유를 0%로 모든 요청이 성공적으로 응답된 것을 확인할 수 있었습니다.

## 문제 원인

## 실험결과 첨부파일
> 1차 테스트 Report

[k6 report(과제 조회 1차 테스트).html](https://github.com/user-attachments/files/22418809/k6.report.1.html)
