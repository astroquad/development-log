# Current Step Plan

최종 업데이트: 2026-05-13

이 문서는 **현재 한 스텝 작업의 실행 계획**만 임시로 기록한다. 매 작업마다 덮어써도 된다.

장기 계획은 여기에 오래 두지 않는다.

- 전체 시스템 기준: `development-log/SYSTEM_SPEC.md`
- 72시간/1주일 MVP 계획: `development-log/MVP_PLAN.md`
- repo별 기준: `uav-onboard/PROJECT_SPEC.md`, `uav-gcs/PROJECT_SPEC.md`
- 현재 스텝 조사: `development-log/RESEARCH.md`

## Current Task

문서 계층 정리.

목표:

- `SYSTEM_SPEC.md` 생성.
- `MVP_PLAN.md` 생성.
- `RESEARCH.md`를 현재 스텝 조사용 문서로 리셋.
- `PLAN.md`를 현재 스텝 실행 계획용 문서로 리셋.
- repo별 `PROJECT_SPEC.md`에서 새 문서 계층을 참조하게 정리.

## Steps

- [x] 기존 `RESEARCH.md`와 `PLAN.md`의 장기 내용을 분류한다.
- [x] 전체 시스템 기준을 `SYSTEM_SPEC.md`로 옮긴다.
- [x] 72시간 MVP와 보고서 계획을 `MVP_PLAN.md`로 옮긴다.
- [x] `RESEARCH.md`를 current-step research 템플릿으로 리셋한다.
- [x] `PLAN.md`를 current-step plan 템플릿으로 리셋한다.
- [x] `PROJECT_SPEC.md` 문서 참조를 정리한다.
- [x] diff check를 실행한다.

## Acceptance Criteria

- 큰 프로젝트 기준은 `SYSTEM_SPEC.md` 또는 repo별 `PROJECT_SPEC.md`에 있다.
- 큰 MVP 일정은 `MVP_PLAN.md`에 있다.
- `RESEARCH.md`와 `PLAN.md`는 다음 작업 때 부담 없이 덮어쓸 수 있는 짧은 문서다.
- 문서 간 역할이 중복되지 않는다.

## Test/Validation

예정:

```powershell
git -C development-log diff --check
git -C uav-onboard diff --check
git -C uav-gcs diff --check
```
