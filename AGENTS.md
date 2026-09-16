# AGENTS.md

## 기여 흐름

- 단순하지 않은 변경은 모두 Pull Request를 통해 반영한다.
- PR 제목은 Conventional Commit 형식을 따라야 한다. 예:
  - `feat: add new slide style`
  - `fix: handle missing image API key`
  - `docs: clarify installation steps`
- 커밋 메시지, PR 제목, CHANGELOG 항목, 릴리스 노트는 **영어로 작성한다**.
- 문서 변경 시 저장소 README와 사용 문서 사이트를 포함해 기존 언어 버전을 함께 동기화한다.

## 변경 기록

- 사용자에게 보이는 변경은 `CHANGELOG.md`를 갱신해야 한다.
- 미출시 변경은 `## Unreleased` 아래에 추가한다.
- 다음 섹션 중 하나를 사용한다.
  - `### Features`
  - `### Improvements`
  - `### Fixes`
  - `### Documentation`
- CHANGELOG 항목은 영어로 작성한다.
- PR을 연 뒤 CHANGELOG 항목에 `(#12)` 같은 PR 참조를 추가한다.
- 첫 커밋 시 PR 번호를 알 수 없다면 PR을 먼저 연 다음 후속 커밋에서 `(#PR_NUMBER)`를 추가한다.

## 릴리스 절차

- 버전은 SemVer를 사용한다.
- Git 태그는 `v0.1.0`처럼 앞에 `v`를 붙인다.
- GitHub Release는 일치하는 `CHANGELOG.md` 버전 섹션을 기반으로 생성한다.
- 기존 릴리스 본문을 `CHANGELOG.md`와 맞추기 위한 경우를 제외하면 GitHub Release 노트를 수동 작성하지 않는다.

## 검증

- PR을 열기 전에 변경된 GitHub workflow YAML이 파싱되는지 가능한 범위에서 확인한다.
- workflow의 shell 스니펫은 가능한 경우 `bash -n` 같은 구문 검사를 실행한다.
