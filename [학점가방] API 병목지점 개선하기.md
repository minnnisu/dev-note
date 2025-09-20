# [학점가방] API 병목지점 개선하기

저희 [학점가방](https://github.com/Dongguk-unibag) 프로젝트를 배포 했을 당시 많은 유저들이 서비스의 기능을 이용해줬고 그 중에서 과제 조회를 위한 API를 주로 호출한다는 사실을 로그를 통해 확인했습니다.
이런 상황에서 과제 조회 API가 트래픽 부하가 발생한 경우 어떤 지점에서 병목이 발생하고, 개선할 수 있을지 고민해보고 싶어 부하테스트를 진행하였습니다.


> **실험 결과 요약**
> 
> 과제 조회 API 부하 테스트 결과, 병목의 원인은 `Assignment` 엔티티 조회 시 발생한 N+1 문제였습니다.  
JPA fetch join 적용으로 단일 쿼리 조회가 가능해지면서 95% 지연시간이 13s → 7s로 약 38% 개선되었고, 처리량도 향상되었습니다.  
그러나 여전히 7s의 지연시간은 사용자 이탈이 발생할 수 있는 수치이며 이를 더 낮추기 위해서는 Scale-Up 혹은 Scale-Out과 같은 서버 확장이 필요하다고 생각합니다.

<br>

## 1. 성능 측정 환경
<img width="500" height="9891" alt="성능 측정 환경" src="https://github.com/user-attachments/assets/4d26e226-5987-4d2b-ac8c-c43a3aae76b3" />

<br>
<br>

| 구분 | 세부 내용 |
|------|-----------|
| 하드웨어 | Odroid (4 Core CPU, 8GB RAM) |
| 운영체제 | Ubuntu 20.04.6 LTS |
| 웹 서버 | Nginx 1.27.0 |
| 웹 애플리케이션 | Spring Boot 3.3.5 (Java 17) |
| 데이터베이스 | MySQL 8.2, Redis 7.2.5 |
| 모니터링 | Prometheus, Grafana, Loki |
| 부하 발생 도구 | k6 |
| 테스트 클라이언트 | MacBook Pro M3 |

<br>

## 2. 부하 테스트

### 2-1. 테스트 시나리오

부하테스트 진행에 앞서 DB에는 약 5000명의 유저 정보와 과제 5만건을 미리 저장해두었습니다.

| <img width="280" alt="스크린샷 2025-09-19 오후 12 31 36" src="https://github.com/user-attachments/assets/2cf9abba-2794-4a74-beb7-dea938d7a30f" /> | <img width="280" alt="스크린샷 2025-09-19 오후 12 31 24" src="https://github.com/user-attachments/assets/048e54c8-654e-44a3-8ea2-b3aa77680221" /> |
|:---:|:---:|
| 과제 테이블 Inspector | 유저 테이블 Inspector |

부하테스트는 최대 500명의 동시 사용자가 Access Token을 이용해 `/api/assigment` API를 호출하는 상황을 점진적으로 늘리고 줄이면서 테스트합니다. 

또한 각 사용자는 1초 간격으로 요청을 보내고, 응답이 정상적으로 `200 OK` 인지 확인합니다.

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

### 2-2. 테스트 결과
| <img width="1000" alt="Frame 197" src="https://github.com/user-attachments/assets/fa997bf6-6f81-4470-8caf-ae8539b52aed" /> | <img width="876" alt="image" src="https://github.com/user-attachments/assets/85572357-6bc7-4504-a752-11f66e2240da" /> |
|:---:|:---:|
| Summary | VUs & Throughput |

K6 Report를 통해 다양한 성능 지표를 확인할 수 있었으며 그 중에서 저는 **지연시간(Latency), 처리량(throughput), 실패율** 3가지 지표에 집중했습니다.

지연시간은 95% 지점에서 13s로 상당한 느린 결과를 확인할 수 있었습니다.

처리량은 30.85/s으로 나타났으며 다른 지표와 함께 비교를 해본 결과 유저 수가 약 50명이 되는 지점부터 병목이 발생하는 것을 확인할 수 있었습니다.

실패율은 0%로 모든 요청이 성공적으로 응답된 것을 확인할 수 있었습니다.

<br>

## 3. 문제 원인
### 3-1. CPU 분석
| <img width="1476" height="891" alt="image" src="https://github.com/user-attachments/assets/8cb73ae4-24a3-4ab2-a467-2e0bf7e7bdc7" /> | <img width="1482" height="761" alt="image2" src="https://github.com/user-attachments/assets/31f5ea9b-c729-415c-b6ad-2f4535e2a24f" /> |
|:---:|:---:|
| Total & Java 사용량 | 컨테이너 별 사용량 |

부하테스트 동안 전체 CPU 사용률이 100%에 근접했으며, 그중 Java 애플리케이션이 약 40~50%를 차지했습니다.

컨테이너 단위로 확인한 결과, Spring Boot 애플리케이션(blue-uni-bag) 다음으로 MySQL의 점유율이 높게 나타났습니다(85.7%).

CPU가 4코어 환경임을 고려하면 이는 약 21% 수준으로, 여전히 적지 않은 사용률입니다.

> **CPU는 전체적으로 높은 부하 상태였으며, 병목의 주요 원인 중 하나로 판단됩니다.**

### 3-2. RAM 분석
| <img width="500" alt="image3" src="https://github.com/user-attachments/assets/39848433-2115-40a0-9a58-a3a9696a4aee" /> |
|:---:|
| RAM 사용량 |

램은 7.5GB 중 3.5GB 정도를 절반보다 조금 적은 양을 부하테스트 동안 사용하고 있었습니다.

> **RAM은 여유가 있는 상태로, 병목 지점은 아닌 것으로 판단됩니다.**


### 3-3. 쓰레드 분석
|<img width="1489" height="897" alt="스크린샷 2025-09-16 오후 3 58 48" src="https://github.com/user-attachments/assets/c03b7ec4-c084-491b-9761-d928d1735b02" /> |<img width="1485" height="892" alt="image4" src="https://github.com/user-attachments/assets/6f8ac183-a549-4743-a7d1-acb998cf4c40" />|
|:---:|:---:|
| 전체 쓰레드 | 상태별 쓰레드|

부하테스트 중 쓰레드 수가 급격히 증가했으며, 이는 다수의 HTTP 요청이 동시에 연결을 맺고 처리되었기 때문으로 보입니다.

그러나 이들 중 대부분은 timed-waiting 상태로 확인되었습니다.

> **해당 정보를 확인하면서 어떤 지점에서 발생한 병목으로 인해 대기 상태인 쓰레드가 급격하게 늘어난 것이 아닌지 의심이 되었습니다.**

<br>

## 4. 가설 및 검증

### 4-1. 인덱싱 적용

부하테스트 결과 CPU와 쓰레드에서 병목 현상이 확인되었고, 그 원인이 데이터베이스 조회 성능과 연관이 있을 것으로 생각했습니다. `user_id` 컬럼을 기준으로 조회가 발생하므로 해당 컬럼에 인덱스를 추가하여 성능 개선 여부를 검증하였습니다.

#### 테스트 결과
| | 지연시간 | 처리량 | 
|:---:|:---:|:---:|
| 전 | 13s | 30.85/s |
| 후 | 13s | 31.36/s |

#### 결과
기존과 비교하여 큰 변화가 없었으며 단순히 `user_id` 컬럼에 인덱스를 추가하는 것으로는 성능 향상을 얻을 수 없었습니다.

<br>

### 4-2. N+1 문제

성능 저하의 원인은 단순히 인덱스 부재가 아니라 **N+1 문제**일 가능성이 있다고 판단하였습니다.  

과제(Assignment) 엔티티를 조회할 때 연관된 강의(Lecture) 엔티티를 개별적으로 추가 조회하면서 불필요하게 많은 SQL 쿼리가 실행되고 있었습니다. 이를 해결하면 데이터 조회 과정에서 발생하는 병목을 줄이고 성능을 개선할 수 있을 것으로 예상하였습니다.  

```
Hibernate: select a1_0.id,a1_0.description,a1_0.end_date_time,a1_0.is_completed,a1_0.lecture_id,a1_0.start_date_time,a1_0.title,a1_0.user_user_id from assignment a1_0 where a1_0.user_user_id=?
Hibernate: select dl1_0.id,dl1_0.academic_year,dl1_0.area,dl1_0.classroom,dl1_0.completion_type,dl1_0.course_code,dl1_0.course_format,dl1_0.course_name,dl1_0.course_type,dl1_0.credits,dl1_0.curriculum,dl1_0.engineering_accreditation,dl1_0.evaluation_method,dl1_0.grade_type,dl1_0.instructor,dl1_0.offering_college,dl1_0.offering_department,dl1_0.offering_major,dl1_0.practical,dl1_0.remarks,dl1_0.semester,dl1_0.target_grade,dl1_0.team_teaching,dl1_0.theory from dg_lecture dl1_0 where dl1_0.id=?
Hibernate: select u1_0.user_id,u1_0.authority,u1_0.is_admin,u1_0.is_ems_logged_in,u1_0.is_tos_accepted,u1_0.name,u1_0.sns_id,u1_0.sns_type,u1_0.student_id from users u1_0 where u1_0.user_id=?
Hibernate: select dl1_0.id,dl1_0.academic_year,dl1_0.area,dl1_0.classroom,dl1_0.completion_type,dl1_0.course_code,dl1_0.course_format,dl1_0.course_name,dl1_0.course_type,dl1_0.credits,dl1_0.curriculum,dl1_0.engineering_accreditation,dl1_0.evaluation_method,dl1_0.grade_type,dl1_0.instructor,dl1_0.offering_college,dl1_0.offering_department,dl1_0.offering_major,dl1_0.practical,dl1_0.remarks,dl1_0.semester,dl1_0.target_grade,dl1_0.team_teaching,dl1_0.theory from dg_lecture dl1_0 where dl1_0.id=?
Hibernate: select dl1_0.id,dl1_0.academic_year,dl1_0.area,dl1_0.classroom,dl1_0.completion_type,dl1_0.course_code,dl1_0.course_format,dl1_0.course_name,dl1_0.course_type,dl1_0.credits,dl1_0.curriculum,dl1_0.engineering_accreditation,dl1_0.evaluation_method,dl1_0.grade_type,dl1_0.instructor,dl1_0.offering_college,dl1_0.offering_department,dl1_0.offering_major,dl1_0.practical,dl1_0.remarks,dl1_0.semester,dl1_0.target_grade,dl1_0.team_teaching,dl1_0.theory from dg_lecture dl1_0 where dl1_0.id=?
```

JPA의 **fetch join**을 적용하여 `Assignment` 엔티티를 조회할 때 `Lecture` 엔티티를 함께 가져오도록 수정하였습니다. 이를 통해 단일 쿼리로 필요한 데이터를 한 번에 조회할 수 있습니다.

``` java
@Query("select a from Assignment a join fetch a.lecture join fetch a.user where a.user = :user")
List<Assignment> findAllByUser(@Param("user") User user);
```

#### 테스트 결과
| | 지연시간 | 처리량 | 
|:---:|:---:|:---:|
| 전 | 13s | 30.85/s |
| 후 | 7s | 47.36/s |

#### 결과 
성능 저하의 원인은 `Assignment` 엔티티에서 `Lecture` 엔티티를 추가 조회하는 과정에서 발생한 **N+1 문제**였습니다.

`fetch join`을 적용하여 단일 쿼리로 데이터를 조회함으로써 병목 현상을 해소할 수 있었고, 실제로 95% 구간에서의 지연시간이 38% 감소하였으며, 처리량 또한 개선되었습니다.  

<img width="500" height="271" alt="image" src="https://github.com/user-attachments/assets/dfbae77d-5a64-4746-b810-dd936b2bb2e4" />

또한 개선 전과 비교하여 runnable 상태의 thread가 대폭 증가하여 병목이 확실히 개선됨을 알 수 있습니다.

불필요한 쿼리 호출로 인한 DB 부하가 줄어들면서 전체적인 성능이 안정화되었다고 해석됩니다.

<br>

## 5. 결론
서비스를 이용하는 유저들이 가장 많이 사용했던 과제 API에 부하테스트를 진행했습니다. 그리고 N+1 문제였음을 확인하였고 지연시간이 약 38% 개선되었습니다.

결과적으로 병목이 어느정도 해결되었지만 500명의 유저가 동시에 서비스를 이용할 때 7s의 지연시간은 여전히 사용자 이탈이 발생할만한 수치입니다.

이는 현재 서버로 사용중인 저전력 PC 오드로이드의 성능적 한계인 것으로 생각되며 지연 시간을 더 낮추기 위해서는 Scale-Up 혹은 Scale-Out이 필요하다고 생각합니다.

<br>

## 실험결과 첨부파일
> 개선 전 부하 테스트 Report

[k6 report(개선 전 부하 테스트)](https://github.com/user-attachments/files/22418809/k6.report.1.html)

> 인덱싱 적용 부하 테스트 Report

[k6 report(인덱싱 적용 부하 테스트)](https://github.com/user-attachments/files/22430160/k6.report.index.html)

> Fetch Join 적용 부하 테스트 Report

[k6 report(Fetch Join 적용 부하 테스트)](https://github.com/user-attachments/files/22430177/k6.report.fetch.join.html)

