# 산불 대응 시스템 API 명세서

| 항목 | 내용 |
| --- | --- |
| 문서 판 | v0.5 · 서버 구현 대조 검토본 |
| 작성·조회 기준일 | 2026-11-10 (v0.4: 2026-11-06) |
| 범위 | React·TypeScript·MapLibre 프론트엔드 ↔ FastAPI 백엔드 REST API |
| 상태 | P1·P2는 서버 `907bd42`와 대조. AI·상황 정정 5개(AI-01·02, COR-01·02, PROP-05)는 서버 PR #24의 구현과 11/10 팀 결정을 추가 반영. P3 전체의 구현 여부와 구분(11절) |
| 저장 예정 위치 | `docs/설계/API-명세서.md` |
| 기준 저장소 | `wildfire-guard-capstone/wfg-docs`의 main |
| 고정 기준 커밋 | v0.4 설계 원본 `43dc969703cdb65192e07e1341785ce38bca60c6` · v0.5 DB 대조 `4025a901d1b69e6ced9d8f0b7b48fd2cc1b97096` · 서버 `907bd42ea9b5c96704b0511567ec79e8b9dfecf5` |
| GitHub 반영 | `docs/api-spec-v0.5` 작업 브랜치 · main 병합 전 팀 검토용 |

> 이 문서는 노션 v0.3의 단순 복사본이 아닙니다. 아래 설계 문서 5개를 다시 읽어 현재 결정에 맞춰 API 계약을 재작성했습니다. 도메인 규칙의 근거와 API 작성자가 제안한 전송 형식을 구분합니다. 경로·권한표·응답 래퍼·작업 조회·확인 토큰·추가 DTO 필드는 서버 구현 완료나 팀 합의를 뜻하지 않습니다.

> **v0.5**: v0.4를 서버 구현과 대조해 고친 판입니다. 성공·실패 구조는 v0.4를 따르고, 바뀐 곳(항목 12개 제안서 응답, 위기경보 삭제, 근거 원문을 제안서 안으로, 지금 날씨 API 추가 등)은 [10절](#changes)에 모았습니다. 아직 정하지 않은 것은 **확인 필요**로 표시했습니다.

## 목차

- [1. 기준 문서와 적용 범위](#sources)
- [2. 공통 요청·응답 규칙](#common)
- [3. 권한과 상태값](#enums)
- [4. 공통 DTO](#dtos)
- [5. API 전체 목록](#index)
- [6. API 상세](#endpoints)
- [7. 지도 레이어와 위험도 변수](#rules)
- [8. 화면 연동 순서와 검증 항목](#flow)
- [9. 구현 전 팀 확인 사항](#questions)
- [10. 바뀐 내용(v0.4 → v0.5, v0.3 → v0.4)](#changes)
- [11. API 우선순위(P1 · P2 · P3)](#priority)

<a id="sources"></a>

## 1. 기준 문서와 적용 범위


| 문서 | 표기된 최종 갱신일 | API에 반영한 내용 |
| --- | --- | --- |
| [DB-설계.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/4025a901d1b69e6ced9d8f0b7b48fd2cc1b97096/docs/설계/DB-설계.md) | 2026-11-09 | v0.5 DB 대조 기준: 제안서 context·근거 스냅샷·자원 연결을 포함한 필드·저장 흐름. v0.4 작성 당시의 DB 기준은 11/5 판 |
| [규칙-엔진-설계.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/규칙-엔진-설계.md) | 2026-11-05 | 현재/P5 단계, S1~S6·E1~E7(v0.5: 11/7 12항목 S1~S6·E1~E6), context·items, 지도 targets·대피소·도로 구간 |
| [LLM-입력-정리.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/LLM-입력-정리.md) | 2026-10-27 | 수치·판정은 규칙 엔진, LLM은 문장 생성, 질의·정정 입력. 표 안에는 11/1~11/5 검토 메모도 있음 |
| [데이터-자원.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/데이터-자원.md) | 2026-10-30 | 자료 출처·확보 범위·제외 항목·추정값·시연 자료 한계. 이후 작성된 메모도 있음 |
| [위험도-정리.md](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/위험도-정리.md) | 2026-10-26 | 23개 변수·원가중치 합 0.94·가용 변수 가중평균·미결 계산 기준 |
| `wfg-server` main `907bd42` (v0.5) | 2026-11-10 | P1·P2 실제 라우트·요청/응답·오류 코드. 제안 항목 12개(S1~S6·E1~E6, 11/7)와 제안서 저장 구조(DB-설계 2.4, 11/9) |

파일 머리의 갱신일만으로 우선순위를 정하지 않습니다. 같은 주제의 명시된 결정과 해당 절의 최신 검토 메모를 우선합니다. 설계 문서 4개의 고정 링크는 v0.4 작성 근거를 보존한 것이며, v0.5는 위의 11/9 DB 설계와 서버 907bd42를 추가로 대조했습니다. v0.5 제안 응답에서는 과거 detail.targets·지도 강조 계약을 제외합니다. 충돌·미결은 9절에 남깁니다. 매뉴얼 기준은 팀 설계 문서에 적힌 내용이며 이번 작업에서 원문 매뉴얼을 별도로 재검증하지 않았습니다.

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
- 결과를 재전달하는 멱등 처리가 확정된 것은 아래 INC-02입니다. PER-02·03·PRED-01은 키를 받는 것만으로 같은 성공 응답 재전달을 보장하지 않습니다. COR-01·PROP-05의 멱등 저장·재전달은 아직 검토 제안입니다. 같은 키로 다른 본문의 409 `IDEMPOTENCY_KEY_REUSED`는 INC-02에 적용합니다.
- 산불별 advisory lock으로 중복 예측·제안 커밋·종료·정정을 조율합니다. 계산 중에는 긴 DB 트랜잭션을 열지 않고 커밋 직전에 종료·화선·상황 변경을 다시 확인합니다.
- **INC-02 확정(v0.5)**: 같은 보고자·같은 키·같은 본문을 다시 보내면 처음 만든 산불을 그대로 201로 돌려주고, 같은 키·다른 본문은 409 `IDEMPOTENCY_KEY_REUSED`입니다. 키는 최대 100자이며 DB(`incidents.idempotency_key`)에 저장합니다. 서버는 키 없이도 받습니다(이때 중복을 막지 않음). PER-02·03·PRED-01은 키를 받기만 하며, 같은 요청의 재전송은 409 `STALE_PERIMETER`·`PREDICTION_RUNNING`으로 막힙니다.
- situation_revision은 DB 컬럼으로 확정하지 않은 불투명 토큰입니다. **v0.5 서버는 `rev-{최신 상황 정정 id}`(정정이 없으면 `rev-0`)를 사용합니다. COR-01은 서버 PR #24와 명세 PR #67에서 구현·계약을 보완했습니다.** 최신 정정 id만으로 충분한지, 관리자/배치 변경·기상·자원 갱신을 어떻게 감지할지 Q-05·Q-06에서 합의합니다.
- 작업·중복키 결과 보존, 프로세스 재시작·다중 worker, 키 만료·응답 재전달은 Q-05의 구현 전 확인 사항입니다.

<a id="enums"></a>

## 3. 권한과 상태값

### 3.1 역할별 범위 — 권한 API 초안

| 역할 | 역할명 | 제공 기능 |
| --- | --- | --- |
| COMMANDER | 통합지휘권자 | 관할 현황·지도·자원·예측·위험도·제안·근거·질의·정정·이력·종료 |
| VIEWER | 열람자 | 관할 현황·지도·자원·예측·위험도 읽기. 제안·AI 접근은 이 초안에서 제외, Q-08 확인 |
| REPORTER | 상황 보고자 | 관할 산불 등록·보고 정보·보고 지도·화선 보고·정정. 분석 데이터는 제한 |
| ADMIN | 전산 관리자 | 계정·기지·자원 관리와 해당 로그, 진행 중 산불 목록(INC-01, 전체 지역 — 자원을 산불에 연결할 때 선택). 지휘권자 기능을 자동 부여하지 않음 |

위 세부 접근 범위는 DB의 4개 역할을 화면 기능에 매핑한 제안입니다. 다른 계정이 보낸 role·admin_area_id로 조회 권한을 넓히지 않습니다. 인접 시군 자원·시설은 백엔드가 확정한 지원 조회 범위에서만 제공합니다.

### 3.2 공통 상태 코드


| 타입 | 값 | 근거·의미 |
| --- | --- | --- |
| UserRole | COMMANDER, VIEWER, REPORTER, ADMIN | DB users.role |
| IncidentStatus | IN_PROGRESS, ENDED | 실측 화선 없음은 별도 상태 대신 v0으로 구분 |
| ResponseStage | INITIAL, STAGE1, STAGE2 | 현재·예측 판정 단계. 공식 발령 단계가 아님 |
| ItemAxis | SUPPRESSION, EVACUATION | 진화/대피 |
| ItemCategory | IMMEDIATE, STANDBY, CONSULT, REQUEST | 즉시/대기/협의/요청 |
| ResourceKind | HELICOPTER, VEHICLE | 개별 인력 없음 |
| ResourceStatus | AVAILABLE, DEPLOYED, MAINTENANCE | 대기/투입/정비 |
| FacilityType | SHELTER, VULNERABLE, HOSPITAL, FIRE_STATION, POLICE, HERITAGE | DB facilities.type, 적재 CSV의 소문자 type은 적재 시 변환 |
| AdminLevel | SIDO, SIGUNGU, EMD | 행정구역 |
| OperationStatus | QUEUED, RUNNING, SUCCEEDED, FAILED | API 작업 계층 제안, predictions DB ENUM이 아님 |
| PredictionMode | LIVE, DEMO_ELMFIRE | PRED-01 예측 방식: 실서비스(ELMFIRE · 실시간 기상) / 시연 1(ELMFIRE · 의성 2025 기상). 안 보내면(null) 서버 기본 설정. 시연 2(DEMO_REPLAY)는 아직 없음. DB ENUM이 아님 (v0.5 보완) |
| ProposalItemState | GENERATED, PARTIAL, FAILED, REUSED, NOT_TRIGGERED | 제안 항목 상태: 생성/일부 생성/실패(검사를 통과한 제안 없음)/재사용(직전 회차 그대로)/해당 없음(발동 안 함) (v0.5) |
| ChangeType | NEW, CHANGED, SAME | PROP-02·03 항목의 직전 버전 대비 표시 (v0.4의 ADDED/CHANGED/UNCHANGED 대체, v0.5) |
| DiffItemChange | CHANGED, NEW, REMOVED | PROP-04 바뀐 항목: 변경/신규/빠짐 (v0.5) |
| DiffProposalChange | ADDED, REMOVED, MODIFIED | PROP-04 바뀐 제안 문단: 새 제안/없어진 제안/바뀐 제안 (v0.5) |

DB의 영문 ENUM과 규칙 엔진·LLM의 한글 표시는 구분합니다. 제안 문단의 구분은 저장값이 「즉시」 같은 한글이어도 응답에서는 `IMMEDIATE` 같은 영문 코드로 바꿔 보냅니다. **한글 그대로 보내는 값은 두 개뿐**입니다: 근거 `Evidence.origin`(판정 근거·검색), 기상 `hours[].kind`(관측·예보·고정·파일). 판정 한 줄·제안 문장·항목 이름 같은 화면 글은 한글입니다. 현재 문서에 없는 미래의 응답 문자열은 프론트에서 기본 표시로 처리할 수 있어야 합니다.

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
| deployed_incident_id | ID 또는 null | 필수·null 허용 | status=DEPLOYED면 투입된 산불, 아니면 null (DB 11/6, v0.5) |

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
  },
  "deployed_incident_id": null
}
```

<a id="dto-situation"></a>

### Situation


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| villages | object<string, VillageState> | 선택·null 불가 | 키는 villages.id. VillageState의 evacuated·evac_order·pre_evac_done은 boolean. 미기록 마을은 규칙 엔진에서 false로 해석 |
| deployed | object | 선택·null 불가 | helicopter·vehicle·crew별 투입 수. 각 묶음이 있으면 total 필수, 0 이상 정수 |
| deployed.helicopter | object | 선택 | total, forest, local, fire, military, police, unknown(미상) |
| deployed.vehicle | object | 선택 | total, forest_vehicle, fire_truck, other |
| deployed.crew | object | 선택 | total, special_crew, forest_officials, fire, military, police, other |
| disaster_sms_sent | boolean | 선택·null 불가 | 사용자가 정정한 송출 여부. E6의 읍면별 Set/Go 판정 이력과 다름 |

모르는 종류별 투입 수는 생략합니다. **기관을 모르는 투입 수는 헬기 `unknown`(미상) · 인력과 차량 `other`(기타) 칸 몫입니다(11/10).** total이 기관 칸 합보다 크면 남는 몫을 이 칸 몫으로 보며, 규칙 엔진 입력 · 정정 확인안 · COR-01 대조가 같은 규칙을 씁니다. 기관을 알게 되면 「미상 헬기 2대는 산림청」처럼 정정해 기관 칸으로 옮깁니다. 인력은 개별 자원·보유 현황으로 관리하지 않습니다.

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
| current_stage | ResponseStage | 필수 | **예측할 때 판정**한 값(PRED-01 성공 시 그 제안서의 judged_stage로 갱신). 화선 보고·정정 때는 판정하지 않으며 첫 예측 전에는 INITIAL. 사용자 입력 금지 (v0.5) |
| situation | Situation | 필수 | 현재 승인된 상황 |
| reported_by / created_at | ID / DateTime | 필수 | 최초 등록 계정 / 등록시각 |
| ended_at / ended_by | DateTime 또는 null / ID 또는 null | 필수·null 허용 | 종료 정보 |
| current_perimeter_id / current_perimeter_version | ID / integer | 필수 | 현재 화선 참조, v0 포함 |
| latest_prediction_id / latest_proposal_id | ID 또는 null | 필수·null 허용 | 성공한 최신 결과 참조, 아직 없으면 null |
| needs_reprediction | boolean | 필수 | 최신 화선 버전 > 현재 예측 기준 화선 버전. 예측이 없으면 true |
| proposal_stale | boolean | 필수 | 정정은 저장됐지만 현재 제안서가 이를 반영하지 못함 |
| situation_revision | string | 필수 | 동시 수정 방지용 불투명 API 토큰 제안. DB 컬럼 추가 확정 아님. v0.5 서버는 `rev-{최신 정정 id}` 임시 값 — 상황 정정(COR-01) 결정 때 확정(확인 필요) |
| running_operation_id | string 또는 null | 필수·null 허용 | 진행 중인 예측 작업 id(PRED-05로 조회). 없으면 null. 화면을 다시 열어도 진행 표시를 이어 감 (v0.5) |

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
  "situation_revision": "rev-example-1",
  "running_operation_id": null
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
| source | KMA_STATIONS / KMA_FORECAST / AWS_OBS / FIXED | 필수 | ELMFIRE 실시간 — 관측소 격자의 발화점 칸(v0.5 보완) / 운영 예보 / 과거 시연 관측(ELMFIRE 시연 덱 포함) / 시험 고정값(v0.5, 서버 설정 WEATHER=fixed) |
| issued_at | DateTime 또는 null | 필수·null 허용 | 예보 발표시각. 관측은 null |
| location | object 또는 null | 필수·null 허용 | 예보 {nx: integer, ny: integer}, 관측 {stn: string}, ELMFIRE 관측소 격자 · 시연 덱 {lat: number, lon: number}(발화점, v0.5 보완), 고정값은 null |
| hours | WeatherHour[9] | 필수 | base_at을 내림한 정시부터 8시간 뒤까지, 1시간 간격 9개 |
| hours[].at | DateTime | 필수 | 대상 정시 |
| hours[].kind | string | 선택 | 「관측」·「예보」·「고정」·「파일」 — 0시는 관측, 뒤는 예보처럼 섞여 오므로 시간마다 표시. **한글 그대로** (v0.5) |
| hours[].wind_dir / wind_speed | number | 필수 | 불어오는 방향 0~360° / m/s, 풍속 0 이상 |
| hours[].temp / humidity / precip | number | 필수 | ℃ / 0~100% / 1시간 강수 mm, 강수 0 이상 |

시간별 기상은 서버가 선택합니다. 기압·지면온도·시정·특보는 예측 weather에 추가하지 않습니다. 한 지점 값을 전체 범위에 적용하는 한계를 표시합니다(KMA_FORECAST). **ELMFIRE 예측(KMA_STATIONS · 시연 AWS_OBS, v0.5 보완)**은 관측소 여러 곳을 600 m 격자로 보간해 계산하고, 이 값은 그 격자의 **발화점 칸 9시간**입니다 — 규칙 판정·화면이 확산 계산과 같은 바람을 봅니다. wind_speed는 늘 평균풍속입니다(대응단계 풍속 기준과 같은 정의).

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
      "kind": "관측",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T12:00:00+09:00",
      "kind": "관측",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T13:00:00+09:00",
      "kind": "관측",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T14:00:00+09:00",
      "kind": "관측",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T15:00:00+09:00",
      "kind": "관측",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T16:00:00+09:00",
      "kind": "관측",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T17:00:00+09:00",
      "kind": "관측",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T18:00:00+09:00",
      "kind": "관측",
      "wind_dir": 270,
      "wind_speed": 2.1,
      "temp": 21.8,
      "humidity": 23,
      "precip": 0.0
    },
    {
      "at": "2025-03-22T19:00:00+09:00",
      "kind": "관측",
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
| stage | string 또는 null | 필수·null 허용 | SPREAD / RISK / RULES / PROPOSAL_GENERATION / COMMIT. v0.5 서버는 EVIDENCE를 내지 않음(근거 검색이 PROPOSAL_GENERATION 안에서 같이 돎) |
| result | object 또는 null | 필수·null 허용 | 성공 때 prediction_id·proposal_id 반환. 정정 관련 작업은 correction_id도 가능. 예측 작업(PREDICTION)은 mode(요청한 PredictionMode, null = 서버 기본)·predictor(실제로 쓴 예측 synthetic·file·elmfire)도 반환 (v0.5 보완) |
| error | object 또는 null | 필수·null 허용 | 실패 때 code·message. HTTP 오류와 같은 코드 체계. v0.5 서버 코드: 커밋 직전 재검사 `STALE_PERIMETER`·`INCIDENT_ENDED`, 계산 실패 `PREDICTION_INVALID`(예측 결과 검사 실패)·`PROPOSAL_FAILED`(규칙·제안 단계)·`INTERNAL_ERROR` — **이름 확인 필요** |

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
| attrs | object | 필수 | 종류별 속성. SHELTER: designated·used_2025·area_m2·evac_area_2025·tier(`village` 마을 시설=먼저 모일 곳 · `hub` 거점 시설=머물 곳 · null 후보 아님, v0.5), VULNERABLE: occupancy·daytime_only·emergency, HOSPITAL: emergency, FIRE_STATION: center |

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
    "tier": "hub"
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

동적으로 만든 도로·송전선 구간명은 제안 항목의 본문(sections의 목록 줄)에서 표시합니다(v0.5). DB 원본 name을 임의로 바꾸지 않습니다.

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

<a id="dto-evidence"></a>

### Evidence

> v0.5 새 DTO. v0.4의 CitationRef + EVD-01 원문을 대신합니다.

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| chunk_id | string | 필수 | 근거 청크 코드(예: SM-p073-s01). 문단의 evidence 목록과 같은 값 |
| label | string 또는 null | 필수·null 허용 | 화면 표기(「p.73」, 「시행령 제21조①·②」). 원자료에 없으면 null |
| doc_name | string 또는 null | 필수·null 허용 | 문서명. 원자료에 없으면 null |
| section | string 또는 null | 필수·null 허용 | 절·조항 |
| page | integer 또는 null | 필수·null 허용 | 쪽. 법령처럼 쪽이 없으면 null |
| origin | string 또는 null | 필수·null 허용 | 「판정 근거」(규칙 엔진이 지정) · 「검색」(근거 검색으로 찾음). **한글 그대로**, 원자료에 없으면 null |
| text | string 또는 null | 필수·null 허용 | 원문 전체. 제안서를 저장할 때의 스냅샷, 원자료에 없으면 null |

항목이 실제로 인용한 원문만 항목과 함께 저장해 돌려줍니다(DB-설계 2.4, 11/9). 근거 자료가 나중에 바뀌어도 그때 본 원문이 남습니다. 원문 코퍼스 · 검색은 GPU 서버(RAG)에 있고 이 API로 노출하지 않습니다.

**DTO Example**

```json
{
  "chunk_id": "SM-p073-s01",
  "label": "p.73",
  "doc_name": "「산불 재난」 위기관리 표준매뉴얼",
  "section": "대응단계 판단기준",
  "page": 73,
  "origin": "판정 근거",
  "text": "(근거 원문 전체)"
}
```

<a id="dto-proposalsection"></a>

### ProposalSection

> v0.5 새 DTO. 항목 카드의 본문 = 소제목(`title`) → 블록(`blocks[]`). 블록은 `type`마다 칸이 다릅니다.

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| title | string | 필수 | 소제목(대부분 규칙 엔진이 정함, 버전마다 구성이 다를 수 있음) |
| blocks[].type | string | 필수 | paragraph(제안 문단) · list(목록) · table(표) · note(※ 안내) · failed(실패 안내) |
| paragraph: category | ItemCategory | 필수 | 문단 구분(저장값 한글을 영문 코드로) |
| paragraph: body | string | 필수 | 본문 = 상황 문장 + 할 일 문장, 근거 표기 없음 |
| paragraph: cite | string | 필수 | 근거 표기(「[p.73·91]」) |
| paragraph: quote | string | 필수 | 근거 문장 — Evidence.text에서 칠할 부분 |
| paragraph: evidence | string[] | 필수 | 인용한 Evidence.chunk_id 목록 |
| paragraph: text / situation / action_text | string | 선택 | 본문 + 표기 / 판정 상황 문장(규칙) / 할 일 문장(LLM) — 상황은 회색 · 할 일은 굵게처럼 꾸밀 때 |
| paragraph: topic / source | string 또는 null / string | 선택 | 다룬 주제 / `rule+llm`(상황 문장 있음) · `llm` |
| list: style / rows[].text | string / string | 필수 | `bullet` · `numbered` / 한 줄 글, `**굵게**` 표기 포함 |
| table: head / rows | string[] / string[][] | 필수 | 열 이름 / 행마다 열 순서 배열 |
| note · failed: text | string | 필수 | ※ 안내 / 「검사를 통과한 제안이 없어 이 항목의 제안은 표시하지 않았습니다」 |

**DTO Example**

```json
{
  "title": "1. 진화 우선순위",
  "blocks": [
    {
      "type": "list",
      "style": "numbered",
      "rows": [
        {
          "text": "① **인명**: 지금(이미 닿음) 안평면 괴산리·신월리"
        }
      ]
    },
    {
      "type": "note",
      "text": "※ 군사시설과 중요 산림자원은 위치 정보가 없어 반영하지 않았습니다."
    }
  ]
}
```

<a id="dto-proposalitem"></a>

### ProposalItem

> v0.5에서 다시 정함. v0.4의 `id` · `text` · `detail`(RuleDetail) · `citations`는 뺐습니다.

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| item_code | string | 필수 | S1~S6 · E1~E6 (12개, 아래 표) |
| name | string | 필수 | 항목 이름 |
| axis | ItemAxis | 필수 | 진화/대피 |
| state | ProposalItemState | 필수 | 생성 · 일부 생성 · 실패 · 재사용 · 해당 없음 |
| categories | ItemCategory[] | 필수 | 문단 구분을 중복 없이 즉시→대기→협의→요청 순서로. 실패 · 해당 없음이면 [] |
| finding | string | 필수 | 판정 한 줄(규칙 엔진). S1의 값이 격상 대비 안내 내용. 해당 없음이면 "" |
| withheld | {field, reason}[] 또는 null | 선택·null 허용 | 엔진이 판단하지 않은 것(자료 · 기준 없음). 서버 기본값 null도 처리 |
| change_type | ChangeType 또는 null | 필수·null 허용 | 바로 앞 버전의 같은 항목과 비교한 값(아래 「변경」 판단). 첫 버전 · 해당 없음이면 null |
| sections | ProposalSection[] | 필수 | 카드 본문 |
| evidence | Evidence[] | 필수 | 이 항목이 인용한 근거 원문 |

제안서는 **늘 12칸**(S1→S6, E1→E6)입니다. 발동하지 않은 항목도 칸이 있고 `state: NOT_TRIGGERED`, 나머지는 빈 값입니다. 미발동을 안전하다는 의미로 해석하지 않습니다. 규칙 판정의 facts · targets · status는 응답에 넣지 않습니다(지도 강조는 MVP 제외).

| 진화 | 이름 | 대피 | 이름 |
| --- | --- | --- | --- |
| S1 | 대응단계 판단·격상 검토·지휘권 인계 | E1 | 위험구역 설정 |
| S2 | 진화 우선지역·보호대상·시설 보호 | E2 | 대피 대상·순서·시점 |
| S3 | 진화자원 현황·동원 범위 | E3 | 취약시설 조치 |
| S4 | 진화 방식·시간대 전략 | E4 | 대피소 지정 및 관련 통제 |
| S5 | 진화인력 안전·투입 관리 | E5 | 재난문자·방송 전파 |
| S6 | 유관기관 동원·협조 요청 | E6 | 유관기관 요청 |

**「변경」 판단** — PROP-01 `changed_count`, PROP-02·03 `change_type`, PROP-04가 같은 규칙을 씁니다.

- 변경(CHANGED): 같은 항목 코드끼리 판정 한 줄 · 구분 · 다룬 주제(topic) 집합 · 목록/표/※ 줄 · 문단 상황 문장 중 하나라도 다름. 「직전 …」처럼 직전 버전을 가리키는 표기는 지우고 비교
- LLM 할 일 문장(action_text)만 다르면 변경이 아님 → SAME(PROP-04에서는 wording_only)
- NEW · REMOVED: 앞 버전에 없던 항목 / 이번에 해당 없음이 된 항목

**DTO Example**

```json
{
  "item_code": "S1",
  "name": "대응단계 판단·격상 검토·지휘권 인계",
  "axis": "SUPPRESSION",
  "state": "GENERATED",
  "categories": [
    "STANDBY",
    "CONSULT"
  ],
  "finding": "현재 기준 1단계(피해면적 48.5 ha · 평균풍속 3.4 m/s · 시설피해 우려 최소 2동) · P5 기준 2단계(피해면적 363 ha) → 5시간 안에 2단계 기준 도달 예상 · 지휘권자 의성군수 → 경상북도지사",
  "withheld": [
    {
      "field": "진화예상시간",
      "reason": "MVP에서 제외"
    }
  ],
  "change_type": "CHANGED",
  "sections": [
    {
      "title": "1. 대응단계 판단·격상 검토",
      "blocks": [
        {
          "type": "paragraph",
          "category": "CONSULT",
          "body": "피해면적 48.5 ha, 평균풍속 3.4 m/s, 시설피해 우려 최소 2동으로 세 기준 모두 확산대응 1단계에 해당합니다. 확산대응 1단계 발령을 산림청장과 협의하십시오.",
          "cite": "[p.73·91]",
          "quote": "산림청장은 대응단계 판단기준에 따라 산불현장 통합지휘본부와 협의하여 대응단계를 발령한다",
          "evidence": [
            "SM-p073-s01",
            "SM-p091-s02"
          ],
          "text": "피해면적 48.5 ha, … 확산대응 1단계 발령을 산림청장과 협의하십시오 [p.73·91].",
          "situation": "피해면적 48.5 ha, 평균풍속 3.4 m/s, 시설피해 우려 최소 2동으로 세 기준 모두 확산대응 1단계에 해당합니다.",
          "action_text": "확산대응 1단계 발령을 산림청장과 협의하십시오.",
          "topic": "현재 기준 단계",
          "source": "rule+llm"
        }
      ]
    }
  ],
  "evidence": [
    {
      "chunk_id": "SM-p073-s01",
      "label": "p.73",
      "doc_name": "「산불 재난」 위기관리 표준매뉴얼",
      "section": "대응단계 판단기준",
      "page": 73,
      "origin": "판정 근거",
      "text": "(근거 원문 전체)"
    }
  ]
}
```

<a id="dto-proposal"></a>

### Proposal

> v0.5에서 다시 정함. v0.4의 incident_id · perimeter_version · seq · correction_id · proposal_stale는 뺐습니다(버전 문자열 · 산불 상세에 있음).

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | 제안서 |
| version | string | 필수 | 표시용 v{기준 화선 버전}.{같은 화선 내 순번} |
| trigger | PREDICTION / CORRECTION | 필수 | 생성 계기 |
| created_at | DateTime | 필수 | 생성시각 |
| base_at | DateTime | 필수 | 예측 기준 시각(기준 화선 관측 시각 = P0). 「예측 기준 12:50 · 28분 전」 표시용 |
| prediction_id | ID | 필수 | 이 제안서가 쓴 예측. PRED-02·03·04 · RISK-01을 같은 번호로 부름 |
| judged_stage / predicted_stage | ResponseStage | 필수 | 이 예측으로 판정한 현재 기준 / P5 · 0~5h 평균풍속 기준 |
| items | ProposalItem[12] | 필수 | 늘 12칸, S1→S6 · E1→E6 |

official_stage는 두지 않습니다. 예측 단계가 현재 판정보다 높으면 격상 대비 안내이며 공식 발령으로 표시하지 않습니다. 아래 items 예시는 형식 설명용 2칸입니다.

**DTO Example**

```json
{
  "id": "71",
  "version": "v1.2",
  "trigger": "PREDICTION",
  "created_at": "2025-03-22T12:53:00+09:00",
  "base_at": "2025-03-22T12:50:00+09:00",
  "prediction_id": "41",
  "judged_stage": "STAGE1",
  "predicted_stage": "STAGE2",
  "items": [
    {
      "item_code": "S1",
      "name": "대응단계 판단·격상 검토·지휘권 인계",
      "axis": "SUPPRESSION",
      "state": "GENERATED",
      "categories": [
        "STANDBY",
        "CONSULT"
      ],
      "finding": "현재 기준 1단계(피해면적 48.5 ha · 평균풍속 3.4 m/s · 시설피해 우려 최소 2동) · P5 기준 2단계(피해면적 363 ha) → 5시간 안에 2단계 기준 도달 예상 · 지휘권자 의성군수 → 경상북도지사",
      "withheld": [
        {
          "field": "진화예상시간",
          "reason": "MVP에서 제외"
        }
      ],
      "change_type": "CHANGED",
      "sections": [
        {
          "title": "1. 대응단계 판단·격상 검토",
          "blocks": [
            {
              "type": "paragraph",
              "category": "CONSULT",
              "body": "피해면적 48.5 ha, 평균풍속 3.4 m/s, 시설피해 우려 최소 2동으로 세 기준 모두 확산대응 1단계에 해당합니다. 확산대응 1단계 발령을 산림청장과 협의하십시오.",
              "cite": "[p.73·91]",
              "quote": "산림청장은 대응단계 판단기준에 따라 산불현장 통합지휘본부와 협의하여 대응단계를 발령한다",
              "evidence": [
                "SM-p073-s01",
                "SM-p091-s02"
              ],
              "text": "피해면적 48.5 ha, … 확산대응 1단계 발령을 산림청장과 협의하십시오 [p.73·91].",
              "situation": "피해면적 48.5 ha, 평균풍속 3.4 m/s, 시설피해 우려 최소 2동으로 세 기준 모두 확산대응 1단계에 해당합니다.",
              "action_text": "확산대응 1단계 발령을 산림청장과 협의하십시오.",
              "topic": "현재 기준 단계",
              "source": "rule+llm"
            }
          ]
        }
      ],
      "evidence": [
        {
          "chunk_id": "SM-p073-s01",
          "label": "p.73",
          "doc_name": "「산불 재난」 위기관리 표준매뉴얼",
          "section": "대응단계 판단기준",
          "page": 73,
          "origin": "판정 근거",
          "text": "(근거 원문 전체)"
        }
      ]
    },
    {
      "item_code": "E3",
      "name": "취약시설 조치",
      "axis": "EVACUATION",
      "state": "NOT_TRIGGERED",
      "categories": [],
      "finding": "",
      "withheld": [],
      "change_type": null,
      "sections": [],
      "evidence": []
    }
  ]
}
```

<a id="dto-proposalversion"></a>

### ProposalVersion

> v0.5 새 DTO. PROP-01 버전 목록의 한 줄.

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | ID | 필수 | 제안서(PROP-03 · 04에 씀) |
| version / trigger / created_at | string / PREDICTION · CORRECTION / DateTime | 필수 | Proposal과 같음 |
| judged_stage / predicted_stage | ResponseStage | 필수 | Proposal과 같음 |
| category_counts | object | 필수 | {IMMEDIATE, STANDBY, CONSULT, REQUEST: integer} 구분별 항목 수(한 항목이 여러 구분이면 각각 셈) |
| changed_count | integer 또는 null | 필수·null 허용 | 바로 앞 버전 대비 바뀐 항목 수(「변경」 판단), 첫 버전은 null |

**DTO Example**

```json
{
  "id": "71",
  "version": "v1.2",
  "trigger": "PREDICTION",
  "created_at": "2025-03-22T12:53:00+09:00",
  "judged_stage": "STAGE1",
  "predicted_stage": "STAGE2",
  "category_counts": {
    "IMMEDIATE": 9,
    "STANDBY": 2,
    "CONSULT": 2,
    "REQUEST": 3
  },
  "changed_count": 2
}
```

<a id="dto-proposaldiff"></a>

### ProposalDiff

> v0.5 새 DTO. PROP-04 직전 버전 비교 — 바뀐 항목, 그 안에서도 바뀐 부분만.

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| version / previous_version | string / string 또는 null | 필수 | 이 버전 / 바로 앞 버전. 첫 버전이면 null |
| trigger / created_at | PREDICTION · CORRECTION / DateTime | 필수 | 이 버전 |
| cause | string 또는 null | 필수·null 허용 | 왜 바뀌었나: 「상황 정정: {입력}」 · 「새 화선(v2, 관측 15:00) 보고로 다시 예측」 · 「같은 화선에서 다시 예측(예측 기준 15:00)」. 첫 버전 null |
| counts | {changed, wording_only, same} 또는 null | 필수·null 허용 | 바뀐 항목 · 표현만 바뀐 항목 · 같은 항목 수. changed = PROP-01 changed_count |
| items[] | object[] | 필수 | 바뀐 항목만(S1→E6). 첫 버전이면 [] |
| items[].item_code / name | string | 필수 | |
| items[].change | DiffItemChange | 필수 | 변경 · 신규 · 빠짐 |
| items[].categories | {before, after} | 필수 | ItemCategory[]. 신규면 before null, 빠짐이면 after null |
| items[].finding | {before, after} 또는 null | 필수·null 허용 | 판정 한 줄. 같으면 null |
| items[].proposals[] | object[] | 필수 | 바뀐 제안 문단만: change(DiffProposalChange) · topic · before{category, body} · after{category, body}(없는 쪽은 null). 주제가 같은 문단끼리 짝 |
| items[].rows | {added, removed} | 필수 | 목록 · 표 · ※ 줄의 + / − (string[]) |
| items[].note | string 또는 null | 필수·null 허용 | 이번 버전에서 실패했으면 그 안내 |
| wording_only_items / same_items | string[] | 필수 | 표현만 바뀐 · 같은 항목 코드 |

바뀐 낱말 칠하기는 하지 않고 이전 · 이후 문장을 통째로 줍니다(LLM이 표현을 바꾸면 칠하기가 소음).

**DTO Example**

```json
{
  "version": "v2.2",
  "previous_version": "v2.1",
  "trigger": "CORRECTION",
  "created_at": "2025-03-22T15:40:05+09:00",
  "cause": "상황 정정: 박곡리 대피 완료, 헬기 6대 투입",
  "counts": {
    "changed": 3,
    "wording_only": 2,
    "same": 7
  },
  "items": [
    {
      "item_code": "S3",
      "name": "진화자원 현황·동원 범위",
      "change": "CHANGED",
      "categories": {
        "before": [
          "IMMEDIATE",
          "STANDBY",
          "REQUEST"
        ],
        "after": [
          "IMMEDIATE",
          "STANDBY",
          "REQUEST"
        ]
      },
      "finding": {
        "before": "투입: 헬기 23 · 장비 26 · 인력 151 / 현재 1단계 → 5시간 뒤 2단계",
        "after": "투입: 헬기 29 · 장비 26 · 인력 151 / 현재 1단계 → 5시간 뒤 2단계"
      },
      "proposals": [
        {
          "change": "MODIFIED",
          "topic": "투입 자원",
          "before": {
            "category": "IMMEDIATE",
            "body": "현재 헬기 23대(산림청 5, 지자체 16, 소방 2)가 투입되어 있습니다. 가용 헬기를 집중 투입하십시오."
          },
          "after": {
            "category": "IMMEDIATE",
            "body": "현재 헬기 29대(산림청 9, 지자체 16, 소방 4)가 투입되어 있습니다. 가용 헬기를 집중 투입하십시오."
          }
        }
      ],
      "rows": {
        "added": [],
        "removed": []
      },
      "note": null
    }
  ],
  "wording_only_items": [
    "S1",
    "E5"
  ],
  "same_items": [
    "S4",
    "S5",
    "S6",
    "E1",
    "E3",
    "E4",
    "E6"
  ]
}
```

<a id="dto-citationref"></a>

### CitationRef

> **v0.5 보완(11/10, 서버 구현대로)**: AI-01 답변(QaMessage.citations)만 씁니다. 제안서는 Evidence를 씁니다. DB에 citations 테이블이 없어 `qa_messages.evidence`에 답변이 인용한 원문을 같은 행으로 저장하고(DB-설계 2.5), Evidence 칸에 답변 속 번호 `k`와 `id`를 더한 모양입니다.

| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| id | string | 필수 | 「질의응답 id-k」(예: 111-1). 별도 테이블 PK가 아님 |
| k | integer | 필수 | 답변 문장의 [k], 1 이상. 같은 표기(같은 쪽)의 조각 여러 개는 번호 하나로 묶음 |
| chunk_id / chunk_code | string | 필수 | 근거 청크 코드(예: SM-p073-s01). 둘은 같은 값(v0.4 호환) |
| label | string 또는 null | 필수·null 허용 | 화면 표기(「p.73」, 「시행령 제21조①·②」) |
| doc_name | string 또는 null | 필수·null 허용 | 문서명 |
| page | integer 또는 null | 필수·null 허용 | 법령처럼 쪽 없으면 null |
| section | string 또는 null | 필수·null 허용 | 항목·조항 |
| origin | string 또는 null | 필수·null 허용 | Evidence.origin과 같음: 「판정 근거」(규칙 엔진이 지정) · 「검색」(근거 검색으로 찾음). **한글 그대로** |
| text | string 또는 null | 필수·null 허용 | 원문(묶인 조각은 「…」로 이어 붙임). 답변할 때의 스냅샷 |

**DTO Example**

```json
{
  "id": "111-1",
  "k": 1,
  "chunk_id": "SM-p073-s01",
  "chunk_code": "SM-p073-s01",
  "label": "p.73",
  "doc_name": "「산불 재난」 위기관리 표준매뉴얼",
  "page": 73,
  "section": "대응단계 판단기준",
  "origin": "판정 근거",
  "text": "(근거 원문)"
}
```

<a id="dto-correctionchange"></a>

### CorrectionChange


| 필드 | 타입 | 필수·null | 설명 |
| --- | --- | --- | --- |
| target | village / deployed / incident / situation | 필수 | API 제안 구분 |
| target_id | ID | village일 때 필수 | villages.id, 다른 target은 생략 |
| item | string | 필수 | village: evacuated·evac_order·pre_evac_done, deployed: `helicopter.total`·`helicopter.forest` 등 Situation.deployed의 묶음.칸(헬기 `unknown` 미상 포함), situation: disaster_sms_sent (위기경보 정정 항목은 v0.5에서 삭제) |
| before / after | boolean 또는 integer | 필수 | 허용 항목과 동일 타입. 서버 현재값과 before 대조(미상·기타 칸은 total의 남는 몫 포함) |
| label | string | 필수 | 화면에 보일 한 줄(예: 「투입 헬기(산림청) 5 → 7대」, 「안평면 괴산리 대피 완료 예(전: 아니오)」). 서버가 만듦 |
| target_name | string | village일 때 | 「읍면 마을」 이름(예: 안평면 괴산리) |

current_stage·official_stage·기상·진화예상시간·부상·고립·자원 개별 상태는 정정 허용 목록에 없습니다. 투입 칸 하나를 바꾸면 같은 묶음의 `total` 변경도 함께 옵니다. 칸만 옮기는 정정(「미상 헬기 2대는 산림청」)은 total이 그대로라 total 줄이 없습니다.

**DTO Example**

```json
{
  "target": "village",
  "target_id": "1024",
  "item": "evacuated",
  "before": false,
  "after": true,
  "label": "안평면 괴산리 대피 완료 예(전: 아니오)",
  "target_name": "안평면 괴산리"
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
| ignored | object[] | 필수 | 반영하지 않은 말과 이유 {text, reason}. 예: 「대응단계는 정정할 수 없음」, 「자원 현황의 가용 대수를 넘음」, 「이미 그렇게 기록됨」. 없으면 [] |
| notes | string[] | 필수 | 확인 카드에 보일 안내. 예: 「헬기 기관을 말하지 않아 「미상」 칸에 넣었습니다(기관을 알면 「미상 헬기 2대는 산림청」처럼 다시 정정해 주십시오).」 없으면 [] |
| summary | string | 필수 | 확인 카드 머리 문장(변경 건수 + label 목록, total 줄은 세지 않음) |
| proposal_regenerates | boolean | 필수 | 확인하면 제안서를 다시 만드는지(예측이 있으면 true) |

확인안을 받는 것만으로 DB 상황·정정 이력은 바뀌지 않습니다. 취소 시 저장 API를 호출하지 않습니다.

**DTO Example**

```json
{
  "type": "CORRECTION_PREVIEW",
  "confirmation_token": "EXAMPLE_PREVIEW_TOKEN",
  "expires_at": "2025-03-22T12:30:00+09:00",
  "situation_revision": "rev-example-1",
  "input_text": "시연리 대피 완료, 2단계 발령됐어",
  "changes": [
    {
      "target": "village",
      "target_id": "1024",
      "item": "evacuated",
      "before": false,
      "after": true,
      "label": "안평면 시연리 대피 완료 예(전: 아니오)",
      "target_name": "안평면 시연리"
    }
  ],
  "ignored": [
    {
      "text": "2단계 발령됐어",
      "reason": "대응단계는 시스템이 화선 면적 · 풍속 · 시설피해로 판정하므로 정정할 수 없습니다. 새 화선을 보고하면 다시 판정됩니다."
    }
  ],
  "notes": [],
  "summary": "변경 1건을 확인하면 상황에 저장하고 제안서를 다시 만듭니다: 안평면 시연리 대피 완료 예(전: 아니오)",
  "proposal_regenerates": true
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
| route | string 또는 null | 필수·null 허용 | 답한 경로: STATUS(지금 상황) · PROPOSAL(제안서 항목) · REGULATION(규정 · 기준) · OUT_OF_SCOPE · GUIDE(사용 안내) |
| sentences | object[] | 필수 | 답변을 문장 단위로. {text: 문장(끝에 [k]), citations: integer[](그 문장의 k), data_refs: {ref, label, value}[](그 문장이 쓴 상황 값 — ref는 situation.deployed.helicopter.total처럼 화면이 강조할 경로)}. 모든 문장은 citations 또는 data_refs가 하나 이상 |
| insufficient | string[] | 필수 | 질문이 물었는데 자료 · 근거에서 찾지 못한 것(최대 2개). 답변 끝에 「…는 찾지 못해 답하지 않았습니다」로도 붙음 |
| no_evidence | boolean | 필수 | 답할 자료가 전혀 없어 근거 부족 안내만 했으면 true |
| created_at | DateTime | 필수 | 저장시각 |

질문·답변 1건이 1행입니다. 이전 질문을 자동 문맥으로 이어 받지 않습니다. 상황은 변경하지 않습니다. 답변의 숫자 · 시각 · 마을 이름은 그 문장의 출처(citations · data_refs)에 있는 것만 씁니다(서버가 검사).

**DTO Example**

```json
{
  "id": "111",
  "incident_id": "100",
  "user_id": "1",
  "proposal_id": "71",
  "item_code": null,
  "question": "지금 헬기 몇 대 투입됐어?",
  "answer": "지금 투입된 헬기는 23대(산림청 5 · 지자체 16 · 소방 2)입니다.",
  "citations": [],
  "route": "STATUS",
  "sentences": [
    {
      "text": "지금 투입된 헬기는 23대(산림청 5 · 지자체 16 · 소방 2)입니다.",
      "citations": [],
      "data_refs": [
        {
          "ref": "situation.deployed.helicopter.total",
          "label": "투입된 헬기(합계)",
          "value": "23대(산림청 5 · 지자체 16 · 소방 2)"
        }
      ]
    }
  ],
  "insufficient": [],
  "no_evidence": false,
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

총 **46개**입니다. 기존 42개 API의 기능 구분을 유지하고, 최신 DB에 정의된 기지 관리 `BASE-01~04` 4개를 추가했습니다. **v0.5**: EVD-01(근거 원문은 PROP-02 응답 안으로) 삭제, WTH-01(지금 날씨) 추가로 개수는 그대로 46개입니다. **모든 경로·DTO는 백엔드와 검토할 API 설계 초안**이며 RISK-01은 저장·계산 합의가 필요한 조건부 API입니다.


| API ID | 기능 | Method | Endpoint | 권한 |
| --- | --- | --- | --- | --- |
| [AUTH-01](#auth-01) | 로그인 | POST | `/api/v1/auth/login` | 비로그인 |
| [AUTH-02](#auth-02) | 내 계정·권한 조회 | GET | `/api/v1/auth/me` | COMMANDER, VIEWER, REPORTER, ADMIN |
| [AUTH-03](#auth-03) | 로그아웃 기록 | POST | `/api/v1/auth/logout` | COMMANDER, VIEWER, REPORTER, ADMIN |
| [INC-01](#inc-01) | 산불 목록 | GET | `/api/v1/incidents` | COMMANDER, VIEWER, REPORTER, ADMIN |
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
| [WTH-01](#wth-01) | 지금 날씨 (v0.5 추가) | GET | `/api/v1/incidents/{incident_id}/weather/now` | COMMANDER, VIEWER |
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
| 권한 | COMMANDER, VIEWER, REPORTER, ADMIN |
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

**처리 규칙:** users의 관할 시군구 내 산불만, reported_at DESC, id DESC로 조회합니다. ADMIN은 관할이 없어 전체 지역을 봅니다(자원을 「투입」으로 바꿀 때 진행 중 산불 선택, v0.5). 페이지 이동 중 변동 시 중복을 id로 제거합니다.

**주요 오류:** 공통 401·403·404·422·500 오류를 적용합니다.

<a id="inc-02"></a>

### INC-02 · 새 산불 보고


| 항목 | 내용 |
| --- | --- |
| Method·URL | `POST /api/v1/incidents` |
| 권한 | REPORTER |
| 성공 상태 | 201 |
| 근거 | [DB-설계.md · 2.2](https://github.com/wildfire-guard-capstone/wfg-docs/blob/43dc969703cdb65192e07e1341785ce38bca60c6/docs/설계/DB-설계.md) |

**추가 Header:** `Idempotency-Key` 필수 제안. v0.5 서버는 키 없이도 받습니다(이때 중복을 막지 않음) — 처리 규칙은 2.4절.

**Path parameter:** 없음

**Query parameter**

없음.

**Request Body**

ignition_point: Point, address: string(1~200), reported_at: DateTime, reporter_name: string(1~50), reporter_phone: string(1~20)은 필수(v0.5: alert_level 삭제 — 보내면 422 VALIDATION_ERROR). started_at·report_text·reporter_org는 생략/null 허용. initial_perimeter: {geometry: Polygon 또는 MultiPolygon(MultiPolygon으로 저장), observed_at: DateTime} 선택

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
  "reporter_phone": "미상"
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
    "current_stage": "INITIAL",
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

**처리 규칙:** 발화점으로 읍면동·관할을 서버가 계산합니다. 산불과 화선 v0을 같은 트랜잭션에서 저장합니다. initial_perimeter가 있으면 v1도 함께 저장합니다. status·current_stage·reported_by·기관별 접수 배열은 입력받지 않습니다. **v0.5: 보고 때 대응단계를 판정하지 않습니다** — current_stage는 INITIAL로 시작하고 예측(PRED-01) 때 판정합니다. 같은 키 재전송 처리는 2.4절.

**주요 오류:** 409 IDEMPOTENCY_KEY_REUSED, 422 INVALID_GEOMETRY / OUTSIDE_JURISDICTION / VALIDATION_ERROR(alert_level 등 모르는 칸)

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
    "situation_revision": "rev-example-1",
    "running_operation_id": null
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
    "situation_revision": "rev-example-1",
    "running_operation_id": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 종료 후 새 화선 보고·정정, 예측, 상황 정정, 제안 재생성을 막습니다. 저장 중인 예측 작업과 같은 산불별 잠금으로 순서를 보장하고 종료 전 대기 작업은 커밋 직전 재검사합니다. 종료 후 기존 제안 조회와 질문은 허용합니다. **v0.5 결정**: 종료하면 그 산불에 투입된 자원(resources.deployed_incident_id)을 자동으로 풀어 줍니다(status AVAILABLE, 연결 null) — 서버 반영 예정.

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

geometry: Polygon 또는 MultiPolygon(MultiPolygon으로 저장), observed_at: DateTime, expected_perimeter_id: ID 모두 필수

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

**처리 규칙:** 다음 version_no로 INSERT합니다. **v0.5: 대응단계는 판정하지 않습니다**(다음 예측 때 판정). 예측은 자동 실행하지 않습니다. 구멍·자기 교차·빈 도형을 검증하며 새 실측 화선은 이전 예측 범위를 포함할 의무가 없습니다. 관측 시각이 현재 화선보다 앞서거나 5분 넘게 미래면 v0.5 서버는 422 INVALID_OBSERVED_AT으로 거절합니다(확인 필요, Q-02).

**주요 오류:** 409 INCIDENT_ENDED / STALE_PERIMETER, 422 INVALID_GEOMETRY / INVALID_OBSERVED_AT(확인 필요)

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

geometry: Polygon 또는 MultiPolygon(MultiPolygon으로 저장), observed_at: DateTime 모두 필수

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

**처리 규칙:** URL의 perimeter_id가 현재 유효한 v1 이상이어야 합니다. 기존 행 corrected_at과 다음 번호 새 행을 한 트랜잭션으로 처리합니다(v0.5: 대응단계는 다음 예측 때 판정). 정정한 화선의 관측 시각이 직전 화선보다 앞서면 422 INVALID_OBSERVED_AT(확인 필요). v0 정정은 금지합니다. 과거 예측과 제안은 보존하고 이력에서 기준 화선 정정 여부를 표시합니다.

**주요 오류:** 409 INCIDENT_ENDED / STALE_PERIMETER, 422 V0_NOT_CORRECTABLE / INVALID_OBSERVED_AT(확인 필요)

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

expected_perimeter_id: ID 필수, mode: PredictionMode 또는 null 선택(v0.5 보완)

```json
{
  "expected_perimeter_id": "21",
  "mode": "LIVE"
}
```

| mode | 예측 | 기상 | Weather.source |
| --- | --- | --- | --- |
| 없음 · null | 서버 기본 설정(PREDICTOR). 지금 EC2는 가짜 타원(synthetic) | 서버 설정의 원천(WEATHER) | 설정에 따름 |
| LIVE | ELMFIRE | 실시간 관측소(매분자료 + 단기예보) → 600 m 격자 | KMA_STATIONS |
| DEMO_ELMFIRE | ELMFIRE(의성 덱) | 덱에 든 2025-03-22 관측 기상, 기온·습도는 ASOS | AWS_OBS |

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

**처리 규칙:** **v0.5 보완 — mode**: 예측마다 고르며 서버 기본값을 바꾸지 않습니다. 고른 mode와 실제로 쓴 예측은 PRED-05 result.mode·predictor와 이벤트 로그에 남고 DB 칸은 없습니다(PRED-02에는 없음). LIVE는 관측소가 3곳 미만이면(기상청 응답 없음 포함), DEMO_ELMFIRE는 화선 시각이 덱 기상(2025-03-22)에 없으면 503 WEATHER_UNAVAILABLE입니다. 최신 유효 화선만 대상으로 하며 과거 화선 지정 실행은 허용하지 않습니다. 서버가 기상 지점을 선택하고 base_at=observed_at으로 고정합니다. **v0.5: 기상은 요청 때 먼저 받습니다**(서버 설정의 원천에서 9시간). 못 받으면 작업을 만들지 않고 503 WEATHER_UNAVAILABLE이며, 받은 기상은 예측에 넘기고 predictions.weather로 저장해 규칙·제안도 같은 값을 씁니다. **ELMFIRE 예측(v0.5 보완)**도 요청 때 기상 입력(관측소 격자 — 실시간 또는 시연)을 만들고, 남은 관측소가 3곳 미만이거나 시연 기상에 없는 시각이면 같은 503입니다(다른 기상으로 대신하지 않음). 이때 predictions.weather = 그 격자의 발화점 칸 9시간. 예측→P1~P8→위험도→규칙→근거→제안 계산 후 최종 결과를 한 트랜잭션으로 저장합니다. 실패 시 직전 예측·제안이 유지됩니다. 계산 중 새 화선·정정 발생 시 오래된 결과를 최신으로 커밋하지 않도록 재검사합니다. **v0.5 결정(Q-01 일부)**: 위험도를 못 구해도 예측·제안은 저장합니다(RISK-01만 data: null). 근거 검색은 제안 작성(PROPOSAL_GENERATION) 안에서 함께 합니다. 작업 저장소는 Q-05에서 확정합니다(v0.5 서버는 메모리).

**주요 오류:** 409 INCIDENT_ENDED / PREDICTION_RUNNING / STALE_PERIMETER, 422 VALIDATION_ERROR(모르는 mode), 503 WEATHER_UNAVAILABLE

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
        "kind": "관측",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T12:00:00+09:00",
        "kind": "관측",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T13:00:00+09:00",
        "kind": "관측",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T14:00:00+09:00",
        "kind": "관측",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T15:00:00+09:00",
        "kind": "관측",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T16:00:00+09:00",
        "kind": "관측",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T17:00:00+09:00",
        "kind": "관측",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T18:00:00+09:00",
        "kind": "관측",
        "wind_dir": 270,
        "wind_speed": 2.1,
        "temp": 21.8,
        "humidity": 23,
        "precip": 0.0
      },
      {
        "at": "2025-03-22T19:00:00+09:00",
        "kind": "관측",
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

**처리 규칙:** 현재 날씨를 새로 수신한 결과가 아니라 해당 예측에 저장된 스냅샷을 반환합니다(지금 날씨는 WTH-01). v0.5: 시간마다 `kind`(관측·예보·고정·파일)가 있고, 서버 설정이 시험 고정값이면 source=FIXED · issued_at · location이 null입니다.

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

성공(SUCCEEDED) 예 — 예측 작업(v0.5 보완)

```json
{
  "data": {
    "id": "op-example",
    "kind": "PREDICTION",
    "status": "SUCCEEDED",
    "stage": "COMMIT",
    "result": {
      "prediction_id": "41",
      "proposal_id": "52",
      "mode": "LIVE",
      "predictor": "elmfire"
    },
    "error": null
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 2초 간격 조회를 제안합니다. 완료는 SUCCEEDED/FAILED로 끝나며 HTTP 200 응답 안에서 작업 실패를 구분합니다. operation은 요청 사용자·산불에 결합하고 보존기간은 Q-05에서 확정합니다. v0.5 서버의 FAILED error.code는 Operation DTO 참고(이름 확인 필요). ELMFIRE(LIVE·DEMO_ELMFIRE)는 화선이 덱 범위 밖 등 입력 문제면 PREDICTION_INVALID, 시간 초과(5분)·실행 실패·실행 컨테이너 연결 실패면 INTERNAL_ERROR입니다(v0.5 보완). 작업 기록이 서버 메모리라 서버를 다시 켜면 진행 중이던 작업도 404입니다(Q-05).

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

<a id="wth-01"></a>

### WTH-01 · 지금 날씨

> v0.5 새 API. 산불현황의 「기상」 칸. 예측에 쓴 기상(PRED-04)과 따로, 지금 관측 1개를 줍니다.

| 항목 | 내용 |
| --- | --- |
| Method·URL | `GET /api/v1/incidents/{incident_id}/weather/now` |
| 권한 | COMMANDER, VIEWER |
| 성공 상태 | 200 |
| 근거 | [DB-설계.md · 2.3 weather_hours](https://github.com/wildfire-guard-capstone/wfg-docs/blob/main/docs/설계/DB-설계.md) |

**Path parameter:** `incident_id`: ID

**Query parameter**

없음.

**Request Body**

없음.

**Response**

`data`: {observed_at: DateTime, source: string, wind_dir: number 또는 null, wind_dir_name: string 또는 null, wind_speed / temp / humidity / precip: number 또는 null, next_refresh_at: DateTime 또는 null, stale: boolean}

```json
{
  "data": {
    "observed_at": "2025-03-22T14:00:00+09:00",
    "source": "KMA_NCST",
    "wind_dir": 135.0,
    "wind_dir_name": "남동",
    "wind_speed": 3.4,
    "temp": 18.2,
    "humidity": 35.0,
    "precip": 0.0,
    "next_refresh_at": "2025-03-22T15:40:00+09:00",
    "stale": false
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 발화점의 기상청 격자에서 최신 정시(지금 − 40분을 내림)의 초단기실황 관측을 줍니다. 같은 격자·같은 정시 관측은 서버가 weather_hours에 저장해 재사용하므로 기상청은 1시간에 1번만 부릅니다. `next_refresh_at`은 다음 관측이 나오는 때(정시 + 1시간 40분)이며, 기상청을 못 부르면 마지막 관측을 `stale: true`로 줍니다. 실시간 원천이 아니거나(서버 설정 WEATHER ≠ kma) 종료된 산불이면 최신 예측의 0시 값을 주고 `next_refresh_at`은 null입니다. `wind_dir_name`은 16방위(「남동」)입니다.

**주요 오류:** 503 WEATHER_UNAVAILABLE (보여 줄 관측도 예측도 없을 때)

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
        },
        "deployed_incident_id": null
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

**처리 규칙:** 전체 보유·개별 상태와 산불 투입 수를 분리합니다. v0.5: 자원마다 `deployed_incident_id`(투입된 산불, 아니면 null)를 줍니다. COMMANDER/VIEWER는 허용된 incident_id 문맥의 관할·응원 범위로 제한합니다. 실제 범위·전국 자원 열람 정책은 Q-08에서 확정합니다.

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

`data`: {bases: ResourceCount[], incident_deployed: object 또는 null, incident_linked: object 또는 null}

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
    },
    "incident_linked": {
      "HELICOPTER": 2,
      "VEHICLE": 5
    }
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 각 ResourceCount는 base_id: ID, kind: ResourceKind, available/deployed/maintenance/total: 0 이상 integer입니다. 상태 필터를 받지 않고 종류·기지·허용범위 전체 상태를 집계합니다. incident_deployed는 incident_id 미지정 시 null, 지정 시 Situation.deployed입니다. 보유 상태의 deployed를 특정 산불 투입 수로 해석하지 않습니다. **v0.5: incident_linked**는 자원 연결(deployed_incident_id)로 센 그 산불의 {HELICOPTER, VEHICLE} 수로, 상황 보고 숫자(incident_deployed)와 따로 보여 줍니다. incident_id 미지정 시 null.

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

Resource의 id를 제외한 kind·name·base_id 필수, status 생략 시 AVAILABLE, attrs 생략 시 {}. v0.5: deployed_incident_id는 status=DEPLOYED면 필수(진행 중 산불), 아니면 생략/null

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

kind·name·base_id·status·attrs·deployed_incident_id 중 1개 이상(v0.5: status=DEPLOYED로 바꾸면 deployed_incident_id 필수, 다른 상태면 null로 풀림). 타입·제약은 Resource 등록과 같고 deployed_incident_id만 null 허용

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

`data`: Page<ProposalVersion>

```json
{
  "data": {
    "items": [
      {
        "id": "72",
        "version": "v1.2",
        "trigger": "PREDICTION",
        "created_at": "2025-03-22T12:53:00+09:00",
        "judged_stage": "STAGE1",
        "predicted_stage": "STAGE2",
        "category_counts": {
          "IMMEDIATE": 9,
          "STANDBY": 2,
          "CONSULT": 2,
          "REQUEST": 3
        },
        "changed_count": 2
      },
      {
        "id": "71",
        "version": "v1.1",
        "trigger": "PREDICTION",
        "created_at": "2025-03-22T12:05:00+09:00",
        "judged_stage": "STAGE1",
        "predicted_stage": "STAGE1",
        "category_counts": {
          "IMMEDIATE": 8,
          "STANDBY": 3,
          "CONSULT": 2,
          "REQUEST": 3
        },
        "changed_count": null
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

**처리 규칙:** id DESC, 산불별 최신부터. 같은 화선에서 예측 재실행·상황 정정을 해도 seq가 증가합니다(version 「v{화선}.{순번}」). 과거 화선 정정 여부는 PER-01로 확인합니다. v0.5: 한 줄은 이력 표에 필요한 칸만(ProposalVersion) — changed_count는 바로 앞 버전 대비 바뀐 항목 수(ProposalItem의 「변경」 판단, PROP-04 counts.changed와 같음)이고 첫 버전은 null입니다. 나머지 칸(prediction_id·base_at 등)은 PROP-03에서 봅니다.

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
    "id": "72",
    "version": "v1.2",
    "trigger": "PREDICTION",
    "created_at": "2025-03-22T12:53:00+09:00",
    "base_at": "2025-03-22T12:50:00+09:00",
    "prediction_id": "41",
    "judged_stage": "STAGE1",
    "predicted_stage": "STAGE2",
    "items": [
      {
        "item_code": "S1",
        "name": "대응단계 판단·격상 검토·지휘권 인계",
        "axis": "SUPPRESSION",
        "state": "GENERATED",
        "categories": [
          "STANDBY",
          "CONSULT"
        ],
        "finding": "현재 기준 1단계(피해면적 48.5 ha · 평균풍속 3.4 m/s · 시설피해 우려 최소 2동) · P5 기준 2단계(피해면적 363 ha) → 5시간 안에 2단계 기준 도달 예상 · 지휘권자 의성군수 → 경상북도지사",
        "withheld": [
          {
            "field": "진화예상시간",
            "reason": "MVP에서 제외"
          }
        ],
        "change_type": "CHANGED",
        "sections": [
          {
            "title": "1. 대응단계 판단·격상 검토",
            "blocks": [
              {
                "type": "paragraph",
                "category": "CONSULT",
                "body": "피해면적 48.5 ha, 평균풍속 3.4 m/s, 시설피해 우려 최소 2동으로 세 기준 모두 확산대응 1단계에 해당합니다. 확산대응 1단계 발령을 산림청장과 협의하십시오.",
                "cite": "[p.73·91]",
                "quote": "산림청장은 대응단계 판단기준에 따라 산불현장 통합지휘본부와 협의하여 대응단계를 발령한다",
                "evidence": [
                  "SM-p073-s01",
                  "SM-p091-s02"
                ],
                "text": "피해면적 48.5 ha, … 확산대응 1단계 발령을 산림청장과 협의하십시오 [p.73·91].",
                "situation": "피해면적 48.5 ha, 평균풍속 3.4 m/s, 시설피해 우려 최소 2동으로 세 기준 모두 확산대응 1단계에 해당합니다.",
                "action_text": "확산대응 1단계 발령을 산림청장과 협의하십시오.",
                "topic": "현재 기준 단계",
                "source": "rule+llm"
              }
            ]
          }
        ],
        "evidence": [
          {
            "chunk_id": "SM-p073-s01",
            "label": "p.73",
            "doc_name": "「산불 재난」 위기관리 표준매뉴얼",
            "section": "대응단계 판단기준",
            "page": 73,
            "origin": "판정 근거",
            "text": "(근거 원문 전체)"
          }
        ]
      },
      {
        "item_code": "E3",
        "name": "취약시설 조치",
        "axis": "EVACUATION",
        "state": "NOT_TRIGGERED",
        "categories": [],
        "finding": "",
        "withheld": [],
        "change_type": null,
        "sections": [],
        "evidence": []
      }
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 아직 제안서가 없으면 data=null입니다. v0.5: items는 **늘 12칸**(S1→S6, E1→E6)이고 발동 안 한 항목은 `state: NOT_TRIGGERED`입니다(예시는 12칸 중 2칸). 필터와 건수는 items의 axis·categories로 프론트에서 계산할 수 있습니다. 여러 categories를 가진 항목은 각 구분에 포함되므로 구분 건수 합과 전체 항목 수는 다를 수 있습니다. 근거 원문은 items[].evidence에 함께 들어 있어 별도 조회 API가 없습니다(EVD-01 삭제). change_type은 바로 앞 버전과 비교한 값입니다(ProposalItem 「변경」 판단). 새 제안서가 저장될 때만 최신이 바뀌므로, 화면은 PRED-05 SUCCEEDED나 INC-03 latest_proposal_id가 바뀔 때 다시 부릅니다.

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
    "version": "v1.1",
    "trigger": "PREDICTION",
    "created_at": "2025-03-22T12:05:00+09:00",
    "base_at": "2025-03-22T12:50:00+09:00",
    "prediction_id": "41",
    "judged_stage": "STAGE1",
    "predicted_stage": "STAGE1",
    "items": [
      {
        "item_code": "E3",
        "name": "취약시설 조치",
        "axis": "EVACUATION",
        "state": "NOT_TRIGGERED",
        "categories": [],
        "finding": "",
        "withheld": [],
        "change_type": null,
        "sections": [],
        "evidence": []
      }
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 과거 버전도 그때 저장한 전체 12칸 스냅샷으로 반환합니다(예시는 1칸만). 한번 만든 버전은 바뀌지 않으므로 change_type도 **그 버전의 바로 앞 버전**과 비교한 고정값입니다(첫 버전이면 모두 null). 공식 단계 필드는 없으며 judged_stage·predicted_stage만 반환합니다. 숫자·근거는 해당 버전에 저장된 것을 사용합니다. 「지난 버전인가」는 INC-03의 latest_proposal_id와 비교합니다. 다른 산불의 proposal_id이거나 없으면 404입니다.

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

`data`: ProposalDiff

```json
{
  "data": {
    "version": "v2.2",
    "previous_version": "v2.1",
    "trigger": "CORRECTION",
    "created_at": "2025-03-22T15:40:05+09:00",
    "cause": "상황 정정: 박곡리 대피 완료, 헬기 6대 투입",
    "counts": {
      "changed": 3,
      "wording_only": 2,
      "same": 7
    },
    "items": [
      {
        "item_code": "S3",
        "name": "진화자원 현황·동원 범위",
        "change": "CHANGED",
        "categories": {
          "before": [
            "IMMEDIATE",
            "STANDBY",
            "REQUEST"
          ],
          "after": [
            "IMMEDIATE",
            "STANDBY",
            "REQUEST"
          ]
        },
        "finding": {
          "before": "투입: 헬기 23 · 장비 26 · 인력 151 / 현재 1단계 → 5시간 뒤 2단계",
          "after": "투입: 헬기 29 · 장비 26 · 인력 151 / 현재 1단계 → 5시간 뒤 2단계"
        },
        "proposals": [
          {
            "change": "MODIFIED",
            "topic": "투입 자원",
            "before": {
              "category": "IMMEDIATE",
              "body": "현재 헬기 23대(산림청 5, 지자체 16, 소방 2)가 투입되어 있습니다. 가용 헬기를 집중 투입하십시오."
            },
            "after": {
              "category": "IMMEDIATE",
              "body": "현재 헬기 29대(산림청 9, 지자체 16, 소방 4)가 투입되어 있습니다. 가용 헬기를 집중 투입하십시오."
            }
          }
        ],
        "rows": {
          "added": [],
          "removed": []
        },
        "note": null
      }
    ],
    "wording_only_items": [
      "S1",
      "E5"
    ],
    "same_items": [
      "S4",
      "S5",
      "S6",
      "E1",
      "E3",
      "E4",
      "E6"
    ]
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 직전은 같은 산불에서 현재 id보다 작은 가장 큰 제안서입니다. v0.5: item_code로 맞추고 ProposalItem의 「변경」 판단으로 **바뀐 항목만, 항목 안에서도 바뀐 부분만** 돌려줍니다 — 판정 한 줄 이전/이후, 바뀐 제안 문단(주제 topic으로 짝, 같은 주제 여러 문단은 순서대로, 주제 없는 문단은 같은 문장끼리만), 목록·표·※ 줄의 +/−. LLM 표현만 바뀐 항목은 건수(counts.wording_only)와 코드 목록만 줍니다. 첫 버전은 previous_version·cause·counts가 null이고 목록은 모두 []입니다. 카드의 「변경」 태그를 누르면 같은 응답에서 그 항목만 골라 보여 줍니다.

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

<a id="ai-01"></a>

### AI-01 · 질의응답 또는 정정 확인안

> **v0.5 보완(11/10)**: 서버 구현(`wfg-server` PR #24)과 맞춤. QaMessage에 route · sentences · insufficient · no_evidence, CitationRef에 label · origin · text, CorrectionPreview에 ignored · notes · summary · proposal_regenerates, UNSUPPORTED에 ignored를 더함.


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

`data`: CorrectionPreview 또는 {type: ANSWER, message: QaMessage} 또는 {type: UNSUPPORTED, reason: string, ignored: {text, reason}[]}

```json
{
  "data": {
    "type": "CORRECTION_PREVIEW",
    "confirmation_token": "EXAMPLE_PREVIEW_TOKEN",
    "expires_at": "2025-03-22T12:30:00+09:00",
    "situation_revision": "rev-example-1",
    "input_text": "헬기 2대 더 왔어",
    "changes": [
      {
        "target": "deployed",
        "item": "helicopter.unknown",
        "before": 0,
        "after": 2,
        "label": "투입 헬기(미상) 0 → 2대"
      },
      {
        "target": "deployed",
        "item": "helicopter.total",
        "before": 23,
        "after": 25,
        "label": "투입 헬기(합계) 23 → 25대"
      }
    ],
    "ignored": [],
    "notes": [
      "헬기 기관을 말하지 않아 「미상」 칸에 넣었습니다(기관을 알면 「미상 헬기 2대는 산림청」처럼 다시 정정해 주십시오)."
    ],
    "summary": "변경 1건을 확인하면 상황에 저장하고 제안서를 다시 만듭니다: 투입 헬기(미상) 0 → 2대",
    "proposal_regenerates": true
  },
  "meta": {
    "request_id": "req-example"
  }
}
```

**처리 규칙:** 질문이면 ANSWER로 질문·답변을 저장하고 상황은 바꾸지 않습니다. 정정이면 확인안만 반환합니다. 허용 목록 밖 입력·모호한 대상은 UNSUPPORTED 또는 재질문 안내로 처리하고 저장하지 않습니다. 없는 village_id·근거를 LLM이 만들어낼 수 없습니다. 종료된 산불도 질문은 허용하지만 정정 확인안은 거절합니다. 여러 턴 자동 문맥은 제외합니다.

- 투입 수: 기관을 말하지 않으면 헬기는 미상, 인력 · 차량은 기타 칸에 넣고 notes로 알립니다. 「미상 헬기 2대는 산림청」은 미상에서 빼 그 기관 칸으로 옮깁니다(total 그대로). 줄이는 말인데 미상 · 기타 칸에 뺄 만큼이 없으면 어느 기관이 줄었는지 묻습니다(ignored).
- 자원 상한: 자원 테이블에 있는 칸(산림청 · 지자체 임차 · 소방 헬기, 진화차 · 소방차)은 그 칸 자원의 대기 + 이 산불 투입 대수를 넘게 늘릴 수 없습니다. 넘는 줄만 ignored로 빼고 나머지는 확인안으로 보입니다(COR-01에서 다시 확인).
- LLM(GPU 서버)에 닿지 않으면 503 AI_UNAVAILABLE입니다. 서버 설정 `ASSISTANT=rules`(기본)는 LLM 없이 규칙으로만 답합니다.

**주요 오류:** 409 INCIDENT_ENDED (정정 입력), 422 VALIDATION_ERROR / TARGET_NOT_FOUND / AMBIGUOUS_TARGET / INVALID_COUNT(0보다 작아짐), 503 AI_UNAVAILABLE

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
        "answer": "화선 면적과 풍속으로 확산대응 1단계로 판정했습니다 [1].",
        "citations": [
          {
            "id": "111-1",
            "k": 1,
            "chunk_id": "SM-p073-s01",
            "chunk_code": "SM-p073-s01",
            "label": "p.73",
            "doc_name": "「산불 재난」 위기관리 표준매뉴얼",
            "page": 73,
            "section": "대응단계 판단기준",
            "origin": "판정 근거",
            "text": "(근거 원문)"
          }
        ],
        "route": "PROPOSAL",
        "sentences": [
          {
            "text": "화선 면적과 풍속으로 확산대응 1단계로 판정했습니다 [1].",
            "citations": [1],
            "data_refs": []
          }
        ],
        "insufficient": [],
        "no_evidence": false,
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

**처리 규칙:** 토큰의 사용자·산불·유효시간·원문·변경안·revision과 서버 현재값·가용 자원 범위를 다시 확인합니다. 가용 자원 범위(11/10) = 투입 칸별로 자원 테이블(resources)의 대기 + 이 산불에 투입된 대수입니다(산림청 헬기 = 기지 org_type forest, 지자체 헬기 = local_lease · local, 소방 헬기 · 소방차 = fire, 진화차 = forest · local_lease · local, 정비 중 · 다른 산불 투입은 뺌). 군 · 경찰 · 미상 · 기타 · 인력처럼 자원 테이블에 없는 칸과, 그 칸 자원이 하나도 등록되지 않은 경우는 상한이 없습니다. 확인안을 만든 뒤 자원 현황이 줄어 넘게 되면 409 RESOURCE_CAPACITY_EXCEEDED입니다. 정정 행+상황 수정+로그를 먼저 한 트랜잭션으로 확정합니다. 예측이 있으면 제안 재생성 작업을 반환하고 없으면 null입니다. 제안 생성 실패는 이미 저장된 정정을 취소하지 않습니다. 작업 시작 실패 시에도 정정 저장 성공을 반환하고 proposal_stale=true, proposal_operation=null로 안내한 뒤 PROP-05로 재시도합니다.

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

## 7. 지도 레이어와 위험도 변수

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

> v0.5: v0.4의 7.2 targets → 지도 연결 · 7.3 제안 detail.facts 해석 · 7.4 E4 대피소 후보 · 7.5 E5 도로·E6 전파 구분은 삭제했습니다. 제안서 응답에 규칙 판정의 facts·targets를 넣지 않고(지도 강조는 MVP 제외), 항목도 12개(S1~S6·E1~E6)로 바뀌었습니다. 대피소 후보·도로 통제 같은 내용은 해당 항목의 sections(목록·표)에 글로 들어 있습니다.

### 7.2 위험도 23개 변수

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
| 예측 실행 | PRED-01의 202(기상을 못 받으면 바로 503, mode는 선택 — 실서비스·시연) → PRED-05 조회(result.mode로 어떤 방식이었는지 표시) → 성공 후 INC-03 재조회 및 PRED-02·03·04, RISK-01, PROP-02를 동일 결과 ID로 연결. 산불현황 기상 칸은 WTH-01 |
| 예측 실패 | 기존 도형·제안 유지, 작업 오류 안내. 버튼만 재활성화하고 중복 요청을 피함 |
| 제안 필터 | PROP-02/03의 items[].axis·categories로 진화/대피 탭·구분 필터. 지도 강조(targets)는 v0.5 응답에 없음(MVP 제외) |
| 근거·이력 | 문단의 cite를 누르면 같은 항목의 evidence[](원문 포함)에서 찾아 표시, quote로 칠함. 이력은 PROP-01 → PROP-03/04. 해당 버전의 근거·수치 표시 |
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
| Q-01 | 위험도 저장·계산·종료 정책 | **v0.5 일부 확정**: 위험도를 못 구해도 예측·제안은 저장하고 RISK-01만 data: null. 아래는 미결 — DB 4.2: predictions 컬럼 추가/별도 risk_assessments, 등급 래스터 저장소, 종료 후 최대 위험도 표시가 미결. 변수별 구간 경계·수분/지형복잡도·임상도 변환·격자 30/100m·기상 시점/공간·취약인구 범위·소방서 거리 기준·최종 표시 등급도 확인. 미산출의 부분 허용 결정과 현재 서버의 시험용 위험도(fake-0)는 구분 | RISK-01, PRED-01, E2·AI |
| Q-02 | 현재 단계 판정 시 P5 가용 시점 | **v0.5 확정**: current_stage는 예측할 때 판정(보고·정정 때 판정 안 함, STAGE_INPUT_UNAVAILABLE 삭제). **남은 것**: 관측시각 역전 — 서버는 422 INVALID_OBSERVED_AT(확인 필요). 원래 질문 — DB는 산불/화선 보고 때 current_stage 판정, 시설피해 우려는 P5 안 주택+국가유산으로 규정. 최초 보고·새 화선 보고 직후 해당 P5가 아직 없거나 최신 화선과 예측 기준이 다를 때 무엇으로 판정할지 불명확. 예측 전 시설피해 결측·기상 수신 실패·NN 단계 저장·관측시각 역전도 함께 결정 | INC-02, PER-02·03, S1 |
| Q-03 | 위기경보 유지 여부 | **v0.5 닫음**: 위기경보(alert_level)를 API·DB에서 삭제. 원래 질문 — DB는 alert_level 입력·정정 유지, LLM·규칙 문서는 쓰는 곳이 없어 제외/팀 논의. API는 DB 필드로 남기되 실제 입력 UI·필수 여부·정정 범위를 최종 확인 | INC·COR, S1·S6 |
| Q-04 | detail·targets 최종 계약 | **v0.5 닫음**: 제안서 응답에 detail·targets를 넣지 않음(finding·withheld만, 지도 강조 MVP 제외). 원래 질문 — DB 옛 예시는 targets 객체, 최신 엔진은 문자열 배열. 이름 참조·kind 포함 참조도 혼재. 서버 어댑터의 PK 기반 정규화·저장 위치·권한 검증·외부 한글/영문 categories 매핑을 확정 | PROP·MAP·E5·E6 |
| Q-05 | 작업·멱등키·동시성 | **v0.5 일부 확정**: INC-02 Idempotency-Key는 DB 저장(2.4). 예측 작업 기록은 서버 메모리(재시작 시 사라짐), 작업 실패 error.code 이름(PREDICTION_INVALID·PROPOSAL_FAILED·INTERNAL_ERROR)은 확인 필요. 원래 질문 — 202/작업 조회는 API 제안. 작업 큐·공유 저장소·보존기간·재시작 복구, idempotency 결과 재전달, 화선·상황·자원 변경·종료 중 커밋 정책을 확정. predictions에는 성공 행만 저장 | PRED·COR·PROP-05·보고 |
| Q-06 | 제안 stale·정정 반영 기준 | **v0.5**: 서버의 situation_revision은 rev-{최신 정정 id} 임시 값, proposal_stale은 최신 정정 시각 > 최신 제안서 시각. COR-01은 서버 PR #24·명세 PR #67에서 구현과 처리 규칙을 보완. 아래 동시성·변경 감지 질문은 별도 보완 사항. 규칙 엔진 결과 전체는 proposals.context에 저장(DB 11/9). 원래 질문 — DB는 정정시각과 제안시각 비교. 다중 정정·재시도·여러 쓰기 경로·동일시각·재생성 대상을 안전하게 판별할 revision을 어떻게 둘지 결정. 과거에는 context 저장 위치가 미정이었으나 현재 11/9 DB에는 proposals.context가 정의됨 | INC-03, COR, PROP-05, AI |
| Q-07 | S7·미결 규칙·근거 준비 | **v0.5 대부분 닫음**: S7 없음(12항목 S1~S6·E1~E6, 11/7). 근거 청킹·검색·인용 검증은 백엔드 에이전트 내부이고 API는 저장된 evidence만 돌려줌. 원래 질문 — 주 확산 방향·S7 facts, 시군구 최소 겹침면적(구현 기본 1ha 제안), 표준매뉴얼 청킹·rule_basis→실제 chunk 매핑·추가 검색 K·인용/숫자 검증을 확인. 문서에 적힌 코드가 실제 검색 가능한 청크임을 가정하지 않음 | 규칙·PROP·AI·EVD |
| Q-08 | 역할·인접 범위·개인정보 | VIEWER의 제안/근거/AI 접근, REPORTER 지도·상황 반환 범위, 관할 밖 소방·경찰·병원·응원 자원, 신고자 이름·전화 마스킹, 내부 매뉴얼 본문·원문 공개 범위를 확인 | 권한 전체·INC·MAP·RES·EVD |
| Q-09 | 인증·관리 API 운영 | 실제 라우트·응답 형식·CORS, JWT 만료값·로그아웃 폐기, 비밀번호 재발급 시 기존 JWT 무효화·초기 변경 강제·관리자 계정 발급 범위, 자원 동시 편집 방식을 확인. DB에 없는 세션/토큰버전/자원 updated_at은 구현 완료로 가정하지 않음 | AUTH·USR·RES |
| Q-10 | 적재·기지 삭제·이력 | **v0.5**: 대피소 tier 값 확정(village·hub·null). 원래 질문 — 전국 시도/시군구, 기지 coverage_area_ids, house, 대피소 tier, 도로 ref·인근 경찰서 적재를 확인. 기지 실제 삭제와 과거 detail.targets 참조 보존 정책 필요. 마을·시설 재적재 시 기존 ID 유지 | MAP·BASE·규칙·과거 제안 |

설계 문서만으로 실제 서버 구현·DB 생성·공유 파일 적재 여부는 확인할 수 없습니다. 따라서 '지도·자원 테이블이 아직 없다'고 쓰지 않고 **테이블 정의는 있으며 필요한 데이터와 구현 상태를 확인해야 한다**고 구분했습니다. 근거 저장은 최신 DB 원칙·LLM·데이터 문서의 FAISS 선택을 따르며, DB 본문의 과거 pgvector/FAISS 미정 문구를 현재 전체 결정으로 확대하지 않았습니다.

<a id="changes"></a>

## 10. 바뀐 내용

### v0.5 보완 — 예측 모드 (2026-11-11)

`wfg-server` PR #40(예측 모드) · #39(ELMFIRE 실행 컨테이너)와 맞췄습니다. 요청에 칸 하나가 늘고 작업 결과에 칸 둘이 늘었습니다. 안 보내면 지금과 같습니다.

| 변경 | 이번 반영 |
| --- | --- |
| PRED-01 요청 | `mode` 선택 칸 — LIVE(실서비스) · DEMO_ELMFIRE(시연 1) · 없음(서버 기본). 모르는 값은 422 |
| PRED-01 503 | LIVE 관측소 3곳 미만 · DEMO_ELMFIRE 덱 기상에 없는 시각 |
| PRED-05 result | `mode` · `predictor` 추가(성공한 예측 작업) |
| 작업 실패 | ELMFIRE 입력 문제 PREDICTION_INVALID, 시간 초과 · 실행 실패 INTERNAL_ERROR |
| 상태값 | PredictionMode 추가. 시연 2(DEMO_REPLAY)는 만들면 값을 더함 |

### v0.5 보완 — ELMFIRE 기상 (2026-11-11)

`wfg-server` PR #36(ELMFIRE 기상 입력 · 출처 실시간 stations · 시연 deck)과 맞췄습니다. 응답 모양(칸)은 그대로이고 값만 늘었습니다.

| 변경 | 이번 반영 |
| --- | --- |
| Weather.source | `KMA_STATIONS` 추가 — ELMFIRE 실시간(관측소 매분자료 + 단기예보 → 600 m 격자)의 발화점 칸 값. ELMFIRE 시연(덱에 든 의성 관측 기상)은 기존 `AWS_OBS` |
| Weather.location | `{lat, lon}`(발화점) 추가 — KMA_STATIONS · 시연 AWS_OBS |
| PRED-01 | ELMFIRE 기상 입력도 요청 때 만듦. 관측소 3곳 미만 · 시연 기상에 없는 시각 → 503 WEATHER_UNAVAILABLE(대체 없음) |
| 저장 기상의 바람 | wind_speed = 평균풍속(대응단계 3 · 11 m/s 기준과 같은 정의). ELMFIRE 입력값(시간 안 가장 센 1분 풍속)은 DB에만 두고 응답(PRED-04)에 내지 않음 |
| hours[].kind | 값은 그대로(「관측」·「예보」). 0시는 섞인 값 중 실제로 잰 값의 비중이 절반 이상이면 「관측」 |

### v0.5 보완 — AI 어시스턴트 · 상황 정정 (2026-11-10)

`wfg-server` PR #24(AI-01 · AI-02 · COR-01 · COR-02 · PROP-05 구현)와 맞추고, 11/10 결정을 반영했습니다.

| 변경 | 이번 반영 |
| --- | --- |
| QaMessage | route · sentences(문장별 citations · data_refs) · insufficient · no_evidence 추가 |
| CitationRef | AI-01 전용. id = 「질의응답 id-k」, chunk_id = chunk_code = 근거 청크 코드, label · origin · text 추가(Evidence 칸 + k · id) |
| CorrectionPreview · UNSUPPORTED | ignored · notes · summary · proposal_regenerates 추가, UNSUPPORTED에 ignored |
| CorrectionChange | label · target_name 추가, village item에 pre_evac_done |
| 헬기 미상 칸 | Situation.deployed.helicopter에 police · unknown(미상). 기관을 모르는 투입 수는 미상(헬기) · 기타(인력 · 차량), total의 남는 몫도 그 칸 몫 |
| 자원 상한 | RESOURCE_CAPACITY_EXCEEDED 기준 = 자원 테이블의 대기 + 이 산불 투입(칸별). AI-01은 넘는 줄만 ignored, COR-01에서 다시 확인 |
| 이벤트 | QA_NO_EVIDENCE(근거 부족 답변) · SITUATION_CORRECTED · PROPOSAL_FAILED(정정 뒤 제안서 다시 만들기 실패) — DB-설계 event_logs 예시에 추가 |
| 남은 일 | 위기경보 격상 · 조정 메뉴(정정 항목이 아니라 별도 기능으로 — DB 칸 · API를 따로 정함) |

### v0.4 → v0.5 (2026-11-10)

P1·P2 API를 서버 구현(`wfg-server` main `907bd42`)과 대조해 고쳤습니다. 성공·실패 구조는 v0.4를 따릅니다.

| 변경 | 이번 반영 |
| --- | --- |
| 제안서(PROP-01~04) | 항목 12개(S1~S6·E1~E6) 늘 12칸, 본문은 sections → blocks, 판정은 finding·withheld만(detail·targets 제외), 근거 원문 Evidence 포함, 변경 태그 NEW/CHANGED/SAME, PROP-01은 ProposalVersion, PROP-04는 바뀐 부분만(ProposalDiff) |
| 근거 | EVD-01 삭제(원문이 PROP-02 안에). CitationRef는 AI-01 정리 전까지만 남김 |
| 위기경보 | AlertLevel·alert_level 삭제(INC-02 요청에 보내면 422). Q-03 닫음 |
| 대응단계 판정 | 산불·화선 보고/정정 때가 아니라 예측할 때. STAGE_INPUT_UNAVAILABLE 삭제 |
| 산불 | INC-01 ADMIN 허용(전체 지역), INC-03 running_operation_id, INC-02 Idempotency-Key 규칙 확정, INC-04 종료 시 투입 자원 자동 해제(서버 반영 예정) |
| 화선 | PER-02·03 Polygon 허용. 관측 시각 역전은 422 INVALID_OBSERVED_AT(확인 필요) |
| 예측·기상 | PRED-01 성공·실패 구조는 v0.4 그대로(기상은 요청 때, 실패는 503). 작업에 EVIDENCE 단계 안 나옴, 작업 실패 코드 이름(확인 필요), 위험도 미산출이어도 저장. Weather source FIXED·hours[].kind 추가. WTH-01 지금 날씨 추가 |
| 지도·자원 | 대피소 tier 값(village·hub·null), Resource·RES-03·04 deployed_incident_id, RES-02 incident_linked |
| 값 표기 | 영문 코드 유지. 한글 그대로는 Evidence.origin·hours[].kind 두 개 |
| 7절 | 7.2~7.5(targets·facts·E4 후보·E5/E6 구분) 삭제, 7.6 → 7.2 |
| 확인 필요(미정) | PER-02·03 관측 시각 규칙, 작업 실패 코드 이름, situation_revision·proposal_stale(COR-01 결정과 함께), AI-01 전체. LOG-01 from_at·to_at은 명세 그대로이고 서버가 추가 예정 |

### v0.3 → v0.4 (노션 초안에서)

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
| 2026-11-10 | v0.5 검토본 | wfg-server main(907bd42)과 P1·P2 대조. 제안서 계약 재작성, EVD-01 삭제·WTH-01 추가, 위기경보 삭제. PR #66 병합 검토에서 Evidence/withheld null 허용, 재시도 보장 범위·DB 대조 출처를 보정 |
| 2026-11-10 | v0.5 보완 | AI-01 · COR를 wfg-server PR #24와 맞춤(응답 칸 추가, 헬기 미상 칸, 자원 상한 기준, 이벤트 이름) |
| 2026-11-11 | v0.5 보완 | ELMFIRE 기상(Weather source KMA_STATIONS · location, wfg-server PR #36) · 예측 모드(PRED-01 mode, PRED-05 result.mode · predictor, PR #40) |

요청·응답 타입과 미결 항목을 팀에서 검토한 후 main에 병합합니다. v0.5는 docs/api-spec-v0.5 작업 브랜치에서 검토하며 다른 설계 파일은 수정하지 않습니다. 이후 API 계약을 바꾸면 관련 DTO·프론트 타입·목업 응답·서버 OpenAPI도 같은 변경에 맞춰 관리합니다.

PR #67·#68 통합 검토에서는 우선순위 표의 AI-01 미정·P3 전체 미구현 문구와 COR-01 채택 대기 설명을 후속 구현에 맞게 보정하고, VillageState의 pre_evac_done을 정정 허용 목록과 맞췄습니다.


<a id="priority"></a>

## 11. API 우선순위(P1 · P2 · P3)

| 우선순위 | 뜻 | API |
| --- | --- | --- |
| P1 (12) | 시연 핵심 흐름: 로그인 → 산불 보고 → 지도 → 예측 → 제안 | AUTH-01 · 02, INC-01 · 02 · 03, MAP-01 · 02, PRED-01 · 02 · 03 · 05, PROP-02 |
| P2 (18) | 요구사항 기능, 핵심 흐름 다음 | AUTH-03, USR-01 · 02, INC-04, PER-01 · 02 · 03, PRED-04, RISK-01, WTH-01(v0.5 추가), RES-01 · 02, BASE-01, PROP-01 · 03 · 04, LOG-01, AI-01 |
| P3 (16) | 관리 세부 · 초과 · 결정 대기 | USR-03 · 04 · 05 · 06, MAP-03 · 04, RES-03 · 04 · 05, BASE-02 · 03 · 04, PROP-05, AI-02, COR-01 · 02 |

v0.5에서 P1의 EVD-01은 삭제했습니다(근거 원문이 PROP-02 안에). WTH-01은 P2에 추가했습니다. 이 표는 구현·확정 상태와 구분한 우선순위입니다.

초기 `907bd42` 대조에서는 P1·P2 중 AI-01을 제외한 API가 구현 대상으로 확인됐습니다. 후속 서버 [PR #24](https://github.com/wildfire-guard-capstone/wfg-server/pull/24)(main 병합)와 명세 [PR #67](https://github.com/wildfire-guard-capstone/wfg-docs/pull/67)은 AI-01·AI-02·COR-01·COR-02·PROP-05를 추가 구현·보완합니다. 따라서 AI-01 미정·P3 전체 미구현이라는 이전 설명은 최신 상태에 적용하지 않습니다. 실제 배포·DB 마이그레이션·프론트 연동 검증은 별도로 확인합니다.

