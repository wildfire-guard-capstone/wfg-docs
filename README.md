# wfg-docs

2026 캡스톤 디자인 산불 감지 프로젝트의 문서 저장소입니다.

회의록, 교수님 피드백, 기획 문서 등 프로젝트 전반의 기록을 관리합니다.

## 구조

```
wfg-docs/
├── meetings/       # 회의록
├── feedback/       # 교수님 피드백
└── docs/           # 기획 및 설계 문서
```

## 파일 네이밍 규칙

- 모든 문서는 반드시 `.md` 확장자를 붙인다.
- 파일명은 한글을 사용하며, **띄어쓰기 대신 `-`(하이픈)** 를 쓴다.
- 형식: `캡스톤디자인(N주차)-<문서종류>-정리.md`
  - 피드백: `캡스톤디자인(2주차)-피드백-정리.md`
  - 회의록: `캡스톤디자인(2주차)-회의록-정리.md`

## 브랜치 전략

- `main` — 최종 확정 문서
- 작업 시 브랜치를 따서 작업 후 PR로 main에 머지

```bash
git checkout main
git pull origin main           # 최신 main 당기기
git checkout -b docs/작업내용  # 브랜치 생성
# 작업
git push origin docs/작업내용
gh pr create --base main
# GitHub에서 머지
```

## 관련 저장소

| 저장소 | 설명 |
|--------|------|
| [wfg-server](https://github.com/wildfire-guard-capstone/wfg-server) | 백엔드 서버 |
| [wfg-client](https://github.com/wildfire-guard-capstone/wfg-client) | 프론트엔드 클라이언트 |
