# 산불 대응 시스템 API 명세서

| 항목 | 내용 |
| --- | --- |
| 문서 판 | v0.4 · 최신 설계 반영 검토본 |
| 작성·조회 기준일 | 2026-11-06 |
| 범위 | React·TypeScript·MapLibre 프론트엔드 ↔ FastAPI 백엔드 REST API |
| 상태 | 팀 검토용 설계 초안. 실제 서버 라우트·OpenAPI·실행 결과 대조 전 |
| 저장 예정 위치 | `docs/설계/API-명세서.md` |
| 기준 저장소 | `wildfire-guard-capstone/wfg-docs`의 main |
| 고정 기준 커밋 | `43dc969703cdb65192e07e1341785ce38bca60c6` |
| GitHub 반영 | `docs/api-spec-v0.4` 작업 브랜치 · main 병합 전 팀 검토용 |

> 이 문서는 노션 v0.3의 단순 복사본이 아닙니다. 아래 설계 문서 5개를 다시 읽어 현재 결정에 맞춰 API 계약을 재작성했습니다. 도메인 규칙의 근거와 API 작성자가 제안한 전송 형식을 구분합니다. 경로·권한표·응답 래퍼·작업 조회·확인 토큰·추가 DTO 필드는 서버 구현 완료나 팀 합의를 뜻하지 않습니다.

## 목차

- [1. 기준 문서와 적용 범위](#sources)
- [2. 공통 요청·응답 규칙](#common)
- [3. 권한과 상태값](#enums)
- [4. 공통 DTO](#dtos)
- [5. API 전체 목록](#index)
- [6. API 상세](#endpoints)
- [7. 지도·규칙 엔진 결과의 화면 적용](#rules)
- [8. 화면 연동 순서와 검증 항목](#flow)
- [9. 구현 전 팀 확인 사항](#questions)
- [10. 노션 v0.3에서 바뀐 내용](#changes)

<a id="sources"></a>

## 1. 기준 문서와 적용 범위


| 문서 | 표기된 최종 갱신일 | API에 반영한 내용 |
| --- | --- | --- |
| [DB-설계.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) | 2026-11-05 | 계정·산불·화선·예측·제안·정정·지도·기지·자원의 필드·제약·저장 흐름 |
| [규칙-엔진-설계.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/규칙-엔진-설계.md) | 2026-11-05 | 현재/P5 단계, S1~S6·E1~E7, context·items, 지도 targets·대피소·도로 구간 |
| [LLM-입력-정리.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/LLM-입력-정리.md) | 2026-10-27 | 수치·판정은 규칙 엔진, LLM은 문장 생성, 질의·정정 입력. 표 안에는 11/1~11/5 검토 메모도 있음 |
| [데이터-자원.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/데이터-자원.md) | 2026-10-30 | 자료 출처·확보 범위·제외 항목·추정값·시연 자료 한계. 이후 작성된 메모도 있음 |
| [위험도-정리.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/위험도-정리.md) | 2026-10-26 | 23개 변수·원가중치 합 0.94·가용 변수 가중평균·미결 계산 기준 |

파일 머리의 갱신일만으로 우선순위를 정하지 않습니다. 같은 주제의 명시된 결정과 해당 절의 최신 검토 메모를 우선했습니다. 예를 들어 DB의 `detail.targets` 옛 객체 예시보다 11/5 규칙 엔진의 대상 참조 형식을 따르고, 서로 충돌하거나 구현 방식이 없는 것은 9절에 남겼습니다. 여기서 인용한 매뉴얼 기준은 팀 설계 문서에 적힌 내용이며 이번 작업에서 원문 매뉴얼을 별도로 재검증하지 않았습니다.

### 포함·제외 범위

- 포함: 인증·계정 관리, 산불·화선 보고, 지도·시설·기지·자원 조회/관리, 예측·위험도, 제안·근거·비교, 질의·정정·로그.
- 제외: 산불 병합, 회원가입, 공식 대응단계 입력, 진화예상시간 입력·정정, 개별 인력 관리·실시간 자원 추적, 자동 대피소 배정, 지정 대피로·최단경로·정확한 통제 지점 생성, 실제 재난문자 발송 API.
- 백엔드 내부: ELMFIRE 실행, 규칙 엔진, EXAONE 호출, KURE·FAISS 검색, 원자료 적재는 백엔드가 연결합니다. 프론트는 이 서비스의 REST API를 호출합니다. 백엔드↔AI 내부 서비스 계약은 이 명세서의 별도 구현 문서가 필요합니다.

<a id="common"></a>

## 2. 공통 요청·응답 규칙

### 2.1 전송 규칙 — API 설계 제안

| 항목 | 계약 초안 |
| --- | --- |
| Base URL | 개발·운영 호스트는 백엔드와 확정. 공통 경로 `/api/v1` |
| 요청·응답 | UTF-8 JSON, `Content-Type: application/json`. 본문이 없는 GET/DELETE에는 Content-Type 불필요 |
| 인증 | AUTH-01 제외 `Authorization: Bearer ACCESS_TOKEN`. JWT의 role만 신뢰하지 않고 매 요청 users 재확인 |
| ID | DB bigint PK/FK를 JSON에서는 10진수 문자열로 반환. 경로 ID도 양의 10진수 문자열. 법정동코드 역시 문자열 |
| 좌표 | GeoJSON, EPSG:4326, 순서 `[경도, 위도]`. 서버 저장·면적·거리 계산은 EPSG:5179 |
| 시각 | 타임존을 포함한 ISO8601 DateTime. 화면 표시는 Asia/Seoul. 날짜는 YYYY-MM-DD |
| 필드명 | 영어 snake_case. 쓰기 DTO에 명시하지 않은 필드는 422로 거부하는 안 |
| PATCH | 생략=유지, null=허용된 nullable 필드만 비우기. 배열·attrs 객체를 보내면 전체 교체 |
| 미확인 값 | 숫자 0·false로 대체하지 않고 null 또는 필드별 미제공 사유 사용. 빈 목록은 [] |
| 목록 | `Page<T>` 사용. 기본 limit 20·최대 100. 마지막 정렬키를 담은 불투명 cursor, 동일 필터에서만 재사용 |
| 지도 목록 | bbox 필수, 기본 limit 500·최대 1000, GeoJSON 최상위 data 안에 next_cursor·has_more를 둠 |
| 범위 검증 | 하위 ID가 URL 산불에 속하는지, 사용자 관할·역할에 맞는지 항상 서버 검증 |
| 응답 캐시 | 개인정보·계정·질의·정정·로그는 `Cache-Control: no-store` 제안. 공개 바탕지도 캐시는 제공자 정책 적용 |

서버 라우팅 시 `/resources/summary`, `/proposals/latest` 같은 고정 경로를 ID 경로와 구분합니다. 도형의 페이지 누락·중복은 프론트가 ID로 정리하고 `has_more=false`까지 받아야 합니다.

### 2.2 성공 응답

6절의 **Response Example은 이 래퍼까지 포함한 전체 JSON**입니다. DTO 예시는 `data` 안의 값만 표시합니다.

```json
{
  "data": {
    "id": "100"
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

목록의 data는 다음과 같습니다. cursor와 limit은 API 제안값입니다.

```json
{
  "items": [
    {
      "id": "100"
    }
  ],
  "next_cursor": null,
  "has_more": false
}
```

`204 No Content`는 JSON 래퍼 없이 본문이 없습니다. 아직 최신 제안서가 없을 때는 `200`과 `data: null`을 반환합니다. 특정 PK가 없으면 404입니다. 모든 예시 ID·좌표·수량·인물·토큰은 형식 설명용이며 실제 운영 자료나 계산 결과가 아닙니다. 각 예시는 독립적이며 예시끼리 하나의 일관된 산불 시나리오를 구성하지 않습니다.

### 2.3 오류 응답

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "입력값을 확인해주세요.",
    "fields": [
      {
        "field": "observed_at",
        "reason": "타임존이 포함된 시각이어야 합니다."
      }
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

| HTTP 상태 | 용도 |
| --- | --- |
| 400 | 깨진 JSON 등 파싱 실패 |
| 401 | 토큰 없음·만료·무효 또는 로그인 실패 |
| 403 | 역할·관할 불일치, 잠김·회수된 계정 |
| 404 | 대상 없음 또는 권한 범위 밖 객체를 숨기는 경우. 서버 정책 통일 필요 |
| 409 | 종료된 산불 쓰기, 이미 실행 중, 오래된 revision/기준 화선, 중복 이름, 마지막 지휘권자 회수 등 |
| 422 | 필수·형식·범위·도형 검증 실패 |
| 503 | 기상·AI 등 외부 의존 서비스 사용 불가 |
| 500 | 처리 중 예상하지 못한 오류. 내부 스택·비밀정보는 응답에 제외 |

오류 `fields`는 배열이며 필드 오류가 없으면 []입니다. 각 API에는 위 공통 오류 외에 주요 업무 오류를 적었습니다. 프론트는 HTTP 상태와 error.code를 함께 처리합니다.

### 2.4 중복·동시 작업 — 구현 합의 필요

- INC-02, PER-02, PER-03, PRED-01, COR-01, PROP-05에는 `Idempotency-Key`(요청별 UUID 문자열)를 필수로 제안합니다. 네트워크 재시도는 같은 키·같은 본문, 새 사용자 동작은 새 키입니다.
- 키는 사용자·산불(해당 시)·경로·요청 본문에 결합합니다. 동일 요청 재전송은 이미 확정된 정정/보고/작업 결과를 반환하고 행·버전을 중복 생성하지 않습니다. 같은 키로 다른 본문은 409 `IDEMPOTENCY_KEY_REUSED`입니다.
- 산불별 advisory lock으로 중복 예측·제안 커밋·종료·정정을 조율합니다. 계산 중에는 긴 DB 트랜잭션을 열지 않고 커밋 직전에 종료·화선·상황 변경을 다시 확인합니다.
- situation_revision은 DB 컬럼으로 확정하지 않은 불투명 토큰입니다. 최신 정정 id만으로 충분한지, 관리자/배치 변경·기상·자원 갱신을 어떻게 감지할지 Q-05·Q-06에서 합의합니다.
- 작업·중복키 결과 보존, 프로세스 재시작·다중 worker, 키 만료·응답 재전달은 Q-05의 구현 전 확인 사항입니다.

<a id="enums"></a>

## 3. 권한과 상태값

### 3.1 역할별 범위 — 권한 API 초안

| 역할 | 역할명 | 제공 기능 |
| --- | --- | --- |
| COMMANDER | 통합지휘권자 | 관할 현황·지도·자원·예측·위험도·제안·근거·질의·정정·이력·종료 |
| VIEWER | 열람자 | 관할 현황·지도·자원·예측·위험도 읽기. 제안·AI 접근은 이 초안에서 제외, Q-08 확인 |
| REPORTER | 상황 보고자 | 관할 산불 등록·보고 정보·보고 지도·화선 보고·정정. 분석 데이터는 제한 |
| ADMIN | 전산 관리자 | 계정·기지·자원 관리와 해당 로그. 지휘권자 기능을 자동 부여하지 않음 |

위 세부 접근 범위는 DB의 4개 역할을 화면 기능에 매핑한 제안입니다. 다른 계정이 보낸 role·admin_area_id로 조회 권한을 넓히지 않습니다. 인접 시군 자원·시설은 백엔드가 확정한 지원 조회 범위에서만 제공합니다.

### 3.2 공통 상태 코드


| 타입 | 값 | 근거·의미 |
| --- | --- | --- |
| UserRole | COMMANDER, VIEWER, REPORTER, ADMIN | DB users.role |
| IncidentStatus | IN_PROGRESS, ENDED | 실측 화선 없음은 별도 상태 대신 v0으로 구분 |
| ResponseStage | INITIAL, STAGE1, STAGE2 | 현재·예측 판정 단계. 공식 발령 단계가 아님 |
| AlertLevel | ATTENTION, CAUTION, ALERT, SERIOUS | DB 위기경보, 유지·활용 여부 팀 논의 |
| ItemAxis | SUPPRESSION, EVACUATION | 진화/대피 |
| ItemCategory | IMMEDIATE, STANDBY, CONSULT, REQUEST | 즉시/대기/협의/요청 |
| ResourceKind | HELICOPTER, VEHICLE | 개별 인력 없음 |
| ResourceStatus | AVAILABLE, DEPLOYED, MAINTENANCE | 대기/투입/정비 |
| FacilityType | SHELTER, VULNERABLE, HOSPITAL, FIRE_STATION, POLICE, HERITAGE | DB facilities.type, 적재 CSV의 소문자 type은 적재 시 변환 |
| AdminLevel | SIDO, SIGUNGU, EMD | 행정구역 |
| OperationStatus | QUEUED, RUNNING, SUCCEEDED, FAILED | API 작업 계층 제안, predictions DB ENUM이 아님 |

DB의 영문 ENUM과 규칙 엔진의 한글 표시는 구분합니다. 예를 들어 `detail.categories=["즉시", "요청"]`는 규칙 원문이고, 외부 항목의 `categories=["IMMEDIATE", "REQUEST"]`는 서버 어댑터가 매핑한 값입니다. 현재 문서에 없는 미래의 응답 문자열은 프론트에서 기본 표시로 처리할 수 있어야 합니다.

<a id="dtos"></a>

## 4. 공통 DTO

`ID`는 앞 절의 bigint 문자열, `DateTime`은 타임존 포함 ISO8601입니다. `Geometry`는 4326 GeoJSON geometry이며 Feature 전체가 아닙니다. `Page<T>`는 `{items: T[], next_cursor: string 또는 null, has_more: boolean}`입니다. 객체 필드는 표에서 선택으로 표시한 것 외에는 필수입니다. `A / B`로 한 줄에 묶인 필드는 각각 A와 B 필드이며 슬래시가 실제 JSON 키에 포함되지는 않습니다.

<a id="dto-user"></a>

### User


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | 사용자 PK |
| login_id | string | 필수 | 최대 50자, 대소문자 무시 유일 |
| name | string | 필수 | 최대 50자 |
| organization | string | 필수 | 최대 100자 |
| role | UserRole | 필수 | 서버 DB에서 확인한 역할 |
| admin_area_id | ID 또는 null | 필수·null 허용 | 시군구 관할, ADMIN만 null |

비밀번호 해시는 어떤 응답에도 반환하지 않습니다.

**DTO Example**

```json
{
  "id": "1",
  "login_id": "cmd01",
  "name": "시연 사용자",
  "organization": "시연 지휘본부",
  "role": "COMMANDER",
  "admin_area_id": "10"
}
```

<a id="dto-adminuser"></a>

### AdminUser


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | 사용자 PK |
| login_id | string | 필수 | 최대 50자, 대소문자 무시 유일 |
| name | string | 필수 | 최대 50자 |
| organization | string | 필수 | 최대 100자 |
| role | UserRole | 필수 | 서버 DB에서 확인한 역할 |
| admin_area_id | ID 또는 null | 필수·null 허용 | 시군구 관할, ADMIN만 null |
| failed_login_count | integer | 필수 | 0 이상. 5 이상이면 잠김 |
| created_at / updated_at | DateTime | 필수 | 발급·변경 시각 |
| deleted_at | DateTime 또는 null | 필수·null 허용 | 회수 시각. null이면 사용 중 |

**DTO Example**

```json
{
  "id": "1",
  "login_id": "cmd01",
  "name": "시연 사용자",
  "organization": "시연 지휘본부",
  "role": "COMMANDER",
  "admin_area_id": "10",
  "failed_login_count": 0,
  "created_at": "2025-03-22T11:55:00+09:00",
  "updated_at": "2025-03-22T11:55:00+09:00",
  "deleted_at": null
}
```

<a id="dto-authtoken"></a>

### AuthToken


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| access_token | string | 필수 | JWT 예시 문자열은 실제 토큰이 아님 |
| token_type | string | 필수 | bearer |
| expires_in | integer | 필수 | 초. 1800은 API 초안 제안값 |
| user | User | 필수 | 현재 역할·관할 |

**DTO Example**

```json
{
  "access_token": "EXAMPLE_ACCESS_TOKEN",
  "token_type": "bearer",
  "expires_in": 1800,
  "user": {
    "id": "1",
    "login_id": "cmd01",
    "name": "시연 사용자",
    "organization": "시연 지휘본부",
    "role": "COMMANDER",
    "admin_area_id": "10"
  }
}
```

<a id="dto-resourcebase"></a>

### ResourceBase


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | 기지 PK |
| name | string | 필수 | 최대 100자, 유일 |
| org_type | string | 필수 | 최대 20자. forest / fire / local_lease / local 등 |
| geometry | Point 또는 null | 필수·null 허용 | 고정 기지 위치, 실시간 자원 위치가 아님 |
| coverage_area_ids | ID[] | 필수 | 시도 또는 시군구 ID 목록 |
| distance_km | number 또는 null | 필수·null 허용 | incident_id가 있을 때 발화점부터 직선거리. 미지정·위치 미상이면 null |
| jurisdiction | LOCAL / SUPPORT 또는 null | 필수·null 허용 | incident_id 기준 관할/응원. 미지정이면 null |

관할 판정은 산불의 시군구 또는 그 상위 시도가 coverage_area_ids에 포함되는지로 합니다. API의 LOCAL/SUPPORT 코드는 표시용 제안입니다.

**DTO Example**

```json
{
  "id": "31",
  "name": "시연 항공관리소",
  "org_type": "forest",
  "geometry": {
    "type": "Point",
    "coordinates": [
      128.6,
      36.37
    ]
  },
  "coverage_area_ids": [
    "10"
  ],
  "distance_km": 22.3,
  "jurisdiction": "LOCAL"
}
```

<a id="dto-resource"></a>

### Resource


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | 자원 PK |
| kind | ResourceKind | 필수 | HELICOPTER / VEHICLE |
| name | string | 필수 | 최대 50자, 유일 |
| base_id | ID | 필수 | 소속 resource_bases.id |
| status | ResourceStatus | 필수 | AVAILABLE / DEPLOYED / MAINTENANCE |
| attrs | object | 필수 | 헬기: model·capacity_l, 차량: vehicle_type·capacity_l. 값이 없으면 키 생략 |

기지와 소속을 분리합니다. DB에 없는 updated_at·deleted_at·실시간 좌표를 자원 DTO에 추가하지 않습니다. 삭제는 실제 행 삭제입니다.

**DTO Example**

```json
{
  "id": "51",
  "kind": "HELICOPTER",
  "name": "DEMO-HELI-01",
  "base_id": "31",
  "status": "AVAILABLE",
  "attrs": {
    "model": "시연 기종",
    "capacity_l": 8000
  }
}
```

<a id="dto-situation"></a>

### Situation


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| villages | object<string, VillageState> | 선택·null 불가 | 키는 villages.id. VillageState의 evacuated·evac_order는 boolean. 미기록 마을은 규칙 엔진에서 false로 해석 |
| deployed | object | 선택·null 불가 | helicopter·vehicle·crew별 투입 수. 각 묶음이 있으면 total 필수, 0 이상 정수 |
| deployed.helicopter | object | 선택 | total, forest, local, fire, military |
| deployed.vehicle | object | 선택 | total, forest_vehicle, fire_truck, other |
| deployed.crew | object | 선택 | total, special_crew, forest_officials, fire, military, police, other |
| disaster_sms_sent | boolean | 선택·null 불가 | 사용자가 정정한 송출 여부. E6의 읍면별 Set/Go 판정 이력과 다름 |

모르는 종류별 투입 수는 생략합니다. total만 정정해도 되므로 종류별 합계와 total의 항상 일치 여부는 별도 검토합니다. 인력은 개별 자원·보유 현황으로 관리하지 않습니다.

**DTO Example**

```json
{
  "villages": {
    "1024": {
      "evacuated": false,
      "evac_order": true
    }
  },
  "deployed": {
    "helicopter": {
      "total": 3,
      "forest": 2,
      "local": 1
    },
    "vehicle": {
      "total": 12
    },
    "crew": {
      "total": 85
    }
  },
  "disaster_sms_sent": false
}
```

<a id="dto-perimeter"></a>

### Perimeter


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / incident_id | ID | 필수 | 화선 PK / 소속 산불 |
| version_no | integer | 필수 | 0=발화점, 1 이상=실측 화선 |
| geometry | Point 또는 MultiPolygon | 필수 | v0은 Point, v1 이상은 MultiPolygon |
| observed_at | DateTime | 필수 | v0은 발생시각, 없으면 접수시각 |
| reported_by | ID | 필수 | 상황 보고자 계정 |
| created_at | DateTime | 필수 | 등록시각 |
| corrected_at | DateTime 또는 null | 필수·null 허용 | null이면 유효 |
| area_ha | number | 필수 | 서버 공간 계산, v0은 0 |

현재 화선은 corrected_at이 null인 행 중 version_no 최대입니다. 정정은 기존 화선을 덮어쓰지 않고 다음 버전을 추가합니다.

**DTO Example**

```json
{
  "id": "21",
  "incident_id": "100",
  "version_no": 1,
  "geometry": {
    "type": "MultiPolygon",
    "coordinates": [
      [
        [
          [
            128.6,
            36.37
          ],
          [
            128.602,
            36.37
          ],
          [
            128.602,
            36.372
          ],
          [
            128.6,
            36.37
          ]
        ]
      ]
    ]
  },
  "observed_at": "2025-03-22T11:55:00+09:00",
  "reported_by": "3",
  "created_at": "2025-03-22T11:55:00+09:00",
  "corrected_at": null,
  "area_ha": 8.0
}
```

<a id="dto-incident"></a>

### Incident


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | 산불 사건 PK |
| name | string | 필수 | 서버 생성: 시군구·읍면동·발생 날짜, 발생시각 없으면 접수 날짜 |
| ignition_point | Point | 필수 | 발화점 |
| address | string | 필수 | 최대 200자 |
| admin_area_id | ID | 필수 | 발화점에서 서버가 찾은 읍면동 |
| started_at | DateTime 또는 null | 필수·null 허용 | 발생시각 |
| reported_at | DateTime | 필수 | 접수시각 |
| report_text | string 또는 null | 필수·null 허용 | 신고내용 |
| reporter_org | string 또는 null | 필수·null 허용 | 최대 100자, 신고 기관명. 기관별 접수 목록 아님 |
| reporter_name / reporter_phone | string 또는 null | 필수·null 허용 | DB 저장은 성명 50자·연락처 20자 필수, 미상 허용. 역할별 응답 마스킹 정책은 Q-08 |
| status | IncidentStatus | 필수 | 진행/종료 |
| current_stage | ResponseStage | 필수 | 화선 보고·정정 때 판정한 값. 사용자 입력 금지 |
| alert_level | AlertLevel | 필수 | DB에 유지하되 규칙 엔진 사용 여부는 Q-03 |
| situation | Situation | 필수 | 현재 승인된 상황 |
| reported_by / created_at | ID / DateTime | 필수 | 최초 등록 계정 / 등록시각 |
| ended_at / ended_by | DateTime 또는 null / ID 또는 null | 필수·null 허용 | 종료 정보 |
| current_perimeter_id / current_perimeter_version | ID / integer | 필수 | 현재 화선 참조, v0 포함 |
| latest_prediction_id / latest_proposal_id | ID 또는 null | 필수·null 허용 | 성공한 최신 결과 참조, 아직 없으면 null |
| needs_reprediction | boolean | 필수 | 최신 화선 버전 > 현재 예측 기준 화선 버전. 예측이 없으면 true |
| proposal_stale | boolean | 필수 | 정정은 저장됐지만 현재 제안서가 이를 반영하지 못함 |
| situation_revision | string | 필수 | 동시 수정 방지용 불투명 API 토큰 제안. DB 컬럼 추가 확정 아님 |

조회 참조들은 일관된 스냅샷으로 읽습니다. REPORTER에는 situation·예측·제안 참조 등 분석용 필드를 제외하고 보고 정보와 화선만 제공하는 안이며, 권한 범위는 Q-08에서 확정합니다.

**DTO Example**

```json
{
  "id": "100",
  "name": "의성군 안평면 산불 (2025-03-22)",
  "ignition_point": {
    "type": "Point",
    "coordinates": [
      128.6,
      36.37
    ]
  },
  "address": "시연 발생 장소",
  "admin_area_id": "11",
  "started_at": "2025-03-22T11:24:00+09:00",
  "reported_at": "2025-03-22T11:25:00+09:00",
  "report_text": "시연 산불 신고",
  "reporter_org": null,
  "reporter_name": "미상",
  "reporter_phone": "미상",
  "status": "IN_PROGRESS",
  "current_stage": "STAGE1",
  "alert_level": "ATTENTION",
  "situation": {
    "villages": {
      "1024": {
        "evacuated": false,
        "evac_order": true
      }
    },
    "deployed": {
      "helicopter": {
        "total": 3,
        "forest": 2,
        "local": 1
      },
      "vehicle": {
        "total": 12
      },
      "crew": {
        "total": 85
      }
    },
    "disaster_sms_sent": false
  },
  "reported_by": "3",
  "created_at": "2025-03-22T11:25:00+09:00",
  "ended_at": null,
  "ended_by": null,
  "current_perimeter_id": "21",
  "current_perimeter_version": 1,
  "latest_prediction_id": "41",
  "latest_proposal_id": "71",
  "needs_reprediction": false,
  "proposal_stale": false,
  "situation_revision": "rev-example-1"
}
```

<a id="dto-incidentsummary"></a>

### IncidentSummary


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / name / status | ID / string / IncidentStatus | 필수 | 산불 식별·목록 표시 |
| admin_area_id / reported_at | ID / DateTime | 필수 | 읍면동 / 접수시각 |
| current_stage | ResponseStage | 필수 | 현재 판정 단계 |
| current_perimeter_version | integer | 필수 | 0이면 실측 화선 없음 |
| needs_reprediction / proposal_stale | boolean | 필수 | 갱신 안내 |

**DTO Example**

```json
{
  "id": "100",
  "name": "의성군 안평면 산불 (2025-03-22)",
  "status": "IN_PROGRESS",
  "admin_area_id": "11",
  "reported_at": "2025-03-22T11:25:00+09:00",
  "current_stage": "STAGE1",
  "current_perimeter_version": 1,
  "needs_reprediction": false,
  "proposal_stale": false
}
```

<a id="dto-prediction"></a>

### Prediction


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / incident_id / perimeter_id | ID | 필수 | 성공 예측 PK / 산불 / 시작 화선 |
| perimeter_version | integer | 필수 | 기준 화선 버전, 계산 응답 필드 |
| base_at | DateTime | 필수 | perimeter.observed_at. 실행 버튼을 누른 시각 아님 |
| requested_by / created_at | ID / DateTime | 필수 | 실행 계정 / 저장시각 |
| slices | SliceSummary[8] | 필수 | hour=1~8 각각 1개. 각 객체 hour, at=base_at+hour, area_ha |

predictions에는 성공 결과만 저장합니다. 진행·실패 상태는 Operation에서 관리하는 API 제안이며 predictions 테이블 컬럼이 아닙니다.

**DTO Example**

```json
{
  "id": "41",
  "incident_id": "100",
  "perimeter_id": "21",
  "perimeter_version": 1,
  "base_at": "2025-03-22T11:55:00+09:00",
  "requested_by": "1",
  "created_at": "2025-03-22T12:05:00+09:00",
  "slices": [
    {
      "hour": 1,
      "at": "2025-03-22T12:55:00+09:00",
      "area_ha": 21
    },
    {
      "hour": 2,
      "at": "2025-03-22T13:55:00+09:00",
      "area_ha": 40
    },
    {
      "hour": 3,
      "at": "2025-03-22T14:55:00+09:00",
      "area_ha": 68
    },
    {
      "hour": 4,
      "at": "2025-03-22T15:55:00+09:00",
      "area_ha": 105
    },
    {
      "hour": 5,
      "at": "2025-03-22T16:55:00+09:00",
      "area_ha": 150
    },
    {
      "hour": 6,
      "at": "2025-03-22T17:55:00+09:00",
      "area_ha": 190
    },
    {
      "hour": 7,
      "at": "2025-03-22T18:55:00+09:00",
      "area_ha": 228
    },
    {
      "hour": 8,
      "at": "2025-03-22T19:55:00+09:00",
      "area_ha": 260
    }
  ]
}
```

<a id="dto-weather"></a>

### Weather


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| source | KMA_FORECAST / AWS_OBS | 필수 | 운영 예보 / 과거 시연 관측 |
| issued_at | DateTime 또는 null | 필수·null 허용 | 예보 발표시각. 관측은 null |
| location | object | 필수 | 예보 {nx: integer, ny: integer}, 관측 {stn: string} |
| hours | WeatherHour[9] | 필수 | base_at을 내림한 정시부터 8시간 뒤까지, 1시간 간격 9개 |
| hours[].at | DateTime | 필수 | 대상 정시 |
| hours[].wind_dir / wind_speed | number | 필수 | 불어오는 방향 0~360° / m/s, 풍속 0 이상 |
| hours[].temp / humidity / precip | number | 필수 | ℃ / 0~100% / 1시간 강수 mm, 강수 0 이상 |

시간별 기상은 서버가 선택합니다. 기압·지면온도·시정·특보는 예측 weather에 추가하지 않습니다. 한 지점 값을 전체 범위에 적용하는 한계를 표시합니다.

**DTO Example**

```json
{
  "source": "AWS_OBS",
  "issued_at": null,
  "location": {
    "stn": "EXAMPLE_STATION"
  },
  "hours": [
    {
      "at": "2025-03-22T11:00:00+09:00",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T12:00:00+09:00",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T13:00:00+09:00",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T14:00:00+09:00",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T15:00:00+09:00",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T16:00:00+09:00",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T17:00:00+09:00",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T18:00:00+09:00",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T19:00:00+09:00",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    }
  ]
}
```

<a id="dto-operation"></a>

### Operation


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | string | 필수 | 불투명 작업 ID |
| kind | PREDICTION / PROPOSAL_REGENERATION | 필수 | 장시간 작업 종류 |
| status | OperationStatus | 필수 | QUEUED / RUNNING / SUCCEEDED / FAILED |
| stage | string 또는 null | 필수·null 허용 | SPREAD / RISK / RULES / EVIDENCE / PROPOSAL_GENERATION / COMMIT 등 API 제안 |
| result | object 또는 null | 필수·null 허용 | 성공 때 prediction_id·proposal_id 반환. 정정 관련 작업은 correction_id도 가능 |
| error | object 또는 null | 필수·null 허용 | 실패 때 code·message. HTTP 오류와 같은 코드 체계 |

작업 큐·보존기간·재시작 후 복구는 Q-05에서 확정해야 합니다. 서버 구현 없이 메모리 작업만으로 운영 계약이 충족됐다고 보지 않습니다.

**DTO Example**

```json
{
  "id": "op-example",
  "kind": "PREDICTION",
  "status": "RUNNING",
  "stage": "PROPOSAL_GENERATION",
  "result": null,
  "error": null
}
```

<a id="dto-adminarea"></a>

### AdminArea


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | 행정구역 PK |
| level | SIDO / SIGUNGU / EMD | 필수 | 시도 / 시군구 / 읍면동 |
| code / name | string | 필수 | 법정동코드 10자리 / 이름 50자 |
| parent_id | ID 또는 null | 필수·null 허용 | 시도만 null |
| neighbor_ids | ID[] | 필수 | 인접 시군구, 다른 level은 빈 배열 |

**DTO Example**

```json
{
  "id": "11",
  "level": "EMD",
  "code": "4773036000",
  "name": "안평면",
  "parent_id": "10",
  "neighbor_ids": []
}
```

<a id="dto-village"></a>

### Village


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / code / admin_area_id | ID / string / ID | 필수 | 마을 PK / 법정동코드 10자리 / 소속 읍면동 |
| name | string | 필수 | 최대 50자 |
| geometry / label_point | MultiPolygon / Point | 필수 | 리 경계 / 이름 표시용 점. 대피 판정은 점이 아닌 경계로 수행 |
| population / population_month | integer / date | 필수 | 주민등록 총인구 / 기준 달 첫날 |
| pop65_est / pop0_14_est / houses_est | integer 또는 null | 필수·null 허용 | 65세 이상·0~14세 추정 인구 / 주택 동수. 실제 정밀 인구가 아님 |

삭제된 villages.neighbor_ids는 반환하지 않습니다. 연령 추정값과 자료 기준시점을 화면에 구분합니다.

**DTO Example**

```json
{
  "id": "1024",
  "code": "4773036021",
  "admin_area_id": "11",
  "name": "시연리",
  "geometry": {
    "type": "MultiPolygon",
    "coordinates": [
      [
        [
          [
            128.6,
            36.37
          ],
          [
            128.602,
            36.37
          ],
          [
            128.602,
            36.372
          ],
          [
            128.6,
            36.37
          ]
        ]
      ]
    ]
  },
  "label_point": {
    "type": "Point",
    "coordinates": [
      128.6,
      36.37
    ]
  },
  "population": 150,
  "population_month": "2025-02-01",
  "pop65_est": 85,
  "pop0_14_est": 6,
  "houses_est": 30
}
```

<a id="dto-facility"></a>

### Facility


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / type | ID / FacilityType | 필수 | 시설 PK / 분류 |
| subtype / name | string | 필수 | 최대 30자 / 최대 100자 |
| capacity / capacity_unit | integer 또는 null / string 또는 null | 필수·null 허용 | 수용 인원·정원·병상. 미확인 시 null |
| address / phone | string 또는 null | 필수·null 허용 | 주소 200자 / 대표번호 20자 |
| village_id | ID 또는 null | 필수·null 허용 | 관할 밖 시설은 null 가능 |
| sigungu_id | ID | 필수 | 시설 소속 시군구 |
| geometry | Geometry | 필수 | 현재 적재는 점, 면도 허용 |
| attrs | object | 필수 | 종류별 속성. SHELTER: designated·used_2025·area_m2·evac_area_2025·tier, VULNERABLE: occupancy·daytime_only·emergency, HOSPITAL: emergency, FIRE_STATION: center |

시설 위치로 지도에 그리며, 응급 이송처는 attrs.emergency=true인 시설을 조회합니다. 대피소 정원은 실시간 남은 자리와 다릅니다.

**DTO Example**

```json
{
  "id": "55",
  "type": "SHELTER",
  "subtype": "학교",
  "name": "시연 대피소",
  "capacity": 109,
  "capacity_unit": "수용 인원",
  "address": null,
  "phone": null,
  "village_id": "1024",
  "sigungu_id": "10",
  "geometry": {
    "type": "Point",
    "coordinates": [
      128.6,
      36.37
    ]
  },
  "attrs": {
    "designated": true,
    "tier": "거점 시설"
  }
}
```

<a id="dto-mapfeature"></a>

### MapFeature


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | map_features PK |
| kind | string | 필수 | road / track / power_line / power_tower / substation / water / house |
| name | string 또는 null | 필수·null 허용 | 최대 100자, 원본 이름 없으면 null |
| geometry | Geometry | 필수 | 점·선·면 |
| attrs | object | 필수 | road: highway·ref, power_line: voltage(V), substation: type·source, water: type·area_ha, house: use·code |

동적으로 만든 도로·송전선 구간명은 제안 facts에서 표시합니다. DB 원본 name을 임의로 바꾸지 않습니다.

**DTO Example**

```json
{
  "id": "1021",
  "kind": "road",
  "name": "시연도로",
  "geometry": {
    "type": "LineString",
    "coordinates": [
      [
        128.6,
        36.37
      ],
      [
        128.61,
        36.38
      ]
    ]
  },
  "attrs": {
    "highway": "primary",
    "ref": "5"
  }
}
```

<a id="dto-ruledetail"></a>

### RuleDetail


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| status | string | 선택 | 규칙 엔진 항목별 상황 코드. 공통 ENUM 아님 |
| finding | string | 필수 | 규칙 판정 한 줄 |
| facts | object | 필수 | 항목별 구조는 7절. 수치·판정은 규칙 엔진 결과 |
| targets | string[] | 필수 | 최신 규칙 문서 형식. village:ID / facility:ID / map_feature:ID / resource_base:ID / admin_area:ID 등 |
| categories | string[] | 필수 | 엔진 원문 즉시·대기·협의·요청. 외부 ProposalItem.categories는 영문 ENUM으로 매핑 |
| withheld | string[] | 필수 | 근거·자료가 없어 제공하지 못한 내용 |
| rule_basis | string[] | 필수 | 규칙 근거 코드. 본문 공개·검색 여부는 EVD-01 정책에 따름 |

DB의 옛 detail.targets 객체 예시보다 최신 규칙 엔진의 문자열 배열 형식을 우선하는 API 초안입니다. 이름을 넣은 admin_area:안평면, kind가 포함된 map_feature:power_line:3 예시가 혼재하므로 출력 전 서버 어댑터가 PK 기반으로 변환해야 합니다(Q-04). detail의 원문 facts·categories는 그대로 두고 외부 표시 필드와 구분합니다.

**DTO Example**

```json
{
  "status": "maintain",
  "finding": "현재 기준과 P5 기준 단계 유지",
  "facts": {
    "now": {
      "stage": "1단계",
      "area_ha": 8,
      "wind_ms": 2.1,
      "facility": 12
    },
    "p5": {
      "stage": "1단계",
      "area_ha": 9,
      "wind_ms": 2.6,
      "facility": 12
    }
  },
  "targets": [],
  "categories": [
    "대기"
  ],
  "withheld": [
    "진화예상시간: MVP 제외"
  ],
  "rule_basis": [
    "SM-p073"
  ]
}
```

<a id="dto-citationref"></a>

### CitationRef


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | citations PK, 근거 조회용 |
| k | integer | 필수 | 해당 문장 안의 [k], 1 이상 |
| chunk_id / chunk_code | ID / string | 필수 | evidence_chunks.id / 변경하지 않는 code |
| doc_name | string | 필수 | 문서명 |
| page | integer 또는 null | 필수·null 허용 | 법령처럼 쪽 없으면 null |
| section | string 또는 null | 필수·null 허용 | 항목·조항 |

**DTO Example**

```json
{
  "id": "81",
  "k": 1,
  "chunk_id": "91",
  "chunk_code": "SM-p073-01",
  "doc_name": "시연 근거 문서",
  "page": 73,
  "section": "대응단계 판단기준"
}
```

<a id="dto-proposalitem"></a>

### ProposalItem


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | proposal_items PK |
| item_code | string | 필수 | 현재 S1~S7·E1~E7. DB 문자열이며 폐기 항목 이름도 코드 상수에 보존 |
| axis | ItemAxis | 필수 | 진화/대피 |
| categories | ItemCategory[] | 필수 | 1개 이상, 중복 없이 |
| text | string | 필수 | LLM 문장. 인용 번호는 citations.k와 대응 |
| detail | RuleDetail | 필수 | 판정 이유·수치·지도 대상·미제공 사유 |
| citations | CitationRef[] | 필수 | 이 문장에 연결된 근거 |
| change_type | ADDED / CHANGED / UNCHANGED | 필수 | 직전 제안과 item_code별 categories·text 비교한 표시용 값 |

발동하지 않거나 근거를 못 찾은 항목은 행을 만들지 않습니다. 미발동을 안전하다는 의미로 해석하지 않습니다. 근거 부족은 이벤트 로그로 별도 안내합니다.

**DTO Example**

```json
{
  "id": "72",
  "item_code": "S1",
  "axis": "SUPPRESSION",
  "categories": [
    "STANDBY"
  ],
  "text": "현재 단계 기준을 유지합니다 [1].",
  "detail": {
    "status": "maintain",
    "finding": "현재 기준과 P5 기준 단계 유지",
    "facts": {
      "now": {
        "stage": "1단계",
        "area_ha": 8,
        "wind_ms": 2.1,
        "facility": 12
      },
      "p5": {
        "stage": "1단계",
        "area_ha": 9,
        "wind_ms": 2.6,
        "facility": 12
      }
    },
    "targets": [],
    "categories": [
      "대기"
    ],
    "withheld": [
      "진화예상시간: MVP 제외"
    ],
    "rule_basis": [
      "SM-p073"
    ]
  },
  "citations": [
    {
      "id": "81",
      "k": 1,
      "chunk_id": "91",
      "chunk_code": "SM-p073-01",
      "doc_name": "시연 근거 문서",
      "page": 73,
      "section": "대응단계 판단기준"
    }
  ],
  "change_type": "UNCHANGED"
}
```

<a id="dto-proposal"></a>

### Proposal


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / incident_id / prediction_id | ID | 필수 | 제안서 / 산불 / 기준 예측 |
| perimeter_version / seq | integer | 필수 | 기준 화선 버전 0 이상 / 같은 화선 내 순번 1 이상 |
| version | string | 필수 | 표시용 v{perimeter_version}.{seq} |
| trigger | PREDICTION / CORRECTION | 필수 | 생성 계기 |
| correction_id | ID 또는 null | 필수·null 허용 | 정정으로 만든 제안만 값 있음 |
| judged_stage / predicted_stage | ResponseStage | 필수 | 생성 당시 current_stage 복사 / P5·0~5h 평균풍속 기준 판정 |
| created_at | DateTime | 필수 | 생성시각 |
| proposal_stale | boolean | 필수 | 현재 사건 상황과 이 버전이 불일치하는지 표시 |
| items | ProposalItem[] | 필수 | 발동하고 근거가 확보된 항목만, 최대 14개 |

official_stage는 두지 않습니다. 예측 단계가 현재 판정보다 높으면 격상 대비 안내이며 공식 발령으로 표시하지 않습니다. items 예시는 형식 설명용 일부 항목입니다.

**DTO Example**

```json
{
  "id": "71",
  "incident_id": "100",
  "prediction_id": "41",
  "perimeter_version": 1,
  "seq": 1,
  "version": "v1.1",
  "trigger": "PREDICTION",
  "correction_id": null,
  "judged_stage": "STAGE1",
  "predicted_stage": "STAGE1",
  "created_at": "2025-03-22T12:05:00+09:00",
  "proposal_stale": false,
  "items": [
    {
      "id": "72",
      "item_code": "S1",
      "axis": "SUPPRESSION",
      "categories": [
        "STANDBY"
      ],
      "text": "현재 단계 기준을 유지합니다 [1].",
      "detail": {
        "status": "maintain",
        "finding": "현재 기준과 P5 기준 단계 유지",
        "facts": {
          "now": {
            "stage": "1단계",
            "area_ha": 8,
            "wind_ms": 2.1,
            "facility": 12
          },
          "p5": {
            "stage": "1단계",
            "area_ha": 9,
            "wind_ms": 2.6,
            "facility": 12
          }
        },
        "targets": [],
        "categories": [
          "대기"
        ],
        "withheld": [
          "진화예상시간: MVP 제외"
        ],
        "rule_basis": [
          "SM-p073"
        ]
      },
      "citations": [
        {
          "id": "81",
          "k": 1,
          "chunk_id": "91",
          "chunk_code": "SM-p073-01",
          "doc_name": "시연 근거 문서",
          "page": 73,
          "section": "대응단계 판단기준"
        }
      ],
      "change_type": "UNCHANGED"
    }
  ]
}
```

<a id="dto-correctionchange"></a>

### CorrectionChange


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| target | village / deployed / incident / situation | 필수 | API 제안 구분 |
| target_id | ID | village일 때 필수 | villages.id, 다른 target은 생략 |
| item | string | 필수 | village: evacuated·evac_order, deployed: helicopter.total 등 Situation의 허용 경로, incident: alert_level, situation: disaster_sms_sent |
| before / after | boolean 또는 integer 또는 AlertLevel | 필수 | 허용 항목과 동일 타입. 서버 현재값과 before 대조 |

current_stage·official_stage·기상·진화예상시간·부상·고립·자원 개별 상태는 정정 허용 목록에 없습니다.

**DTO Example**

```json
{
  "target": "village",
  "target_id": "1024",
  "item": "evacuated",
  "before": false,
  "after": true
}
```

<a id="dto-correction"></a>

### Correction


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / incident_id / user_id | ID | 필수 | 정정 / 산불 / 확인 계정 |
| input_text | string | 필수 | 확인한 입력 원문 |
| changes | CorrectionChange[] | 필수 | 1개 이상. 입력 1건에 여러 변경 가능 |
| created_at | DateTime | 필수 | 확인·저장시각 |

**DTO Example**

```json
{
  "id": "101",
  "incident_id": "100",
  "user_id": "1",
  "input_text": "시연리 대피 완료",
  "changes": [
    {
      "target": "village",
      "target_id": "1024",
      "item": "evacuated",
      "before": false,
      "after": true
    }
  ],
  "created_at": "2025-03-22T12:20:00+09:00"
}
```

<a id="dto-correctionpreview"></a>

### CorrectionPreview


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| type | string | 필수 | CORRECTION_PREVIEW |
| confirmation_token | string | 필수 | 서버가 입력·변경안·사용자·산불·revision에 결합한 토큰 제안 |
| expires_at | DateTime | 필수 | 10분 제안, 팀 확인 필요 |
| situation_revision | string | 필수 | 확인안 생성 당시 버전 |
| input_text | string | 필수 | 원문 |
| changes | CorrectionChange[] | 필수 | 확인할 변경안 |

확인안을 받는 것만으로 DB 상황·정정 이력은 바뀌지 않습니다. 취소 시 저장 API를 호출하지 않습니다.

**DTO Example**

```json
{
  "type": "CORRECTION_PREVIEW",
  "confirmation_token": "EXAMPLE_PREVIEW_TOKEN",
  "expires_at": "2025-03-22T12:30:00+09:00",
  "situation_revision": "rev-example-1",
  "input_text": "시연리 대피 완료",
  "changes": [
    {
      "target": "village",
      "target_id": "1024",
      "item": "evacuated",
      "before": false,
      "after": true
    }
  ]
}
```

<a id="dto-qamessage"></a>

### QaMessage


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / incident_id / user_id | ID | 필수 | 질의응답 / 산불 / 질문 계정 |
| proposal_id | ID 또는 null | 필수·null 허용 | 질문 당시 보고 있던 제안, 예측 전에는 null |
| item_code | string 또는 null | 필수·null 허용 | 지정 항목 |
| question / answer | string | 필수 | 질문 / 답변, 답변의 [k]는 citations로 연결 |
| citations | CitationRef[] | 필수 | 근거 부족 시 [] 가능, 추정 답변 금지 |
| created_at | DateTime | 필수 | 저장시각 |

질문·답변 1건이 1행입니다. 이전 질문을 자동 문맥으로 이어 받지 않습니다. 상황은 변경하지 않습니다.

**DTO Example**

```json
{
  "id": "111",
  "incident_id": "100",
  "user_id": "1",
  "proposal_id": "71",
  "item_code": "S1",
  "question": "이 단계의 판단 근거는 무엇인가요?",
  "answer": "판단 기준은 연결된 근거에서 확인할 수 있습니다 [1].",
  "citations": [
    {
      "id": "81",
      "k": 1,
      "chunk_id": "91",
      "chunk_code": "SM-p073-01",
      "doc_name": "시연 근거 문서",
      "page": 73,
      "section": "대응단계 판단기준"
    }
  ],
  "created_at": "2025-03-22T12:10:00+09:00"
}
```

<a id="dto-risk"></a>

### Risk


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| prediction_id | ID | 필수 | 기준 예측 |
| status | READY / PARTIAL / UNAVAILABLE | 필수 | 23개 전부 계산 / 일부 제외 / 산출 불가, API 제안 |
| score | number 또는 null | 필수·null 허용 | R 1.00~5.00, 산출 불가면 null |
| grade | string 또는 null | 필수·null 허용 | 표시 등급 구간 미확정. 합의 전 null |
| axes | object | 필수 | terrain·fuel·weather·infrastructure 각각 number 또는 null. 축별 가용 변수의 가중평균 제안 |
| included_weight_sum | number | 필수 | 실제로 계산에 포함한 CSV 원가중치 합 |
| factors | RiskFactor[] | 필수 | 실구현 시 23개 항목 모두 반환, 미계산 항목도 사유와 함께 |
| method_version | string 또는 null | 필수·null 허용 | 계산 기준 판, 저장 방식과 함께 확정 |
| unavailable_reasons | string[] | 필수 | 미산출·부분산출 이유 |

이 예시의 빈 factors는 미구현·미합의 상태 예시입니다. 실제 완료 응답은 누락 변수를 포함한 23개를 반환해야 합니다. 실제 점수·등급·저장 완료를 보장하는 명세가 아닙니다(Q-01).

**DTO Example**

```json
{
  "prediction_id": "41",
  "status": "UNAVAILABLE",
  "score": null,
  "grade": null,
  "axes": {
    "terrain": null,
    "fuel": null,
    "weather": null,
    "infrastructure": null
  },
  "included_weight_sum": 0,
  "factors": [],
  "method_version": null,
  "unavailable_reasons": [
    "위험도 저장·계산 기준 합의 필요"
  ]
}
```

<a id="dto-riskfactor"></a>

### RiskFactor


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| code / name / axis | string | 필수 | 변수 코드·한글 이름·4축 중 하나. 7절 목록 |
| raw_value | number 또는 string 또는 null | 필수·null 허용 | 원값 |
| grade | integer 또는 null | 필수·null 허용 | 1~5, 산출 불가면 null |
| weight | number | 필수 | CSV 원가중치 |
| included | boolean | 필수 | 이번 계산에 실제 포함 여부 |
| contribution | number 또는 null | 필수·null 허용 | weight × grade ÷ included_weight_sum, 분모가 0이면 null |
| reason | string 또는 null | 필수·null 허용 | 미포함·대체값 사유 |
| source / value_at | string 또는 null / DateTime 또는 null | 필수·null 허용 | 자료·대체 관측 출처와 대상시각 |

**DTO Example**

```json
{
  "code": "wind_speed",
  "name": "풍속",
  "axis": "weather",
  "raw_value": null,
  "grade": null,
  "weight": 0.1,
  "included": false,
  "contribution": null,
  "reason": "등급 판정 시각 미확정",
  "source": null,
  "value_at": null
}
```

<a id="dto-layer"></a>

### Layer


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / name | string | 필수 | 아래 고정 레이어 ID / 표시명 |
| entity_type | string | 필수 | admin_area / village / facility / map_feature |
| available | boolean | 필수 | 실제 적재·조회 가능 여부 |
| source / reference_date | string 또는 null | 필수·null 허용 | 서비스 코드의 출처 메타데이터. DB 컬럼 추가 아님 |
| attribution | string 또는 null | 필수·null 허용 | 지도 출처 표시문 |

**DTO Example**

```json
{
  "id": "villages",
  "name": "법정리",
  "entity_type": "village",
  "available": true,
  "source": "주소정보누리집",
  "reference_date": "2026-10-09",
  "attribution": null
}
```

<a id="dto-eventlog"></a>

### EventLog


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id / created_at | ID / DateTime | 필수 | 로그 PK / 발생시각 |
| user_id / incident_id | ID 또는 null | 필수·null 허용 | 시스템 이벤트는 user_id null, 인증·계정 관리는 incident_id null |
| action | string | 필수 | 행위 코드 최대 50자 |
| details | object 또는 null | 필수·null 허용 | 행위별 사실. 개인정보·비밀번호·토큰 제외 |

**DTO Example**

```json
{
  "id": "121",
  "created_at": "2025-03-22T11:55:00+09:00",
  "user_id": "3",
  "incident_id": "100",
  "action": "PERIMETER_REPORTED",
  "details": {
    "perimeter_id": "21",
    "version_no": 1
  }
}
```

<a id="index"></a>

## 5. API 전체 목록

총 **46개**입니다. 기존 42개 API의 기능 구분을 유지하고, 최신 DB에 정의된 기지 관리 `BASE-01~04` 4개를 추가했습니다. **모든 경로·DTO는 백엔드와 검토할 API 설계 초안**이며 RISK-01은 저장·계산 합의가 필요한 조건부 API입니다.


| API ID | 기능 | Method | Endpoint | 권한 |
| --- | --- | --- | --- | --- |
| [AUTH-01](#auth-01) | 로그인 | POST | `/api/v1/auth/login` | 비로그인 |
| [AUTH-02](#auth-02) | 내 계정·권한 조회 | GET | `/api/v1/auth/me` | COMMANDER, VIEWER, REPORTER, ADMIN |
| [AUTH-03](#auth-03) | 로그아웃 기록 | POST | `/api/v1/auth/logout` | COMMANDER, VIEWER, REPORTER, ADMIN |
| [INC-01](#inc-01) | 산불 목록 | GET | `/api/v1/incidents` | COMMANDER, VIEWER, REPORTER |
| [INC-02](#inc-02) | 새 산불 보고 | POST | `/api/v1/incidents` | REPORTER |
| [INC-03](#inc-03) | 산불 현황 상세 | GET | `/api/v1/incidents/{incident_id}` | COMMANDER, VIEWER, REPORTER |
| [INC-04](#inc-04) | 산불 종료 | POST | `/api/v1/incidents/{incident_id}/end` | COMMANDER |
| [PER-01](#per-01) | 화선 버전 목록 | GET | `/api/v1/incidents/{incident_id}/perimeters` | COMMANDER, VIEWER, REPORTER |
| [PER-02](#per-02) | 실측 화선 보고 | POST | `/api/v1/incidents/{incident_id}/perimeters` | REPORTER |
| [PER-03](#per-03) | 현재 실측 화선 정정 | POST | `/api/v1/incidents/{incident_id}/perimeters/{perimeter_id}/corrections` | REPORTER |
| [PRED-01](#pred-01) | 확산 예측·위험도·제안 생성 요청 | POST | `/api/v1/incidents/{incident_id}/predictions` | COMMANDER |
| [PRED-02](#pred-02) | 성공 예측 상세 | GET | `/api/v1/incidents/{incident_id}/predictions/{prediction_id}` | COMMANDER, VIEWER |
| [PRED-03](#pred-03) | 시간별 누적 확산 도형 | GET | `/api/v1/incidents/{incident_id}/predictions/{prediction_id}/slices` | COMMANDER, VIEWER |
| [PRED-04](#pred-04) | 예측 사용 기상 | GET | `/api/v1/incidents/{incident_id}/predictions/{prediction_id}/weather` | COMMANDER, VIEWER |
| [PRED-05](#pred-05) | 계산 작업 조회 | GET | `/api/v1/incidents/{incident_id}/operations/{operation_id}` | COMMANDER |
| [RISK-01](#risk-01) | P5 위험도 조회 | GET | `/api/v1/incidents/{incident_id}/predictions/{prediction_id}/risk` | COMMANDER, VIEWER |
| [MAP-01](#map-01) | 지도 레이어 목록 | GET | `/api/v1/map/layers` | COMMANDER, VIEWER, REPORTER |
| [MAP-02](#map-02) | 지도 레이어 도형 | GET | `/api/v1/map/layers/{layer_id}/features` | COMMANDER, VIEWER, REPORTER |
| [MAP-03](#map-03) | 지도 대상 상세 | GET | `/api/v1/map/entities/{entity_type}/{entity_id}` | COMMANDER, VIEWER, REPORTER |
| [MAP-04](#map-04) | 행정구역 검색 | GET | `/api/v1/admin-areas` | COMMANDER, VIEWER, REPORTER, ADMIN |
| [RES-01](#res-01) | 진화자원 목록 | GET | `/api/v1/resources` | COMMANDER, VIEWER, ADMIN |
| [RES-02](#res-02) | 진화자원 상태 집계 | GET | `/api/v1/resources/summary` | COMMANDER, VIEWER, ADMIN |
| [RES-03](#res-03) | 진화자원 등록 | POST | `/api/v1/admin/resources` | ADMIN |
| [RES-04](#res-04) | 진화자원 수정 | PATCH | `/api/v1/admin/resources/{resource_id}` | ADMIN |
| [RES-05](#res-05) | 진화자원 삭제 | DELETE | `/api/v1/admin/resources/{resource_id}` | ADMIN |
| [BASE-01](#base-01) | 진화자원 기지 목록 | GET | `/api/v1/resource-bases` | COMMANDER, VIEWER, ADMIN |
| [BASE-02](#base-02) | 기지 등록 | POST | `/api/v1/admin/resource-bases` | ADMIN |
| [BASE-03](#base-03) | 기지 수정 | PATCH | `/api/v1/admin/resource-bases/{base_id}` | ADMIN |
| [BASE-04](#base-04) | 기지 삭제 | DELETE | `/api/v1/admin/resource-bases/{base_id}` | ADMIN |
| [PROP-01](#prop-01) | 제안서 버전 목록 | GET | `/api/v1/incidents/{incident_id}/proposals` | COMMANDER |
| [PROP-02](#prop-02) | 최신 제안서 | GET | `/api/v1/incidents/{incident_id}/proposals/latest` | COMMANDER |
| [PROP-03](#prop-03) | 특정 버전 제안서 | GET | `/api/v1/incidents/{incident_id}/proposals/{proposal_id}` | COMMANDER |
| [PROP-04](#prop-04) | 직전 버전 비교 | GET | `/api/v1/incidents/{incident_id}/proposals/{proposal_id}/diff` | COMMANDER |
| [PROP-05](#prop-05) | 정정 미반영 제안 재생성 | POST | `/api/v1/incidents/{incident_id}/proposal-regenerations` | COMMANDER |
| [EVD-01](#evd-01) | 인용 근거 본문 | GET | `/api/v1/citations/{citation_id}` | COMMANDER |
| [AI-01](#ai-01) | 질의응답 또는 정정 확인안 | POST | `/api/v1/incidents/{incident_id}/assistant/messages` | COMMANDER |
| [AI-02](#ai-02) | 질의응답 이력 | GET | `/api/v1/incidents/{incident_id}/assistant/messages` | COMMANDER |
| [COR-01](#cor-01) | 확인한 상황 정정 저장 | POST | `/api/v1/incidents/{incident_id}/corrections` | COMMANDER |
| [COR-02](#cor-02) | 확인된 상황 정정 이력 | GET | `/api/v1/incidents/{incident_id}/corrections` | COMMANDER |
| [LOG-01](#log-01) | 이벤트 로그 | GET | `/api/v1/event-logs` | COMMANDER, ADMIN |
| [USR-01](#usr-01) | 계정 목록 | GET | `/api/v1/admin/users` | ADMIN |
| [USR-02](#usr-02) | 계정 발급 | POST | `/api/v1/admin/users` | ADMIN |
| [USR-03](#usr-03) | 계정 정보·권한 변경 | PATCH | `/api/v1/admin/users/{user_id}` | ADMIN |
| [USR-04](#usr-04) | 계정 회수 | DELETE | `/api/v1/admin/users/{user_id}` | ADMIN |
| [USR-05](#usr-05) | 비밀번호 재발급 | POST | `/api/v1/admin/users/{user_id}/password-reset` | ADMIN |
| [USR-06](#usr-06) | 로그인 잠금 해제 | POST | `/api/v1/admin/users/{user_id}/unlock` | ADMIN |

<a id="endpoints"></a>

## 6. API 상세

공통 헤더·오류·중복키 규칙은 2절을 적용합니다. 별도 Query가 없으면 Query parameter는 없습니다. 경로의 `{incident_id}`·`{prediction_id}` 등 PK는 모두 ID이고, `{operation_id}`는 불투명 문자열, `{layer_id}`·`{entity_type}`은 해당 절의 코드입니다.

<a id="auth-01"></a>

### AUTH-01 · 로그인


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/auth/login` |
| 권한 | 비로그인 |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.1 계정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

없음.

**Request Body**

login_id: string(1~50), password: 비어 있지 않은 string, role: UserRole; 모두 필수

```json
{
  "login_id": "cmd01",
  "password": "EXAMPLE_PASSWORD",
  "role": "COMMANDER"
}
```

**Response**

`data`: AuthToken

```json
{
  "data": {
    "access_token": "EXAMPLE_ACCESS_TOKEN",
    "token_type": "bearer",
    "expires_in": 1800,
    "user": {
      "id": "1",
      "login_id": "cmd01",
      "name": "시연 사용자",
      "organization": "시연 지휘본부",
      "role": "COMMANDER",
      "admin_area_id": "10"
    }
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 회원가입은 없습니다. role은 선택 화면 확인값일 뿐이며 DB 역할과 일치해야 합니다. 회수 계정·실패 5회 이상 계정은 거부하고 성공 시 실패 횟수를 초기화합니다. JWT 30분은 제안값입니다.

**주요 오류:** 401 INVALID_CREDENTIALS, 403 ACCOUNT_UNAVAILABLE / ROLE_MISMATCH

<a id="auth-02"></a>

### AUTH-02 · 내 계정·권한 조회


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/auth/me` |
| 권한 | COMMANDER, VIEWER, REPORTER, ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.1 계정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: User

```json
{
  "data": {
    "id": "1",
    "login_id": "cmd01",
    "name": "시연 사용자",
    "organization": "시연 지휘본부",
    "role": "COMMANDER",
    "admin_area_id": "10"
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** JWT뿐 아니라 매 요청 users의 현재 역할·관할·회수·잠금을 확인합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="auth-03"></a>

### AUTH-03 · 로그아웃 기록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/auth/logout` |
| 권한 | COMMANDER, VIEWER, REPORTER, ADMIN |
| 성공 상태 | 204 |
| 근거 | [DB-설계.md · 2.1 계정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

없음.

**Request Body**

없음.

**Response**

본문 없음.

**처리 규칙:** 이벤트를 기록하고 프론트는 토큰과 사용자 상태를 삭제합니다. 서버 JWT 즉시 폐기는 현 DB 구조로 확정되지 않았습니다(Q-09).

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="inc-01"></a>

### INC-01 · 산불 목록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents` |
| 권한 | COMMANDER, VIEWER, REPORTER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.2](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| status | IncidentStatus | 선택 | 미지정이면 전체 진행상태 |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<IncidentSummary>

```json
{
  "data": {
    "items": [
      {
        "id": "100",
        "name": "의성군 안평면 산불 (2025-03-22)",
        "status": "IN_PROGRESS",
        "admin_area_id": "11",
        "reported_at": "2025-03-22T11:25:00+09:00",
        "current_stage": "STAGE1",
        "current_perimeter_version": 1,
        "needs_reprediction": false,
        "proposal_stale": false
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** users의 관할 시군구 내 산불만, reported_at DESC, id DESC로 조회합니다. 페이지 이동 중 변동 시 중복을 id로 제거합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="inc-02"></a>

### INC-02 · 새 산불 보고


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/incidents` |
| 권한 | REPORTER |
| 성공 상태 | 201 |
| 근거 | [DB-설계.md · 2.2](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**추가 Header:** `Idempotency-Key` 필수.

**Path parameter:** 없음

**Query parameter**

없음.

**Request Body**

ignition_point: Point, address: string(1~200), reported_at: DateTime, reporter_name: string(1~50), reporter_phone: string(1~20), alert_level: AlertLevel은 필수. started_at·report_text·reporter_org는 생략/null 허용. initial_perimeter: {geometry: MultiPolygon, observed_at: DateTime} 선택

```json
{
  "ignition_point": {
    "type": "Point",
    "coordinates": [
      128.6,
      36.37
    ]
  },
  "address": "시연 발생 장소",
  "started_at": null,
  "reported_at": "2025-03-22T11:25:00+09:00",
  "report_text": null,
  "reporter_org": null,
  "reporter_name": "미상",
  "reporter_phone": "미상",
  "alert_level": "ATTENTION"
}
```

**Response**

`data`: Incident (REPORTER용 필드 제한 적용)

```json
{
  "data": {
    "id": "100",
    "name": "의성군 안평면 산불 (2025-03-22)",
    "ignition_point": {
      "type": "Point",
      "coordinates": [
        128.6,
        36.37
      ]
    },
    "address": "시연 발생 장소",
    "admin_area_id": "11",
    "started_at": "2025-03-22T11:24:00+09:00",
    "reported_at": "2025-03-22T11:25:00+09:00",
    "report_text": "시연 산불 신고",
    "reporter_org": null,
    "reporter_name": "미상",
    "reporter_phone": "미상",
    "status": "IN_PROGRESS",
    "current_stage": "STAGE1",
    "alert_level": "ATTENTION",
    "reported_by": "3",
    "created_at": "2025-03-22T11:25:00+09:00",
    "ended_at": null,
    "ended_by": null,
    "current_perimeter_id": "20",
    "current_perimeter_version": 0
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 발화점으로 읍면동·관할을 서버가 계산합니다. 산불과 화선 v0을 같은 트랜잭션에서 저장합니다. initial_perimeter가 있으면 v1도 함께 저장합니다. status·current_stage·reported_by·기관별 접수 배열은 입력받지 않습니다. current_stage 판정 입력이 부족한 경우 처리는 Q-02에서 확정합니다.

**주요 오류:** 422 INVALID_GEOMETRY / OUTSIDE_JURISDICTION / STAGE_INPUT_UNAVAILABLE

<a id="inc-03"></a>

### INC-03 · 산불 현황 상세


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}` |
| 권한 | COMMANDER, VIEWER, REPORTER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.2·4.2](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: Incident (예시는 COMMANDER)

```json
{
  "data": {
    "id": "100",
    "name": "의성군 안평면 산불 (2025-03-22)",
    "ignition_point": {
      "type": "Point",
      "coordinates": [
        128.6,
        36.37
      ]
    },
    "address": "시연 발생 장소",
    "admin_area_id": "11",
    "started_at": "2025-03-22T11:24:00+09:00",
    "reported_at": "2025-03-22T11:25:00+09:00",
    "report_text": "시연 산불 신고",
    "reporter_org": null,
    "reporter_name": "미상",
    "reporter_phone": "미상",
    "status": "IN_PROGRESS",
    "current_stage": "STAGE1",
    "alert_level": "ATTENTION",
    "situation": {
      "villages": {
        "1024": {
          "evacuated": false,
          "evac_order": true
        }
      },
      "deployed": {
        "helicopter": {
          "total": 3,
          "forest": 2,
          "local": 1
        },
        "vehicle": {
          "total": 12
        },
        "crew": {
          "total": 85
        }
      },
      "disaster_sms_sent": false
    },
    "reported_by": "3",
    "created_at": "2025-03-22T11:25:00+09:00",
    "ended_at": null,
    "ended_by": null,
    "current_perimeter_id": "21",
    "current_perimeter_version": 1,
    "latest_prediction_id": "41",
    "latest_proposal_id": "71",
    "needs_reprediction": false,
    "proposal_stale": false,
    "situation_revision": "rev-example-1"
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 산불의 현재 상태와 최신 성공 결과 참조를 일관되게 반환합니다. REPORTER 응답은 보고·화선 정보로 제한하는 안입니다. 성명·연락처 공개 범위는 Q-08에서 확정합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="inc-04"></a>

### INC-04 · 산불 종료


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/incidents/{incident_id}/end` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.2·2.3·2.5](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID

**Query parameter**

없음.

**Request Body**

expected_perimeter_id: ID 필수(현재 화선과 대조하는 API 제안)

```json
{
  "expected_perimeter_id": "21"
}
```

**Response**

`data`: Incident

```json
{
  "data": {
    "id": "100",
    "name": "의성군 안평면 산불 (2025-03-22)",
    "ignition_point": {
      "type": "Point",
      "coordinates": [
        128.6,
        36.37
      ]
    },
    "address": "시연 발생 장소",
    "admin_area_id": "11",
    "started_at": "2025-03-22T11:24:00+09:00",
    "reported_at": "2025-03-22T11:25:00+09:00",
    "report_text": "시연 산불 신고",
    "reporter_org": null,
    "reporter_name": "미상",
    "reporter_phone": "미상",
    "status": "ENDED",
    "current_stage": "STAGE1",
    "alert_level": "ATTENTION",
    "situation": {
      "villages": {
        "1024": {
          "evacuated": false,
          "evac_order": true
        }
      },
      "deployed": {
        "helicopter": {
          "total": 3,
          "forest": 2,
          "local": 1
        },
        "vehicle": {
          "total": 12
        },
        "crew": {
          "total": 85
        }
      },
      "disaster_sms_sent": false
    },
    "reported_by": "3",
    "created_at": "2025-03-22T11:25:00+09:00",
    "ended_at": "2025-03-22T18:00:00+09:00",
    "ended_by": "1",
    "current_perimeter_id": "21",
    "current_perimeter_version": 1,
    "latest_prediction_id": "41",
    "latest_proposal_id": "71",
    "needs_reprediction": false,
    "proposal_stale": false,
    "situation_revision": "rev-example-1"
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 종료 후 새 화선 보고·정정, 예측, 상황 정정, 제안 재생성을 막습니다. 저장 중인 예측 작업과 같은 산불별 잠금으로 순서를 보장하고 종료 전 대기 작업은 커밋 직전 재검사합니다. 종료 후 기존 제안 조회와 질문은 허용합니다.

**주요 오류:** 409 INCIDENT_ENDED / RESOURCE_BUSY / STALE_PERIMETER

<a id="per-01"></a>

### PER-01 · 화선 버전 목록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/perimeters` |
| 권한 | COMMANDER, VIEWER, REPORTER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.2](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<Perimeter>

```json
{
  "data": {
    "items": [
      {
        "id": "21",
        "incident_id": "100",
        "version_no": 1,
        "geometry": {
          "type": "MultiPolygon",
          "coordinates": [
            [
              [
                [
                  128.6,
                  36.37
                ],
                [
                  128.602,
                  36.37
                ],
                [
                  128.602,
                  36.372
                ],
                [
                  128.6,
                  36.37
                ]
              ]
            ]
          ]
        },
        "observed_at": "2025-03-22T11:55:00+09:00",
        "reported_by": "3",
        "created_at": "2025-03-22T11:55:00+09:00",
        "corrected_at": null,
        "area_ha": 8.0
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** v0과 정정된 화선을 포함해 version_no DESC로 반환합니다. 정정된 버전도 이력에서 숨기지 않습니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="per-02"></a>

### PER-02 · 실측 화선 보고


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/incidents/{incident_id}/perimeters` |
| 권한 | REPORTER |
| 성공 상태 | 201 |
| 근거 | [DB-설계.md · 2.2](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**추가 Header:** `Idempotency-Key` 필수.

**Path parameter:** `incident_id`: ID

**Query parameter**

없음.

**Request Body**

geometry: MultiPolygon, observed_at: DateTime, expected_perimeter_id: ID 모두 필수

```json
{
  "geometry": {
    "type": "MultiPolygon",
    "coordinates": [
      [
        [
          [
            128.6,
            36.37
          ],
          [
            128.602,
            36.37
          ],
          [
            128.602,
            36.372
          ],
          [
            128.6,
            36.37
          ]
        ]
      ]
    ]
  },
  "observed_at": "2025-03-22T11:55:00+09:00",
  "expected_perimeter_id": "20"
}
```

**Response**

`data`: Perimeter

```json
{
  "data": {
    "id": "21",
    "incident_id": "100",
    "version_no": 1,
    "geometry": {
      "type": "MultiPolygon",
      "coordinates": [
        [
          [
            [
              128.6,
              36.37
            ],
            [
              128.602,
              36.37
            ],
            [
              128.602,
              36.372
            ],
            [
              128.6,
              36.37
            ]
          ]
        ]
      ]
    },
    "observed_at": "2025-03-22T11:55:00+09:00",
    "reported_by": "3",
    "created_at": "2025-03-22T11:55:00+09:00",
    "corrected_at": null,
    "area_ha": 8.0
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 다음 version_no로 INSERT하고 같은 트랜잭션에서 current_stage를 다시 판정합니다. 예측은 자동 실행하지 않습니다. 구멍·자기 교차·빈 도형을 검증하며 새 실측 화선은 이전 예측 범위를 포함할 의무가 없습니다. 관측시각 역전 처리와 P5 미확보 판정은 Q-02에서 확정합니다.

**주요 오류:** 409 INCIDENT_ENDED / STALE_PERIMETER, 422 INVALID_GEOMETRY / STAGE_INPUT_UNAVAILABLE

<a id="per-03"></a>

### PER-03 · 현재 실측 화선 정정


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/incidents/{incident_id}/perimeters/{perimeter_id}/corrections` |
| 권한 | REPORTER |
| 성공 상태 | 201 |
| 근거 | [DB-설계.md · 2.2 화선 정정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**추가 Header:** `Idempotency-Key` 필수.

**Path parameter:** `incident_id`: ID, `perimeter_id`: ID

**Query parameter**

없음.

**Request Body**

geometry: MultiPolygon, observed_at: DateTime 모두 필수

```json
{
  "geometry": {
    "type": "MultiPolygon",
    "coordinates": [
      [
        [
          [
            128.6,
            36.37
          ],
          [
            128.602,
            36.37
          ],
          [
            128.602,
            36.372
          ],
          [
            128.6,
            36.37
          ]
        ]
      ]
    ]
  },
  "observed_at": "2025-03-22T12:05:00+09:00"
}
```

**Response**

`data`: Perimeter (정정 후 새 버전)

```json
{
  "data": {
    "id": "22",
    "incident_id": "100",
    "version_no": 2,
    "geometry": {
      "type": "MultiPolygon",
      "coordinates": [
        [
          [
            [
              128.6,
              36.37
            ],
            [
              128.602,
              36.37
            ],
            [
              128.602,
              36.372
            ],
            [
              128.6,
              36.37
            ]
          ]
        ]
      ]
    },
    "observed_at": "2025-03-22T12:05:00+09:00",
    "reported_by": "3",
    "created_at": "2025-03-22T12:05:00+09:00",
    "corrected_at": null,
    "area_ha": 8.0
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** URL의 perimeter_id가 현재 유효한 v1 이상이어야 합니다. 기존 행 corrected_at과 다음 번호 새 행·current_stage 갱신을 한 트랜잭션으로 처리합니다. v0 정정은 금지합니다. 과거 예측과 제안은 보존하고 이력에서 기준 화선 정정 여부를 표시합니다.

**주요 오류:** 409 INCIDENT_ENDED / STALE_PERIMETER, 422 V0_NOT_CORRECTABLE

<a id="pred-01"></a>

### PRED-01 · 확산 예측·위험도·제안 생성 요청


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/incidents/{incident_id}/predictions` |
| 권한 | COMMANDER |
| 성공 상태 | 202 |
| 근거 | [DB-설계.md · 2.3·2.4](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**추가 Header:** `Idempotency-Key` 필수.

**Path parameter:** `incident_id`: ID

**Query parameter**

없음.

**Request Body**

expected_perimeter_id: ID 필수

```json
{
  "expected_perimeter_id": "21"
}
```

**Response**

`data`: Operation

```json
{
  "data": {
    "id": "op-example",
    "kind": "PREDICTION",
    "status": "RUNNING",
    "stage": "PROPOSAL_GENERATION",
    "result": null,
    "error": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 최신 유효 화선만 대상으로 하며 과거 화선 지정 실행은 허용하지 않습니다. 서버가 기상 지점을 선택하고 base_at=observed_at으로 고정합니다. 예측→P1~P8→위험도→규칙→근거→제안 계산 후 최종 결과를 한 트랜잭션으로 저장합니다. 실패 시 직전 예측·제안이 유지됩니다. 계산 중 새 화선·정정 발생 시 오래된 결과를 최신으로 커밋하지 않도록 재검사합니다. 위험도 미산출을 전체 실패로 볼지는 Q-01, 작업 저장소는 Q-05에서 확정합니다.

**주요 오류:** 409 INCIDENT_ENDED / PREDICTION_RUNNING / STALE_PERIMETER, 503 WEATHER_UNAVAILABLE

<a id="pred-02"></a>

### PRED-02 · 성공 예측 상세


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/predictions/{prediction_id}` |
| 권한 | COMMANDER, VIEWER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.3](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID, `prediction_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: Prediction

```json
{
  "data": {
    "id": "41",
    "incident_id": "100",
    "perimeter_id": "21",
    "perimeter_version": 1,
    "base_at": "2025-03-22T11:55:00+09:00",
    "requested_by": "1",
    "created_at": "2025-03-22T12:05:00+09:00",
    "slices": [
      {
        "hour": 1,
        "at": "2025-03-22T12:55:00+09:00",
        "area_ha": 21
      },
      {
        "hour": 2,
        "at": "2025-03-22T13:55:00+09:00",
        "area_ha": 40
      },
      {
        "hour": 3,
        "at": "2025-03-22T14:55:00+09:00",
        "area_ha": 68
      },
      {
        "hour": 4,
        "at": "2025-03-22T15:55:00+09:00",
        "area_ha": 105
      },
      {
        "hour": 5,
        "at": "2025-03-22T16:55:00+09:00",
        "area_ha": 150
      },
      {
        "hour": 6,
        "at": "2025-03-22T17:55:00+09:00",
        "area_ha": 190
      },
      {
        "hour": 7,
        "at": "2025-03-22T18:55:00+09:00",
        "area_ha": 228
      },
      {
        "hour": 8,
        "at": "2025-03-22T19:55:00+09:00",
        "area_ha": 260
      }
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 예측 PK가 해당 산불에 속하는지 확인합니다. 진행·실패 중인 작업은 이 API 결과로 나타나지 않습니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="pred-03"></a>

### PRED-03 · 시간별 누적 확산 도형


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/predictions/{prediction_id}/slices` |
| 권한 | COMMANDER, VIEWER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · prediction_slices](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID, `prediction_id`: ID

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| hour | integer | 선택 | 1~8, 미지정이면 P1~P8 전체 |

**Request Body**

없음.

**Response**

`data`: FeatureCollection<SliceProperties>

```json
{
  "data": {
    "type": "FeatureCollection",
    "features": [
      {
        "type": "Feature",
        "id": "61",
        "geometry": {
          "type": "MultiPolygon",
          "coordinates": [
            [
              [
                [
                  128.6,
                  36.37
                ],
                [
                  128.602,
                  36.37
                ],
                [
                  128.602,
                  36.372
                ],
                [
                  128.6,
                  36.37
                ]
              ]
            ]
          ]
        },
        "properties": {
          "prediction_id": "41",
          "hour": 5,
          "at": "2025-03-22T16:55:00+09:00",
          "area_ha": 150
        }
      }
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** GeoJSON properties는 prediction_id: ID, hour: integer, at: DateTime, area_ha: number입니다. 선택 hour는 1개, 미지정은 8개 반환합니다. 예시는 hour=5 요청입니다. 도형은 MultiPolygon, P(n)이 P(n-1)을 포함해야 합니다. P5=위험구역, P8=잠재 위험구역이며 지도에서 시간 순서로 재생합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="pred-04"></a>

### PRED-04 · 예측 사용 기상


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/predictions/{prediction_id}/weather` |
| 권한 | COMMANDER, VIEWER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · predictions.weather](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID, `prediction_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: Weather

```json
{
  "data": {
    "source": "AWS_OBS",
    "issued_at": null,
    "location": {
      "stn": "EXAMPLE_STATION"
    },
    "hours": [
      {
        "at": "2025-03-22T11:00:00+09:00",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T12:00:00+09:00",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T13:00:00+09:00",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T14:00:00+09:00",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T15:00:00+09:00",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T16:00:00+09:00",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T17:00:00+09:00",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T18:00:00+09:00",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T19:00:00+09:00",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      }
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 현재 날씨를 새로 수신한 결과가 아니라 해당 예측에 저장된 스냅샷을 반환합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="pred-05"></a>

### PRED-05 · 계산 작업 조회


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/operations/{operation_id}` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.3의 실행·실패 처리](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID, `operation_id`: string

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: Operation

```json
{
  "data": {
    "id": "op-example",
    "kind": "PREDICTION",
    "status": "RUNNING",
    "stage": "PROPOSAL_GENERATION",
    "result": null,
    "error": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 2초 간격 조회를 제안합니다. 완료는 SUCCEEDED/FAILED로 끝나며 HTTP 200 응답 안에서 작업 실패를 구분합니다. operation은 요청 사용자·산불에 결합하고 보존기간은 Q-05에서 확정합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="risk-01"></a>

### RISK-01 · P5 위험도 조회


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/predictions/{prediction_id}/risk` |
| 권한 | COMMANDER, VIEWER |
| 성공 상태 | 200 |
| 근거 | [위험도-정리.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/위험도-정리.md) · [DB-설계.md · 4.2](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID, `prediction_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: Risk

```json
{
  "data": {
    "prediction_id": "41",
    "status": "UNAVAILABLE",
    "score": null,
    "grade": null,
    "axes": {
      "terrain": null,
      "fuel": null,
      "weather": null,
      "infrastructure": null
    },
    "included_weight_sum": 0,
    "factors": [],
    "method_version": null,
    "unavailable_reasons": [
      "위험도 저장·계산 기준 합의 필요"
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 계산 방식·저장소가 합의된 뒤 구현하는 조건부 API입니다. 미산출을 0 또는 안전 등급으로 채우지 않습니다. 종료된 산불의 조회 허용 여부는 Q-01에서 결정하며, 구현 전 임의로 차단/허용을 확정하지 않습니다.

**주요 오류:** 409 RISK_POLICY_PENDING (종료 후 정책 미확정 시), 503 RISK_UNAVAILABLE (조회 기능 미구현 시)

<a id="map-01"></a>

### MAP-01 · 지도 레이어 목록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/map/layers` |
| 권한 | COMMANDER, VIEWER, REPORTER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.6](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) · [데이터-자원.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/데이터-자원.md) |

**Path parameter:** 없음

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: Layer[]

```json
{
  "data": [
    {
      "id": "villages",
      "name": "법정리",
      "entity_type": "village",
      "available": true,
      "source": "주소정보누리집",
      "reference_date": "2026-10-09",
      "attribution": null
    }
  ],
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 응답의 available은 실제 적재 여부로 판단합니다. 본문 예시는 레이어 1개이며 7절 레이어 ID 목록 전체 중 권한 있는 것을 반환합니다. 원자료 수집 완료와 DB 적재 완료를 구분합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="map-02"></a>

### MAP-02 · 지도 레이어 도형


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/map/layers/{layer_id}/features` |
| 권한 | COMMANDER, VIEWER, REPORTER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.6](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `layer_id`: string

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| bbox | string | 필수 | min_lon,min_lat,max_lon,max_lat, 4326 범위 |
| cursor | string | 선택 | 지도 전용 커서 |
| limit | integer | 선택 | 기본 500, 최대 1000 |

**Request Body**

없음.

**Response**

`data`: FeatureCollection<레이어별 속성> + next_cursor + has_more

```json
{
  "data": {
    "type": "FeatureCollection",
    "features": [
      {
        "type": "Feature",
        "id": "1024",
        "geometry": {
          "type": "MultiPolygon",
          "coordinates": [
            [
              [
                [
                  128.6,
                  36.37
                ],
                [
                  128.602,
                  36.37
                ],
                [
                  128.602,
                  36.372
                ],
                [
                  128.6,
                  36.37
                ]
              ]
            ]
          ]
        },
        "properties": {
          "id": "1024",
          "code": "4773036021",
          "admin_area_id": "11",
          "name": "시연리",
          "population": 150,
          "population_month": "2025-02-01",
          "pop65_est": 85,
          "pop0_14_est": 6,
          "houses_est": 30
        }
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** layer_id와 bbox를 허용 범위·관할로 제한합니다. ID는 문자열입니다. properties는 레이어에 따라 AdminArea/Village/Facility/MapFeature에서 geometry를 제외한 속성이며 label_point는 지도 라벨용으로 필요하면 포함합니다. has_more=true이면 다음 페이지를 모아 레이어를 구성합니다. 전체 데이터가 1000개라고 가정하지 않습니다.

**주요 오류:** 422 INVALID_BBOX / UNKNOWN_LAYER, 503 LAYER_UNAVAILABLE

<a id="map-03"></a>

### MAP-03 · 지도 대상 상세


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/map/entities/{entity_type}/{entity_id}` |
| 권한 | COMMANDER, VIEWER, REPORTER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.6](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) · [규칙-엔진-설계.md · S2·E5 targets](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/규칙-엔진-설계.md) |

**Path parameter:** `entity_type`: string, `entity_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: {entity_type: string, entity: Village 또는 Facility 또는 MapFeature 또는 ResourceBase 또는 AdminArea}

```json
{
  "data": {
    "entity_type": "village",
    "entity": {
      "id": "1024",
      "code": "4773036021",
      "admin_area_id": "11",
      "name": "시연리",
      "geometry": {
        "type": "MultiPolygon",
        "coordinates": [
          [
            [
              [
                128.6,
                36.37
              ],
              [
                128.602,
                36.37
              ],
              [
                128.602,
                36.372
              ],
              [
                128.6,
                36.37
              ]
            ]
          ]
        ]
      },
      "label_point": {
        "type": "Point",
        "coordinates": [
          128.6,
          36.37
        ]
      },
      "population": 150,
      "population_month": "2025-02-01",
      "pop65_est": 85,
      "pop0_14_est": 6,
      "houses_est": 30
    }
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** entity_type은 village / facility / map_feature / resource_base / admin_area입니다. 제안 targets와 연결합니다. REPORTER는 보고 지도용 허용 종류·범위만 조회하며 기지·분석 범위는 Q-08에서 합의합니다. 동적 노선 구간은 원본 map_feature ID 여러 개로 강조합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="map-04"></a>

### MAP-04 · 행정구역 검색


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/admin-areas` |
| 권한 | COMMANDER, VIEWER, REPORTER, ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · admin_areas](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| level | AdminLevel | 선택 | SIDO / SIGUNGU / EMD |
| parent_id | ID | 선택 | 상위 행정구역 |
| q | string | 선택 | 이름 검색, 1~50자 |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<AdminArea>

```json
{
  "data": {
    "items": [
      {
        "id": "11",
        "level": "EMD",
        "code": "4773036000",
        "name": "안평면",
        "parent_id": "10",
        "neighbor_ids": []
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 법정동코드 10자리를 숫자로 변환하지 않습니다. 관할 계정은 보고용 읍면과 필요한 상위 범위를, ADMIN은 계정·기지 관할 선택에 필요한 전국 범위를 조회하는 안입니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="res-01"></a>

### RES-01 · 진화자원 목록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/resources` |
| 권한 | COMMANDER, VIEWER, ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · resources·resource_bases](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| incident_id | ID | COMMANDER/VIEWER 필수 | 관할·응원·거리 판단 문맥, ADMIN은 생략 가능 |
| kind | ResourceKind | 선택 | 자원 종류 |
| status | ResourceStatus | 선택 | 자원 상태 |
| base_id | ID | 선택 | 소속 기지 |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<Resource>

```json
{
  "data": {
    "items": [
      {
        "id": "51",
        "kind": "HELICOPTER",
        "name": "DEMO-HELI-01",
        "base_id": "31",
        "status": "AVAILABLE",
        "attrs": {
          "model": "시연 기종",
          "capacity_l": 8000
        }
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 전체 보유·개별 상태와 산불 투입 수를 분리합니다. COMMANDER/VIEWER는 허용된 incident_id 문맥의 관할·응원 범위로 제한합니다. 실제 범위·전국 자원 열람 정책은 Q-08에서 확정합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="res-02"></a>

### RES-02 · 진화자원 상태 집계


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/resources/summary` |
| 권한 | COMMANDER, VIEWER, ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.6·L-10](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| incident_id | ID | COMMANDER/VIEWER 필수 | 관할·응원·거리 판단 문맥, ADMIN은 생략 가능 |
| kind | ResourceKind | 선택 | 자원 종류 |
| base_id | ID | 선택 | 기지별 제한 |

**Request Body**

없음.

**Response**

`data`: {bases: ResourceCount[], incident_deployed: object 또는 null}

```json
{
  "data": {
    "bases": [
      {
        "base_id": "31",
        "kind": "HELICOPTER",
        "available": 2,
        "deployed": 0,
        "maintenance": 1,
        "total": 3
      }
    ],
    "incident_deployed": {
      "helicopter": {
        "total": 3,
        "forest": 2,
        "local": 1
      },
      "vehicle": {
        "total": 12
      },
      "crew": {
        "total": 85
      }
    }
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 각 ResourceCount는 base_id: ID, kind: ResourceKind, available/deployed/maintenance/total: 0 이상 integer입니다. 상태 필터를 받지 않고 종류·기지·허용범위 전체 상태를 집계합니다. incident_deployed는 incident_id 미지정 시 null, 지정 시 Situation.deployed입니다. 보유 상태의 deployed를 특정 산불 투입 수로 해석하지 않습니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="res-03"></a>

### RES-03 · 진화자원 등록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/admin/resources` |
| 권한 | ADMIN |
| 성공 상태 | 201 |
| 근거 | [DB-설계.md · resources](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

없음.

**Request Body**

Resource의 id를 제외한 kind·name·base_id 필수, status 생략 시 AVAILABLE, attrs 생략 시 {}

```json
{
  "kind": "HELICOPTER",
  "name": "DEMO-HELI-01",
  "base_id": "31",
  "status": "AVAILABLE",
  "attrs": {
    "model": "시연 기종",
    "capacity_l": 8000
  }
}
```

**Response**

`data`: Resource

```json
{
  "data": {
    "id": "51",
    "kind": "HELICOPTER",
    "name": "DEMO-HELI-01",
    "base_id": "31",
    "status": "AVAILABLE",
    "attrs": {
      "model": "시연 기종",
      "capacity_l": 8000
    }
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 헬기·차량 1대씩 등록합니다. base_id 존재 확인, name 유일, capacity_l은 있으면 0 이상 숫자입니다. 등록·수정·삭제는 RESOURCE_UPDATED 이벤트로 기록합니다.

**주요 오류:** 409 DUPLICATE_RESOURCE_NAME, 422 INVALID_BASE

<a id="res-04"></a>

### RES-04 · 진화자원 수정


| 항목 | 내용 |
| --- | --- |
| Method·URL | `PATCH /api/v1/admin/resources/{resource_id}` |
| 권한 | ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · resources](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `resource_id`: ID

**Query parameter**

없음.

**Request Body**

kind·name·base_id·status·attrs 중 1개 이상. 타입·제약은 Resource 등록과 같고 null 불가

```json
{
  "status": "MAINTENANCE"
}
```

**Response**

`data`: Resource

```json
{
  "data": {
    "id": "51",
    "kind": "HELICOPTER",
    "name": "DEMO-HELI-01",
    "base_id": "31",
    "status": "MAINTENANCE",
    "attrs": {
      "model": "시연 기종",
      "capacity_l": 8000
    }
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** attrs를 보내면 객체 전체를 교체합니다. resources에 없는 org_type·관할·실시간 위치는 수정할 수 없습니다. 서버 동시 편집 보호 방식은 Q-09에서 확정합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="res-05"></a>

### RES-05 · 진화자원 삭제


| 항목 | 내용 |
| --- | --- |
| Method·URL | `DELETE /api/v1/admin/resources/{resource_id}` |
| 권한 | ADMIN |
| 성공 상태 | 204 |
| 근거 | [DB-설계.md · resources](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `resource_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

본문 없음.

**처리 규칙:** DB 결정대로 행을 실제 삭제하고 로그를 남깁니다. 계정 회수와 다르게 soft-delete하지 않습니다. DELETE 재요청의 없는 ID는 404입니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="base-01"></a>

### BASE-01 · 진화자원 기지 목록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/resource-bases` |
| 권한 | COMMANDER, VIEWER, ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · resource_bases](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| incident_id | ID | COMMANDER/VIEWER 필수 | 산불 문맥, ADMIN 선택 |
| org_type | string | 선택 | 기지 기관 구분 |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<ResourceBase>

```json
{
  "data": {
    "items": [
      {
        "id": "31",
        "name": "시연 항공관리소",
        "org_type": "forest",
        "geometry": {
          "type": "Point",
          "coordinates": [
            128.6,
            36.37
          ]
        },
        "coverage_area_ids": [
          "10"
        ],
        "distance_km": 22.3,
        "jurisdiction": "LOCAL"
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 고정 기지·관할을 조회합니다. 제안용 C-06 집계는 관할 전부와 응원 가까운 5곳이지만 이 API는 화면 목록이므로 페이지로 전체 허용 범위를 탐색합니다. 좌표 미상은 null, 거리 0으로 대체하지 않습니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="base-02"></a>

### BASE-02 · 기지 등록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/admin/resource-bases` |
| 권한 | ADMIN |
| 성공 상태 | 201 |
| 근거 | [DB-설계.md · resource_bases](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

없음.

**Request Body**

name: string(1~100), org_type: string(1~20), coverage_area_ids: ID[] 필수. geometry: Point 또는 null 선택

```json
{
  "name": "시연 항공관리소",
  "org_type": "forest",
  "geometry": {
    "type": "Point",
    "coordinates": [
      128.6,
      36.37
    ]
  },
  "coverage_area_ids": [
    "10"
  ]
}
```

**Response**

`data`: ResourceBase (산불 문맥 필드는 null)

```json
{
  "data": {
    "id": "31",
    "name": "시연 항공관리소",
    "org_type": "forest",
    "geometry": {
      "type": "Point",
      "coordinates": [
        128.6,
        36.37
      ]
    },
    "coverage_area_ids": [
      "10"
    ],
    "distance_km": null,
    "jurisdiction": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** coverage_area_ids는 SIDO 또는 SIGUNGU 행만 허용하고 존재를 확인합니다. 기지명 유일, 위치 미상 허용입니다.

**주요 오류:** 409 DUPLICATE_BASE_NAME, 422 INVALID_COVERAGE

<a id="base-03"></a>

### BASE-03 · 기지 수정


| 항목 | 내용 |
| --- | --- |
| Method·URL | `PATCH /api/v1/admin/resource-bases/{base_id}` |
| 권한 | ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · resource_bases](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `base_id`: ID

**Query parameter**

없음.

**Request Body**

name·org_type·coverage_area_ids·geometry 중 1개 이상. geometry만 null 허용. 배열은 전체 교체

```json
{
  "coverage_area_ids": [
    "10"
  ],
  "geometry": null
}
```

**Response**

`data`: ResourceBase (산불 문맥 필드는 null)

```json
{
  "data": {
    "id": "31",
    "name": "시연 항공관리소",
    "org_type": "forest",
    "geometry": null,
    "coverage_area_ids": [
      "10"
    ],
    "distance_km": null,
    "jurisdiction": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 관할 변경은 해당 기지 자원의 관할·응원 계산에 영향을 줍니다. 과거 제안 facts는 저장 당시 값을 유지합니다. 변경 로그를 남깁니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="base-04"></a>

### BASE-04 · 기지 삭제


| 항목 | 내용 |
| --- | --- |
| Method·URL | `DELETE /api/v1/admin/resource-bases/{base_id}` |
| 권한 | ADMIN |
| 성공 상태 | 204 |
| 근거 | [DB-설계.md · resource_bases·4.3](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `base_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

본문 없음.

**처리 규칙:** 소속 자원이 있으면 삭제를 거절하는 안입니다. 과거 proposal detail.targets가 참조하는 기지도 참조 해소/보존 정책 확정 전 삭제를 거절합니다(Q-10). 자동 자원 삭제·과거 이력 훼손은 하지 않습니다.

**주요 오류:** 409 BASE_IN_USE / HISTORICAL_REFERENCE

<a id="prop-01"></a>

### PROP-01 · 제안서 버전 목록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/proposals` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.4](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<Proposal에서 items 제외>

```json
{
  "data": {
    "items": [
      {
        "id": "71",
        "incident_id": "100",
        "prediction_id": "41",
        "perimeter_version": 1,
        "seq": 1,
        "version": "v1.1",
        "trigger": "PREDICTION",
        "correction_id": null,
        "judged_stage": "STAGE1",
        "predicted_stage": "STAGE1",
        "created_at": "2025-03-22T12:05:00+09:00",
        "proposal_stale": false
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** id DESC, 산불별 최신부터. 같은 화선에서 예측 재실행·상황 정정을 해도 seq가 증가합니다. 과거 화선 정정 여부는 PER-01로 확인합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="prop-02"></a>

### PROP-02 · 최신 제안서


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/proposals/latest` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · proposals·proposal_items](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: Proposal 또는 null

```json
{
  "data": {
    "id": "71",
    "incident_id": "100",
    "prediction_id": "41",
    "perimeter_version": 1,
    "seq": 1,
    "version": "v1.1",
    "trigger": "PREDICTION",
    "correction_id": null,
    "judged_stage": "STAGE1",
    "predicted_stage": "STAGE1",
    "created_at": "2025-03-22T12:05:00+09:00",
    "proposal_stale": false,
    "items": [
      {
        "id": "72",
        "item_code": "S1",
        "axis": "SUPPRESSION",
        "categories": [
          "STANDBY"
        ],
        "text": "현재 단계 기준을 유지합니다 [1].",
        "detail": {
          "status": "maintain",
          "finding": "현재 기준과 P5 기준 단계 유지",
          "facts": {
            "now": {
              "stage": "1단계",
              "area_ha": 8,
              "wind_ms": 2.1,
              "facility": 12
            },
            "p5": {
              "stage": "1단계",
              "area_ha": 9,
              "wind_ms": 2.6,
              "facility": 12
            }
          },
          "targets": [],
          "categories": [
            "대기"
          ],
          "withheld": [
            "진화예상시간: MVP 제외"
          ],
          "rule_basis": [
            "SM-p073"
          ]
        },
        "citations": [
          {
            "id": "81",
            "k": 1,
            "chunk_id": "91",
            "chunk_code": "SM-p073-01",
            "doc_name": "시연 근거 문서",
            "page": 73,
            "section": "대응단계 판단기준"
          }
        ],
        "change_type": "UNCHANGED"
      }
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 아직 제안서가 없으면 data=null입니다. 필터와 건수는 items의 axis·categories로 프론트에서 계산할 수 있습니다. 여러 categories를 가진 항목은 각 구분에 포함되므로 구분 건수 합과 전체 항목 수는 다를 수 있습니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="prop-03"></a>

### PROP-03 · 특정 버전 제안서


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/proposals/{proposal_id}` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.4](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID, `proposal_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: Proposal

```json
{
  "data": {
    "id": "71",
    "incident_id": "100",
    "prediction_id": "41",
    "perimeter_version": 1,
    "seq": 1,
    "version": "v1.1",
    "trigger": "PREDICTION",
    "correction_id": null,
    "judged_stage": "STAGE1",
    "predicted_stage": "STAGE1",
    "created_at": "2025-03-22T12:05:00+09:00",
    "proposal_stale": false,
    "items": [
      {
        "id": "72",
        "item_code": "S1",
        "axis": "SUPPRESSION",
        "categories": [
          "STANDBY"
        ],
        "text": "현재 단계 기준을 유지합니다 [1].",
        "detail": {
          "status": "maintain",
          "finding": "현재 기준과 P5 기준 단계 유지",
          "facts": {
            "now": {
              "stage": "1단계",
              "area_ha": 8,
              "wind_ms": 2.1,
              "facility": 12
            },
            "p5": {
              "stage": "1단계",
              "area_ha": 9,
              "wind_ms": 2.6,
              "facility": 12
            }
          },
          "targets": [],
          "categories": [
            "대기"
          ],
          "withheld": [
            "진화예상시간: MVP 제외"
          ],
          "rule_basis": [
            "SM-p073"
          ]
        },
        "citations": [
          {
            "id": "81",
            "k": 1,
            "chunk_id": "91",
            "chunk_code": "SM-p073-01",
            "doc_name": "시연 근거 문서",
            "page": 73,
            "section": "대응단계 판단기준"
          }
        ],
        "change_type": "UNCHANGED"
      }
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 과거 버전도 전체 항목 스냅샷으로 반환합니다. 공식 단계 필드는 없으며 judged_stage·predicted_stage만 반환합니다. 숫자·근거는 해당 버전에 연결된 것을 사용합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="prop-04"></a>

### PROP-04 · 직전 버전 비교


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/proposals/{proposal_id}/diff` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · proposal_items](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID, `proposal_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: {proposal_id: ID, previous_proposal_id: ID 또는 null, added: ProposalItem[], changed: {before: ProposalItem, after: ProposalItem}[], removed: ProposalItem[]}

```json
{
  "data": {
    "proposal_id": "71",
    "previous_proposal_id": null,
    "added": [
      {
        "id": "72",
        "item_code": "S1",
        "axis": "SUPPRESSION",
        "categories": [
          "STANDBY"
        ],
        "text": "현재 단계 기준을 유지합니다 [1].",
        "detail": {
          "status": "maintain",
          "finding": "현재 기준과 P5 기준 단계 유지",
          "facts": {
            "now": {
              "stage": "1단계",
              "area_ha": 8,
              "wind_ms": 2.1,
              "facility": 12
            },
            "p5": {
              "stage": "1단계",
              "area_ha": 9,
              "wind_ms": 2.6,
              "facility": 12
            }
          },
          "targets": [],
          "categories": [
            "대기"
          ],
          "withheld": [
            "진화예상시간: MVP 제외"
          ],
          "rule_basis": [
            "SM-p073"
          ]
        },
        "citations": [
          {
            "id": "81",
            "k": 1,
            "chunk_id": "91",
            "chunk_code": "SM-p073-01",
            "doc_name": "시연 근거 문서",
            "page": 73,
            "section": "대응단계 판단기준"
          }
        ],
        "change_type": "UNCHANGED"
      }
    ],
    "changed": [],
    "removed": []
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 직전은 같은 산불에서 현재 id보다 작은 가장 큰 제안서입니다. item_code로 맞추고 categories·text를 비교합니다. 첫 버전은 전부 added입니다. detail만 달라졌을 때 변경 태그를 확장할지는 기존 DB 기준을 유지하고 추후 별도 합의합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="prop-05"></a>

### PROP-05 · 정정 미반영 제안 재생성


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/incidents/{incident_id}/proposal-regenerations` |
| 권한 | COMMANDER |
| 성공 상태 | 202 |
| 근거 | [DB-설계.md · 2.5](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**추가 Header:** `Idempotency-Key` 필수.

**Path parameter:** `incident_id`: ID

**Query parameter**

없음.

**Request Body**

expected_prediction_id: ID, situation_revision: string 모두 필수

```json
{
  "expected_prediction_id": "41",
  "situation_revision": "rev-example-2"
}
```

**Response**

`data`: Operation

```json
{
  "data": {
    "id": "op-example",
    "kind": "PROPOSAL_REGENERATION",
    "status": "RUNNING",
    "stage": "PROPOSAL_GENERATION",
    "result": null,
    "error": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 새 예측을 돌리지 않고 최신 성공 예측과 승인된 현재 상황으로 제안을 만듭니다. 미반영 정정이 있어야 하며 가장 최근 미반영 correction_id를 연결하는 안입니다. 여러 정정 누적 시 한 제안이 어디까지 반영했는지 판별하는 기준은 Q-06에서 확정합니다.

**주요 오류:** 409 INCIDENT_ENDED / STALE_PREDICTION / STALE_SITUATION / REGENERATION_NOT_NEEDED, 422 NO_PREDICTION

<a id="evd-01"></a>

### EVD-01 · 인용 근거 본문


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/citations/{citation_id}` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · citations·evidence_chunks](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) · [데이터-자원.md · D-31](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/데이터-자원.md) |

**Path parameter:** `citation_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: CitationRef + {text: string, document_url: string 또는 null}

```json
{
  "data": {
    "id": "81",
    "k": 1,
    "chunk_id": "91",
    "chunk_code": "SM-p073-01",
    "doc_name": "시연 근거 문서",
    "page": 73,
    "section": "대응단계 판단기준",
    "text": "시연용 근거 본문입니다. 실제 서비스에서는 연결된 청크 본문을 반환합니다.",
    "document_url": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** citation→제안 항목 또는 질문→산불의 관할을 검증합니다. 청크가 retired_at을 가졌어도 지난 제안·답변의 근거는 계속 열려야 합니다. 내부 매뉴얼 원문 링크 공개 범위 미정으로 document_url=null을 제안합니다. 임베딩·FAISS 내부 번호는 노출하지 않습니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="ai-01"></a>

### AI-01 · 질의응답 또는 정정 확인안


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/incidents/{incident_id}/assistant/messages` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.5](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) · [LLM-입력-정리.md · A-01~05](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/LLM-입력-정리.md) |

**Path parameter:** `incident_id`: ID

**Query parameter**

없음.

**Request Body**

input_text: string(1~2000) 필수. proposal_id: ID 또는 null, item_code: string(최대 10) 또는 null 선택; 2000자 제한은 제안값

```json
{
  "input_text": "시연리 대피 완료",
  "proposal_id": "71",
  "item_code": null
}
```

**Response**

`data`: CorrectionPreview 또는 {type: ANSWER, message: QaMessage} 또는 {type: UNSUPPORTED, reason: string}

```json
{
  "data": {
    "type": "CORRECTION_PREVIEW",
    "confirmation_token": "EXAMPLE_PREVIEW_TOKEN",
    "expires_at": "2025-03-22T12:30:00+09:00",
    "situation_revision": "rev-example-1",
    "input_text": "시연리 대피 완료",
    "changes": [
      {
        "target": "village",
        "target_id": "1024",
        "item": "evacuated",
        "before": false,
        "after": true
      }
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 질문이면 ANSWER로 질문·답변을 저장하고 상황은 바꾸지 않습니다. 정정이면 확인안만 반환합니다. 허용 목록 밖 입력·모호한 대상은 UNSUPPORTED 또는 재질문 안내로 처리하고 저장하지 않습니다. 없는 village_id·근거를 LLM이 만들어낼 수 없습니다. 종료된 산불도 질문은 허용하지만 정정 확인안은 거절합니다. 여러 턴 자동 문맥은 제외합니다.

**주요 오류:** 409 INCIDENT_ENDED (정정 입력), 422 TARGET_NOT_FOUND / AMBIGUOUS_TARGET, 503 AI_UNAVAILABLE

<a id="ai-02"></a>

### AI-02 · 질의응답 이력


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/assistant/messages` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · qa_messages](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<QaMessage>

```json
{
  "data": {
    "items": [
      {
        "id": "111",
        "incident_id": "100",
        "user_id": "1",
        "proposal_id": "71",
        "item_code": "S1",
        "question": "이 단계의 판단 근거는 무엇인가요?",
        "answer": "판단 기준은 연결된 근거에서 확인할 수 있습니다 [1].",
        "citations": [
          {
            "id": "81",
            "k": 1,
            "chunk_id": "91",
            "chunk_code": "SM-p073-01",
            "doc_name": "시연 근거 문서",
            "page": 73,
            "section": "대응단계 판단기준"
          }
        ],
        "created_at": "2025-03-22T12:10:00+09:00"
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 저장된 질문·답변만 id DESC 조회합니다. 확인 전 정정 입력·취소한 확인 카드는 포함하지 않습니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="cor-01"></a>

### COR-01 · 확인한 상황 정정 저장


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/incidents/{incident_id}/corrections` |
| 권한 | COMMANDER |
| 성공 상태 | 201 |
| 근거 | [DB-설계.md · 2.5](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**추가 Header:** `Idempotency-Key` 필수.

**Path parameter:** `incident_id`: ID

**Query parameter**

없음.

**Request Body**

confirmation_token: string 필수. 클라이언트가 changes를 재작성해서 보내지 않음

```json
{
  "confirmation_token": "EXAMPLE_PREVIEW_TOKEN"
}
```

**Response**

`data`: {correction: Correction, situation_revision: string, proposal_stale: boolean, proposal_operation: Operation 또는 null}

```json
{
  "data": {
    "correction": {
      "id": "101",
      "incident_id": "100",
      "user_id": "1",
      "input_text": "시연리 대피 완료",
      "changes": [
        {
          "target": "village",
          "target_id": "1024",
          "item": "evacuated",
          "before": false,
          "after": true
        }
      ],
      "created_at": "2025-03-22T12:20:00+09:00"
    },
    "situation_revision": "rev-example-2",
    "proposal_stale": true,
    "proposal_operation": {
      "id": "op-example",
      "kind": "PROPOSAL_REGENERATION",
      "status": "RUNNING",
      "stage": "PROPOSAL_GENERATION",
      "result": null,
      "error": null
    }
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 토큰의 사용자·산불·유효시간·원문·변경안·revision과 서버 현재값·가용 자원 범위를 다시 확인합니다. 정정 행+상황 수정+로그를 먼저 한 트랜잭션으로 확정합니다. 예측이 있으면 제안 재생성 작업을 반환하고 없으면 null입니다. 제안 생성 실패는 이미 저장된 정정을 취소하지 않습니다. 작업 시작 실패 시에도 정정 저장 성공을 반환하고 proposal_stale=true, proposal_operation=null로 안내한 뒤 PROP-05로 재시도합니다.

**주요 오류:** 409 INCIDENT_ENDED / STALE_SITUATION / RESOURCE_CAPACITY_EXCEEDED, 422 INVALID_CONFIRMATION_TOKEN / EXPIRED_CONFIRMATION_TOKEN

<a id="cor-02"></a>

### COR-02 · 확인된 상황 정정 이력


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/corrections` |
| 권한 | COMMANDER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · situation_corrections](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<Correction>

```json
{
  "data": {
    "items": [
      {
        "id": "101",
        "incident_id": "100",
        "user_id": "1",
        "input_text": "시연리 대피 완료",
        "changes": [
          {
            "target": "village",
            "target_id": "1024",
            "item": "evacuated",
            "before": false,
            "after": true
          }
        ],
        "created_at": "2025-03-22T12:20:00+09:00"
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 입력 1건=정정 1행, changes 배열에 여러 항목을 담습니다. 취소·미확인 입력은 이력에 없습니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="log-01"></a>

### LOG-01 · 이벤트 로그


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/event-logs` |
| 권한 | COMMANDER, ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · event_logs](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| incident_id | ID | COMMANDER 필수 | 관할 산불 로그 |
| action | string | 선택 | 행위 코드 |
| user_id | ID | 선택 | 사용자별 |
| from_at / to_at | DateTime | 선택 | 발생시각 범위 |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<EventLog>

```json
{
  "data": {
    "items": [
      {
        "id": "121",
        "created_at": "2025-03-22T11:55:00+09:00",
        "user_id": "3",
        "incident_id": "100",
        "action": "PERIMETER_REPORTED",
        "details": {
          "perimeter_id": "21",
          "version_no": 1
        }
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 읽기 전용, created_at DESC·id DESC. COMMANDER는 관할 사건, ADMIN은 인증·계정·자원 관리 로그를 조회하는 권한안입니다. 로그 details에 신고자 개인정보·원문 비밀번호·토큰을 넣지 않습니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="usr-01"></a>

### USR-01 · 계정 목록


| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/admin/users` |
| 권한 | ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.1 계정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| role | UserRole | 선택 | 역할 |
| admin_area_id | ID | 선택 | 관할 시군구 |
| include_deleted | boolean | 선택 | 기본 false |
| cursor | string | 선택 | 서버 발급 불투명 커서 |
| limit | integer | 선택 | 기본 20, 최대 100 |

**Request Body**

없음.

**Response**

`data`: Page<AdminUser>

```json
{
  "data": {
    "items": [
      {
        "id": "1",
        "login_id": "cmd01",
        "name": "시연 사용자",
        "organization": "시연 지휘본부",
        "role": "COMMANDER",
        "admin_area_id": "10",
        "failed_login_count": 0,
        "created_at": "2025-03-22T11:55:00+09:00",
        "updated_at": "2025-03-22T11:55:00+09:00",
        "deleted_at": null
      }
    ],
    "next_cursor": null,
    "has_more": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 공통 권한·관할·입력 검증 규칙을 적용합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="usr-02"></a>

### USR-02 · 계정 발급


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/admin/users` |
| 권한 | ADMIN |
| 성공 상태 | 201 |
| 근거 | [DB-설계.md · 2.1 계정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** 없음

**Query parameter**

없음.

**Request Body**

login_id: string(1~50), name: string(1~50), organization: string(1~100), role: UserRole, admin_area_id: ID 또는 null 모두 필수

```json
{
  "login_id": "rep01",
  "name": "시연 보고자",
  "organization": "시연 보고기관",
  "role": "REPORTER",
  "admin_area_id": "10"
}
```

**Response**

`data`: {user: AdminUser, temporary_password: string}

```json
{
  "data": {
    "user": {
      "id": "3",
      "login_id": "rep01",
      "name": "시연 보고자",
      "organization": "시연 보고기관",
      "role": "REPORTER",
      "admin_area_id": "10",
      "failed_login_count": 0,
      "created_at": "2025-03-22T11:55:00+09:00",
      "updated_at": "2025-03-22T11:55:00+09:00",
      "deleted_at": null
    },
    "temporary_password": "EXAMPLE_INITIAL_PASSWORD"
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** login_id 대소문자 무시 유일, ADMIN만 관할 null입니다. 초기 비밀번호 발급·전달 및 관리자 계정 발급 가능 범위는 Q-09에서 확정합니다. 임시비밀번호는 로그에 남기지 않고 재조회로 반환하지 않습니다.

**주요 오류:** 409 DUPLICATE_LOGIN_ID, 422 INVALID_JURISDICTION

<a id="usr-03"></a>

### USR-03 · 계정 정보·권한 변경


| 항목 | 내용 |
| --- | --- |
| Method·URL | `PATCH /api/v1/admin/users/{user_id}` |
| 권한 | ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.1 계정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `user_id`: ID

**Query parameter**

없음.

**Request Body**

name·organization·role·admin_area_id 중 1개 이상, 타입은 발급과 동일. role을 ADMIN으로 바꾸면 admin_area_id=null도 함께 전송

```json
{
  "organization": "변경된 시연 기관"
}
```

**Response**

`data`: AdminUser

```json
{
  "data": {
    "id": "1",
    "login_id": "cmd01",
    "name": "시연 사용자",
    "organization": "변경된 시연 기관",
    "role": "COMMANDER",
    "admin_area_id": "10",
    "failed_login_count": 0,
    "created_at": "2025-03-22T11:55:00+09:00",
    "updated_at": "2025-03-22T11:55:00+09:00",
    "deleted_at": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 관할의 마지막 COMMANDER는 회수하거나 COMMANDER 역할/해당 관할에서 빼지 못합니다. users 변경 후 다음 요청부터 새 권한을 적용합니다.

**주요 오류:** 409 LAST_COMMANDER, 422 INVALID_JURISDICTION

<a id="usr-04"></a>

### USR-04 · 계정 회수


| 항목 | 내용 |
| --- | --- |
| Method·URL | `DELETE /api/v1/admin/users/{user_id}` |
| 권한 | ADMIN |
| 성공 상태 | 204 |
| 근거 | [DB-설계.md · 2.1 계정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `user_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

본문 없음.

**처리 규칙:** users 행은 지우지 않고 deleted_at을 기록합니다. 관할의 마지막 COMMANDER는 회수 금지입니다. 기존 JWT라도 다음 요청의 users 조회에서 거절합니다.

**주요 오류:** 409 LAST_COMMANDER

<a id="usr-05"></a>

### USR-05 · 비밀번호 재발급


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/admin/users/{user_id}/password-reset` |
| 권한 | ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.1 계정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `user_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: {user_id: ID, temporary_password: string}

```json
{
  "data": {
    "user_id": "3",
    "temporary_password": "EXAMPLE_INITIAL_PASSWORD"
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 서버가 새 비밀번호를 만들어 해시를 저장하고 이번 응답에서만 전달하는 안입니다. 기존 JWT 폐기와 최초 변경 강제는 현 스키마에 근거가 없어 Q-09에서 확정해야 합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="usr-06"></a>

### USR-06 · 로그인 잠금 해제


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/admin/users/{user_id}/unlock` |
| 권한 | ADMIN |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.1 계정](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**Path parameter:** `user_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: AdminUser

```json
{
  "data": {
    "id": "1",
    "login_id": "cmd01",
    "name": "시연 사용자",
    "organization": "시연 지휘본부",
    "role": "COMMANDER",
    "admin_area_id": "10",
    "failed_login_count": 0,
    "created_at": "2025-03-22T11:55:00+09:00",
    "updated_at": "2025-03-22T11:55:00+09:00",
    "deleted_at": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** failed_login_count를 0으로 설정합니다. 회수 계정을 재활성화하지 않습니다.

**주요 오류:** 409 ACCOUNT_REVOKED

<a id="rules"></a>

## 7. 지도·규칙 엔진 결과의 화면 적용

### 7.1 지도 레이어 ID — API 매핑 제안

| layer_id | 원본 | 화면 표시 |
| --- | --- | --- |
| admin_areas | admin_areas.geom | 행정경계, 속성 level로 시도·시군구·읍면 구분 |
| villages | villages.geom | 법정리 경계·추정 인구. 라벨은 label_point |
| facilities | facilities.geom | 시설 종류별 아이콘·팝업, type으로 필터 |
| road | map_features.kind=road | 큰 도로, ref·highway 유지 |
| track | map_features.kind=track | 임도 대용, OSM 농로가 섞인 자료임을 구분 |
| power_line / power_tower / substation | 각 map_features.kind | 송전선·철탑·변전/발전소 |
| water | map_features.kind=water | 헬기 취수 후보. 실제 사용 가능 확정 자료 아님 |
| house | map_features.kind=house | 주택 점, 축척에 따라 표시 |

바탕지도·위성영상·라벨은 이 공간 데이터 API와 별개 외부 제공자 리소스입니다. 서버 API 키·내부 파일 경로를 프론트에 보내지 않습니다. OpenFreeMap 사용 시 데이터 자원 문서의 출처 표기를 적용합니다. 실제 운영 스타일·출처 표기는 배포 설정에서 확인합니다.

### 7.2 targets → 지도 연결

최신 규칙 엔진의 출력 참조를 서버에서 아래 형식으로 정규화하는 안입니다. **문자열 안 숫자는 해당 테이블의 실제 PK**이며 이름을 ID처럼 보내지 않습니다.

| 참조 | MAP-03 entity_type | 표시 대상 |
| --- | --- | --- |
| village:1024 | village | villages.geom |
| facility:55 | facility | facilities.geom |
| map_feature:1021 | map_feature | 원본 도로·송전 등 요소 |
| resource_base:31 | resource_base | 고정 기지. 위치 null이면 지도 표시 생략 |
| admin_area:11 | admin_area | 행정구역 경계 |

도로 통제 구간 하나가 여러 도로 행을 합친 결과면 targets에 관련 map_feature ID를 모두 둡니다. `map_feature:power_line:3` 같은 혼합 예시와 `admin_area:안평면` 같은 이름 참조는 규칙 엔진과 어댑터 사이에서 바꿔야 합니다. 어떤 필드에서 정규화·저장할지는 Q-04 검토 사항입니다.

### 7.3 제안 detail.facts 해석

`RuleDetail`의 공통 6개 키는 유지하고, facts는 항목마다 다릅니다. 아래는 현재 규칙 엔진 설계의 필드이며 전체 원문은 1절의 고정 커밋 링크로 추적합니다. 엔진의 판정 status와 categories를 프론트가 자체 재판정하지 않습니다.

| 항목 | 화면 주제 | facts에서 읽는 정보·현재 범위 |
| --- | --- | --- |
| S1 | 대응단계·격상 대비·지휘권 인계 | now/p5의 stage·area_ha·wind_ms·facility·요소별 단계·decided_by, change.prev/type, commander.reader/now/next/handover/reason. status는 maintain 또는 escalation_expected 등 |
| S2 | 우선지역·보호대상 | life.villages/vulnerable, infrastructure, property.villages/total, forest.area_p5_ha. 인명→송전·국가유산→주택→산림 순의 근거 기반 안내 |
| S3 | 자원 투입·응원 | deployed, scope.now/p5, standby_now, added_at_p5. 기지·거리·대기 대수. 추가 투입 대수·배치 구역 산정은 근거 없어 withheld |
| S4 | 낮·밤 전략 | sun, wind[]의 time/dir/ms, max.ms/time. 운용 가능 여부를 임의로 판단하지 않고 withheld의 풍속·시정 기준 공백 표시 |
| S5 | 시설 보호 | houses.villages/total, power[]의 kind/name/kv/zone/km, heritage[]의 name/grade/zone |
| S6 | 대책회의·협조 | requests[]의 item/what/targets/near. E3·E5·E7·S3·S5에서 타 기관에 요청할 일을 모음. 지휘권 인계는 S1로 이동 |
| S7 | 진화인력 안전 | 바람·확산 방향·임도 참고 등 기존 LLM 입력 범위. 최신 문서에서 S7의 구체 facts 계약이 완성되지 않았으므로 Q-07에서 확정 |
| E1 | 위험구역 | burning/p5/p8 마을의 읍면·리. 법정리 영역 겹침이며 주거지에 실제 불이 닿았다는 확정이 아님 |
| E2 | 대피 순서·시점 | immediate/pre_evac, 이름·읍면·people/by/elderly/night/evac_order/evacuated, sunset. 도달 시각→65세 이상 순, 야간은 일몰 전 안내 |
| E3 | 취약시설 조치 | immediate/pre_evac, 시설명·kind·village·emd·by·daytime_only, sunset. 마을 단계로 조치. 차량 필요 대수 산정은 제외 |
| E4 | 대피소 후보 | facts.villages는 shelter_options. 마을별 people/arrival/evacuated, village_sites/hub_sites, excluded, decision_needed. 아래 후보 DTO 참조 |
| E5 | 도로 통제 요청 | facts.roads는 road_sections 배열. road/ref/grade/section/emd/zone/arrival/km/police. 예시의 'road_sections 그대로' 문구는 실제 데이터로 대체 |
| E6 | 재난문자·방송 전파 제안 | areas[]의 sigungu/emd/stage(go 또는 set)/villages/change(new·up·same)/prev. 전체 대상 목록과 직전 제안 대비 변화. 실제 발송·문안 작성은 제외 |
| E7 | 인명 수색·구조 요청 | villages[]의 name/emd/arrival/evac_order/fire_station(name·km). 이미 닿았고 대피 완료 보고 없는 마을. 출동 안전센터·고립 여부는 판정하지 않음 |

위 facts의 세부 중첩 키는 항목별 설계입니다. 엔진 내부 JSON은 11/5 문서와 동기화하고, 프론트는 원문 전체를 강한 고정 타입으로 단정하기 전에 담당자와 타입을 확정합니다. 화면 공통 기능은 finding·targets·withheld·text·citations를 먼저 연결할 수 있습니다.

### 7.4 E4 대피소 후보

대피소를 자동 배정하거나 주민을 여러 시설로 나누지 않습니다. 리·읍면·시군 경계로 후보를 자르지 않고 **마을 시설·거점 시설 각각 가까운 2곳**을 제시합니다. 마을 대표점부터 직선거리로 계산하며, 이동 경로·차량 소요시간으로 표시하지 않습니다.

| 필드 | 타입·의미 |
| --- | --- |
| village / emd / zone | string. 대상 리·읍면·구역(p5 또는 p8 등 원문값) |
| arrival | string. 서버가 만든 표시 시각, 다음 날 구분 포함. 프론트가 단독 HH:mm로 재계산하지 않음 |
| people / evacuated | integer / boolean. 위험구역은 인구, 잠재 위험구역은 65세 이상, 대피 완료면 people=0 |
| village_sites / hub_sites | 후보 배열. 각 name·kind: string, safety: safe 또는 exception, km: number, capacity: integer 또는 null |
| excluded | {name: string, reason: string}[] |
| decision_needed | 후보가 전혀 없을 때 string, 그 외 생략 |

P0~P5 안 후보는 제외합니다. P6~P8 후보는 원칙 제외이고 안전 후보가 없을 때만 exception으로 별도 표시합니다. 모텔은 후보에서 제외합니다. 개방 여부·남은 수용 인원은 실시간 자료가 없어 확인 필요로 안내합니다.

### 7.5 E5 도로·E6 전파 구분

- E5는 P0~P5와 겹치는 큰 도로를 노선 번호(ref, 없으면 도로명)로 묶어 이어진 구간을 만듭니다. E5의 zone=p0은 이미 닿음, p5는 5시간 안입니다. 다른 결과 블록의 burning/p5/p8 표현과 혼동하지 않습니다.
- E5는 구간명·범위 내 길이·가장 이른 도달 시각·가까운 **경찰서**를 표시합니다. 정확한 통제 지점·우회로·통제 순위는 제안하지 않습니다. 요청 기관은 도로 등급과 근거를 받아 LLM이 문장에 적습니다.
- E6의 Go/Set은 대피 전파 제안 단계이고, 실제 재난문자 발송 여부나 공식 대응단계가 아닙니다. `disaster_sms_sent` 하나로 읍면별 발송 이력을 추정하지 않습니다.

### 7.6 위험도 23개 변수

R = Σ(포함 변수의 원가중치 × 등급) ÷ Σ(포함 변수의 원가중치). 모든 변수가 있으면 분모는 **0.94**입니다. 빠진 변수를 제외할 때 0.94로 고정해서 나누지 않습니다. 포함 가중치 합이 0이면 score=null입니다. 등급은 변수별 1~5, 최종 R은 1~5이며 화면에서 소수 둘째 자리로 표시하는 안입니다. 기여도 합은 score와 일치해야 합니다.


| API 변수 코드(제안) | 원문 변수 | 축 | CSV 원가중치 |
| --- | --- | --- | --- |
| slope_up | 경사도_up | terrain | 0.08 |
| aspect | 사면방향 | terrain | 0.06 |
| curvature | 곡률 | terrain | 0.04 |
| terrain_complexity | 지형복잡도 | terrain | 0.03 |
| elevation | 고도 | terrain | 0.02 |
| topographic_position | 지형위치지수 | terrain | 0.02 |
| water_distance | 수계와의 거리 | terrain | 0.02 |
| fuel_load | 산림밀도/연료량 | fuel | 0.05 |
| forest_type | 수종종류 | fuel | 0.04 |
| surface_fuel_depth | 지표면 연료깊이 | fuel | 0.04 |
| stand_age | 평균수령 | fuel | 0.02 |
| canopy_height | 수관높이 | fuel | 0.02 |
| canopy_cover | 수관피복률 | fuel | 0.02 |
| wind_speed | 풍속 | weather | 0.10 |
| humidity | 상대습도 | weather | 0.07 |
| temperature | 기온 | weather | 0.05 |
| precipitation | 강수량 | weather | 0.04 |
| ground_temperature | 지면온도 | weather | 0.04 |
| precipitation_7d | 7일 누적 강수량 | weather | 0.03 |
| pressure | 기압 | weather | 0.02 |
| fuel_moisture | 수분함량 | fuel | 0.06 |
| vulnerable_population | 취약계층 수 | infrastructure | 0.04 |
| fire_station_distance | 소방서와의 거리 | infrastructure | 0.03 |

등급 구간은 [위험도 정리](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/위험도-정리.md)에 적힌 원표를 기준으로 하되, 등호 경계 중복·변환식·시간/격자 기준은 Q-01에서 정해야 합니다. 미확보였던 소방서·수계 자료를 현재도 전부 없다고 단정하지 않습니다. 최신 지도·자원 문서의 확보 상태와 계산 가능 여부를 각각 확인합니다. 기압·지면온도를 관측값으로 보완할 때 예측 시점의 실제 값처럼 표시하지 않습니다.

<a id="flow"></a>

## 8. 화면 연동 순서와 검증 항목

| 화면·동작 | API 호출·표시 흐름 |
| --- | --- |
| 로그인·새로고침 | AUTH-01 → AUTH-02. 401은 로그인, 403은 접근 제한 안내 |
| 산불 선택 | INC-01 → INC-03 → PER-01. 미생성 예측·제안 참조는 null로 처리 |
| 보고 지도 | MAP-01 → MAP-02·MAP-04. 발화점·실측 폴리곤 작성, 서버 결과로 버전 갱신 |
| 보고·정정 | INC-02 또는 PER-02·03 → INC-03·PER-01 재조회. 새 화선이면 재예측 안내 |
| 지도 탐색 | bbox 변경에 맞춰 MAP-02. 이전 응답이 뒤늦게 와도 현재 화면 범위를 덮어쓰지 않게 취소/요청 구분 |
| 예측 실행 | PRED-01의 202 → PRED-05 조회 → 성공 후 INC-03 재조회 및 PRED-02·03·04, RISK-01, PROP-02를 동일 결과 ID로 연결 |
| 예측 실패 | 기존 도형·제안 유지, 작업 오류 안내. 버튼만 재활성화하고 중복 요청을 피함 |
| 제안 필터·지도 강조 | PROP-02/03의 axis·categories·detail.targets 사용 → MAP-03 대상 조회. targets가 빈 배열이면 강조 없음 |
| 근거·이력 | 문장 [k] → EVD-01, PROP-01 → PROP-03/04. 해당 버전의 근거·수치 표시 |
| 질문 | AI-01 ANSWER → 질문·답변 및 인용 표시. AI-02는 기록만 조회, 자동 다중 턴 문맥 아님 |
| 상황 정정 | AI-01 확인안 → 사용자가 변경 전후 확인 → COR-01 → 정정 성공 즉시 상황 갱신. proposal_operation이 있으면 PRED-05 조회 |
| 정정 후 생성 실패 | 정정은 저장됐다는 안내와 proposal_stale 표시 → PROP-05로 재시도. COR-01을 새 정정처럼 중복 호출하지 않음 |
| 자원·기지 관리 | RES-01/02·BASE-01 조회 → RES-03~05·BASE-02~04. 인력은 Situation 투입 수 표시만 |
| 종료 | INC-04 → 쓰기 버튼 차단. 기존 제안·이력·질문 허용, 종료 위험도는 합의 정책 적용 |

### 프론트·백엔드 공동 확인

- [ ] v0(발화점만)·화선 v1 이상·화선 정정 이력의 표시가 다르게 나온다.
- [ ] base_at은 관측시각이며 P1~P8·자정 넘는 시각·기상 정시 9개가 맞는다.
- [ ] 새 화선·동시 정정·종료 중 계산 결과가 오래된 상태를 최신으로 덮어쓰지 않는다.
- [ ] 202는 완료가 아니며 작업 실패 시 기존 결과를 유지한다.
- [ ] 정정 저장 성공과 제안 생성 실패를 구분하고 재시도해도 정정이 중복 저장되지 않는다.
- [ ] 부분 위험도·미확인 인구·capacity null·위치 미상 기지를 0/안전으로 표시하지 않는다.
- [ ] 대피소는 후보이며 도로 통제는 요청 안내다. 자동 배정·실제 발송 완료로 표시하지 않는다.
- [ ] 401·403·404·409·422·503, 빈 목록·제안 없음·근거 부족·지도 페이지 분할을 처리한다.
- [ ] bigint 문자열·법정동코드·GeoJSON 좌표순서와 실제 서버 OpenAPI가 일치한다.

<a id="questions"></a>

## 9. 구현 전 팀 확인 사항

과거 문서 차이를 그대로 미결 목록에 복사하지 않고 **현재 5개 설계 문서에도 남아 있거나 API 구현에 반드시 필요한 내용**만 정리했습니다. 완료된 항목은 담당자 검토 근거와 함께 갱신합니다.

| ID | 확인할 내용 | 현재 근거·질문 | 영향 |
| --- | --- | --- | --- |
| Q-01 | 위험도 저장·계산·종료 정책 | DB 4.2: predictions 컬럼 추가/별도 risk_assessments, 등급 래스터 저장소, 종료 후 최대 위험도 표시가 미결. 변수별 구간 경계·수분/지형복잡도·임상도 변환·격자 30/100m·기상 시점/공간·취약인구 범위·소방서 거리 기준·최종 표시 등급도 확인. 위험도 미산출 시 예측 전체 실패인지 부분 허용인지 합의 | RISK-01, PRED-01, E2·AI |
| Q-02 | 현재 단계 판정 시 P5 가용 시점 | DB는 산불/화선 보고 때 current_stage 판정, 시설피해 우려는 P5 안 주택+국가유산으로 규정. 최초 보고·새 화선 보고 직후 해당 P5가 아직 없거나 최신 화선과 예측 기준이 다를 때 무엇으로 판정할지 불명확. 예측 전 시설피해 결측·기상 수신 실패·NN 단계 저장·관측시각 역전도 함께 결정 | INC-02, PER-02·03, S1 |
| Q-03 | 위기경보 유지 여부 | DB는 alert_level 입력·정정 유지, LLM·규칙 문서는 쓰는 곳이 없어 제외/팀 논의. API는 DB 필드로 남기되 실제 입력 UI·필수 여부·정정 범위를 최종 확인 | INC·COR, S1·S6 |
| Q-04 | detail·targets 최종 계약 | DB 옛 예시는 targets 객체, 최신 엔진은 문자열 배열. 이름 참조·kind 포함 참조도 혼재. 서버 어댑터의 PK 기반 정규화·저장 위치·권한 검증·외부 한글/영문 categories 매핑을 확정 | PROP·MAP·E5·E6 |
| Q-05 | 작업·멱등키·동시성 | 202/작업 조회는 API 제안. 작업 큐·공유 저장소·보존기간·재시작 복구, idempotency 결과 재전달, 화선·상황·자원 변경·종료 중 커밋 정책을 확정. predictions에는 성공 행만 저장 | PRED·COR·PROP-05·보고 |
| Q-06 | 제안 stale·정정 반영 기준 | DB는 정정시각과 제안시각 비교. 다중 정정·재시도·여러 쓰기 경로·동일시각·재생성 대상을 안전하게 판별할 revision을 어떻게 둘지 결정. rule context 저장 위치도 DB에 없음(proposals.context는 제안일 뿐) | INC-03, COR, PROP-05, AI |
| Q-07 | S7·미결 규칙·근거 준비 | 주 확산 방향·S7 facts, 시군구 최소 겹침면적(구현 기본 1ha 제안), 표준매뉴얼 청킹·rule_basis→실제 chunk 매핑·추가 검색 K·인용/숫자 검증을 확인. 문서에 적힌 코드가 실제 검색 가능한 청크임을 가정하지 않음 | 규칙·PROP·AI·EVD |
| Q-08 | 역할·인접 범위·개인정보 | VIEWER의 제안/근거/AI 접근, REPORTER 지도·상황 반환 범위, 관할 밖 소방·경찰·병원·응원 자원, 신고자 이름·전화 마스킹, 내부 매뉴얼 본문·원문 공개 범위를 확인 | 권한 전체·INC·MAP·RES·EVD |
| Q-09 | 인증·관리 API 운영 | 실제 라우트·응답 형식·CORS, JWT 만료값·로그아웃 폐기, 비밀번호 재발급 시 기존 JWT 무효화·초기 변경 강제·관리자 계정 발급 범위, 자원 동시 편집 방식을 확인. DB에 없는 세션/토큰버전/자원 updated_at은 구현 완료로 가정하지 않음 | AUTH·USR·RES |
| Q-10 | 적재·기지 삭제·이력 | 전국 시도/시군구, 기지 coverage_area_ids, house, 대피소 tier, 도로 ref·인근 경찰서 적재를 확인. 기지 실제 삭제와 과거 detail.targets 참조 보존 정책 필요. 마을·시설 재적재 시 기존 ID 유지 | MAP·BASE·규칙·과거 제안 |

설계 문서만으로 실제 서버 구현·DB 생성·공유 파일 적재 여부는 확인할 수 없습니다. 따라서 '지도·자원 테이블이 아직 없다'고 쓰지 않고 **테이블 정의는 있으며 필요한 데이터와 구현 상태를 확인해야 한다**고 구분했습니다. 근거 저장은 최신 DB 원칙·LLM·데이터 문서의 FAISS 선택을 따르며, DB 본문의 과거 pgvector/FAISS 미정 문구를 현재 전체 결정으로 확대하지 않았습니다.

<a id="changes"></a>

## 10. 노션 v0.3에서 바뀐 내용

| 변경 | 이번 반영 |
| --- | --- |
| 근거 기준 | 10/30 노션 원문 이관 → 11/6 조회한 main의 설계 파일 5개 기준 재작성 |
| 공식 대응단계 | official_stage 필드·입력·정정 제거. current_stage 자동 판정, judged_stage/predicted_stage 구분 |
| 시설피해 | 피해 우려=P5 안 주택+국가유산, 현재 화선 안 동수=피해 발생으로 구분 |
| 지도·자원 | 정의되지 않았다는 과거 설명 제거. admin_areas·villages·facilities·map_features·resource_bases·resources의 현재 구조 반영 |
| 기지 관리 | BASE-01~04 추가. 기지와 개별 자원 분리, coverage_area_ids·고정 위치·관할/응원 판정 |
| 자원 속성·삭제 | base_id·attrs 사용. DB에 없는 updated_at·deleted_at 제거, 자원은 실제 행 삭제. 인력 개별 CRUD 제외 |
| 지도 강조 | 기존 village/facility뿐 아니라 map_feature·resource_base·admin_area 참조 반영, 형식 혼재는 Q-04로 분리 |
| 대피소 | 인원 자동 배정 대신 마을/거점 후보 각 2곳, 직선거리·안전/예외·결정 필요 표시 |
| E5 | 대피로/진입로 충돌 대신 P0~P5 큰 도로 통제 구간·도달 시각·가까운 경찰서 |
| S1·S6 | 지휘권 인계는 S1, S6는 타 항목의 요청을 모아 대책회의·기관 협조 안내 |
| E6·E7 | 읍면별 Go/Set·직전 대비 변화, 이미 닿고 대피 완료 보고 없는 마을의 가까운 소방서 |
| 위험도 | 23개 변수·가용 가중치·결측·저장 공백·종료 정책을 분리해 조건부 API로 표시 |
| 검토 목록 | 과거 '미확보' 항목의 일괄 나열 대신 현재 미결과 API 구현 합의 10개로 갱신 |

### 변경 이력

| 날짜 | 판 | 내용 |
| --- | --- | --- |
| 2026-10-30 | v0.3 | 기존 노션 API 초안(이 문서의 이전 기준) |
| 2026-11-06 | v0.4 검토본 | main 고정 커밋의 설계 5개 재검토, 46개 API·요청/응답·규칙/지도 계약 재작성. docs/api-spec-v0.4 브랜치에 추가 |

요청·응답 타입과 미결 항목을 팀에서 검토한 후 main에 병합합니다. 이 문서는 docs/api-spec-v0.4 작업 브랜치에 추가하며 기존 설계 파일 5개는 수정하지 않습니다. 이후 API 계약을 바꾸면 관련 DTO·프론트 타입·목업 응답·서버 OpenAPI도 같은 변경에 맞춰 관리합니다.
