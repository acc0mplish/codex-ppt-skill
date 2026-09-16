---
name: codex-ppt
description: 기사, 보고서, 논문, 메모 또는 개요를 바탕으로 시각적으로 통일된 이미지 기반 PPT/PPTX 프레젠테이션을 생성한다.
metadata:
  openclaw:
    requires:
      bins:
        - python3
    primaryEnv: OPENAI_API_KEY
    envVars:
      - name: OPENAI_API_KEY
        required: false
        description: CLI 폴백에서 사용할 API 키.
      - name: OPENAI_BASE_URL
        required: false
        description: API 기본 URL.
      - name: CODEX_PPT_IMAGE_MODEL
        required: false
        description: 이미지 모델. 기본값은 gpt-image-2.5-flare.
      - name: CODEX_PPT_HOME
        required: false
        description: 런타임 홈 경로 재정의.
    homepage: https://github.com/ningzimu/codex-ppt-skill
---

# Codex PPT

## 개요

이 스킬은 원본 자료를 바탕으로 이미지 기반 PowerPoint 프레젠테이션을 만든다. 각 슬라이드는 완성된 16:9 이미지 한 장으로 생성하며, 최종 이미지는 `scripts/assemble_ppt.py`로 `.pptx` 파일에 조립한다.

사용자가 시각적으로 일관된 프레젠테이션을 원하고 슬라이드 전체가 이미지로 구성되는 방식을 허용할 때 사용한다. 모든 텍스트 상자, 차트 또는 도형을 개별 편집 가능한 상태로 유지해야 한다면 사용하지 않는다.

가능하면 내장 이미지 생성/편집 도구를 우선 사용한다. 내장 백엔드를 사용할 수 없거나 필요한 기능을 지원하지 않거나 사용자가 API/CLI 모드를 명시적으로 요청한 경우에만 `scripts/image_gen.py`를 사용한다.

## 필수 제약 조건

- 각 단계에 들어가기 전에 `참조 맵`에 지정된 관련 파일을 읽는다. 이 파일은 전체 오케스트레이션 계약이며, 세부 규칙은 `docs/`, 작업자 프롬프트는 `prompts/`에 있다.
- 승인 게이트를 반드시 지킨다. `docs/workflow-gates-and-progress.md`에 정의된 승인이 완료되기 전에는 최종 `deck_spec.json`, `speech.md`, 프롬프트 작업 파일, 슬라이드 이미지 또는 `.pptx`를 만들지 않는다.
- 사용자가 샘플 슬라이드를 승인하고 전체 덱 생성을 허가한 뒤에는, 서브에이전트를 사용할 수 있는 환경이라면 나머지 모든 슬라이드 이미지 작업을 반드시 슬라이드 서브에이전트에 배정한다.
- 메인 에이전트는 오케스트레이션, 프롬프트 작업, 상태 기록, QA, 발표자 노트, 조립을 담당한다. 사용 가능한 슬라이드 서브에이전트를 무시하고 순차 생성으로 조용히 대체하지 않는다.
- 최종 `origin_image/slide_XX.png`는 모두 선택된 이미지 백엔드, 즉 내장 이미지 생성/편집 도구 또는 `scripts/image_gen.py`를 통해 생성해야 한다.
- 로컬 드로잉, Pillow, SVG, HTML/CSS/canvas 스크린샷, python-pptx/PptxGenJS 레이아웃, 수동 오버레이는 폴백이 아니라 실패 방식으로 간주한다.
- 백엔드 확인이 끝난 뒤에는 선택한 이미지 백엔드를 고정한다. 서브에이전트가 편의를 이유로 백엔드를 변경하지 못하게 한다.
- 샘플 승인 후에는 승인된 샘플이 어떤 방식으로 생성되었는지 기록하고, 정확히 같은 방식을 모든 슬라이드 서브에이전트에 전달한다.
- 슬라이드 배정 및 결과 상태는 반드시 번들 스크립트로 기록한다. 채팅 메시지만으로는 슬라이드가 배정되었거나 완료된 것으로 간주하지 않는다.
- 필수 서브에이전트, 이미지 백엔드 또는 필수 이미지 경로를 사용할 수 없다면 중단하고 슬라이드 ID와 근거를 포함한 블로커를 보고한다. 낮은 품질의 대체물을 만들지 않는다.

## 사용자에게 보이는 진행 상태

단순하지 않은 덱 작업에서는 활성 단계가 하나만 존재하는 사용자 표시 체크리스트를 유지한다. 정식 완료 근거는 `docs/workflow-gates-and-progress.md`에 정의한다.

기본 표시 단계:

1. 원본 자료, 개요, 스타일, 백엔드 결정을 준비한다.
2. 샘플 슬라이드 한 장을 생성하고 승인받는다.
3. 슬라이드 작업과 슬라이드 상태를 준비한다.
4. 슬라이드 서브에이전트에 작업을 배정한다.
5. 생성된 슬라이드 결과를 기록한다.
6. QA, 수정, 노트 작성, PPT 조립을 수행한다.

채팅 내용만으로 단계를 완료 처리하지 않는다. 실제 파일 또는 스크립트로 기록된 상태를 근거로 삼는다.

## 기본 워크플로

1. 원본 콘텐츠를 이해한다.
   - 주제, 대상 독자, 목표, 페이지 수, 스타일/브랜드 제약, 포함하거나 제외할 섹션을 파악한다.
   - 페이지 수가 지정되지 않았다면 실용적인 분량을 정한다. 일반적인 덱은 8~12장이다.

2. 덱 개요를 설계한다.
   - `outline.md`를 작성하거나 수정하기 전에 `docs/workflow-gates-and-progress.md`와 `docs/outline-style-and-sample.md`를 읽는다.
   - 슬라이드별 역할과 필요한 원본 이미지를 초안으로 작성한다. 사용자에게 확인을 요청한 뒤 승인이 끝날 때까지 스타일, 백엔드, 샘플 또는 후속 산출물을 만들지 않고 멈춘다.

3. 통일된 시각 스타일을 확정한다.
   - 스타일 옵션을 제안하거나 `references/`의 파일을 사용하기 전에 `docs/outline-style-and-sample.md`를 읽는다.
   - 구체적인 스타일 방향 2~3개를 제안하고 하나를 추천한 뒤 확인을 기다린다. 이후 하나의 시각적 정체성을 유지하되 페이지 역할에 따라 레이아웃을 다양화한다.

4. 이미지 백엔드를 확정한다.
   - 슬라이드 이미지를 생성하기 전에 `docs/backend-selection.md`를 읽는다.
   - 내장 이미지 도구를 실제로 호출할 수 있는지 확인하고, 무엇을 확인했는지와 사용할 백엔드, 폴백 필요 여부를 설명한 뒤 확인을 기다린다.
   - CLI/API 폴백을 선택했다면 `docs/cli-api-fallback.md`를 읽는다. 설정 오류가 발생했거나 사용자가 API 설정 변경을 명시적으로 요청했을 때만 `docs/image-model-configuration.md`를 읽는다.

5. 승인을 위한 샘플 슬라이드 한 장을 생성한다.
   - 샘플 슬라이드를 생성하거나 승인 처리하기 전에 `docs/outline-style-and-sample.md`를 읽는다.
   - 개요, 스타일, 백엔드가 모두 확정된 후 대표 샘플을 정확히 한 장만 생성한다. 승인 전에는 전체 덱을 생성하지 않는다.
   - 승인 후에는 작업과 서브에이전트가 동일한 경로를 상속하도록 `deck_spec.json`에 `sample_generation_method`를 기록한다.

6. 프로젝트 디렉터리를 만든다.
   - 폴더를 초기화하거나 파일을 조립하기 전에 `docs/project-assembly-and-reporting.md`를 읽는다.
   - 대상 위치가 지정되지 않았다면 현재 작업 디렉터리 또는 원본 파일이 있는 디렉터리를 사용한다.

7. 사용자가 제공한 자산을 준비한다.
   - 논문 그림, 차트, 스크린샷, 로고 또는 기타 필수 자산을 사용하기 전에 `docs/user-supplied-assets.md`를 읽는다.
   - 필수 자산은 엄격한 입력으로 취급하고 생성 전에 슬라이드와 자산 간 매핑을 확인한다.

8. 모든 슬라이드 이미지를 생성한다.
   - 전체 덱 이미지 생성 전에 `docs/slide-generation-and-subagents.md`를 읽는다.
   - `scripts/prepare_slide_prompts.py` 또는 저장된 `prompts/slide_XX.json` 파일로 슬라이드별 작업을 만든다.
   - 모든 최종 이미지는 선택된 백엔드에서 생성하고 번들 상태 스크립트로 기록해야 한다.

9. 슬라이드 서브에이전트에 작업을 배정한다.
   - 슬라이드 작업자를 배정하거나 교체하기 전에 `docs/slide-generation-and-subagents.md`와 `prompts/slide-worker.md`를 읽는다.
   - 가능하면 남은 슬라이드 작업마다 서브에이전트 하나를 사용한다. 필요한 서브에이전트를 생성할 수 없다면 사용자가 워크플로 변경을 허용하지 않는 한 중단하고 블로커를 보고한다.

10. 품질을 점검하고 수정한다.
    - QA 또는 조립 전에 `docs/project-assembly-and-reporting.md`를 읽는다.
    - 조립 전에 모든 슬라이드를 확인한다. 텍스트, 개요 일치 여부, 잘림, 스타일, 원치 않는 페이지 번호, 겹침, 필수 자산을 점검한다.
    - 심각한 실패는 더 엄격한 프롬프트로 다시 생성한다. 국소적인 문제는 가능할 경우 백엔드 편집 기능으로 수정한다.
    - CLI/API 폴백 편집 명령이 필요한 경우 `docs/cli-api-fallback.md`를 읽는다. 편집 결과를 검증한 후에만 최종 슬라이드를 교체한다.

11. 발표자 노트를 작성하고 PPT를 조립한다.
    - `speech.md`를 작성하거나 조립을 실행하기 전에 `docs/project-assembly-and-reporting.md`를 읽는다.
    - `outline.md`가 최종 확정된 덱 개요를 반영하는지 확인한다. `speech.md`의 제목은 `Slide N`과 대응되게 작성한다.
    - 조립 전에 `slide_jobs.json`에서 생성된 슬라이드는 `recorded`, 승인된 샘플은 `accepted` 상태인지 확인한다. 하나라도 `pending`, `dispatched`, `blocked` 상태라면 중단한다.

12. 결과를 보고한다.
    - `docs/project-assembly-and-reporting.md`의 최종 보고 체크리스트를 사용한다.
    - 경로, 슬라이드 수, 사용한 백엔드, 기록된 결과 상태, 제한 사항 또는 블로커를 포함한다.

13. 재사용 가능한 스타일을 저장한다.
    - 현재 덱 스타일 또는 사용자가 제공한 이미지/PDF/PPT/PPTX의 스타일을 저장해 달라는 요청이 있으면 `docs/style-library.md`를 읽는다.
    - 최종 덱이 사용자 지정 또는 수정된 스타일을 사용했다면 `docs/project-assembly-and-reporting.md`에 따라 최종 보고에서 해당 스타일 저장을 먼저 제안한다. 사용자 지정 스타일은 스킬 설치 경로 밖의 `${CODEX_PPT_HOME:-~/.codex-ppt-skill}/references/`에 저장한다.

## 서브에이전트 작업 배정

샘플 승인 이후 런타임이 서브에이전트를 생성할 수 있다면 슬라이드 서브에이전트 사용은 필수다. 메인 에이전트는 작업을 준비하고 상태를 기록한다. 각 작업자는 정확히 하나의 `prompts/slide_XX.json` 작업만 처리하고 선택한 이미지 경로, 백엔드, QA 메모만 반환한다.

배정, 명령, 결과 기록, 블로커, 백엔드 출처 추적 규칙은 `docs/slide-generation-and-subagents.md`를 사용한다. 인계 템플릿은 `prompts/slide-worker.md`를 사용한다.

서브에이전트는 `outline.md`, `deck_spec.json`, 다른 슬라이드 작업, `origin_image/`, `speech.md` 또는 최종 `.pptx`를 수정하면 안 된다. 부모 에이전트가 결과를 기록하고 조립한다.

## 승인 기준

- 출력물이 유효한 `.pptx`여야 한다.
- 예상되는 모든 최종 슬라이드 이미지가 `origin_image/slide_XX.png` 아래에 존재해야 한다.
- 실행 상태에서 `accepted`로 표시된 승인 샘플을 제외하면 모든 최종 슬라이드 이미지는 확정된 백엔드로 생성되고 `record_slide_result.py`를 통해 기록되어야 한다.
- `outline.md`는 승인된 덱 개요를 반영해야 한다.
- 발표자 노트가 필요한 경우 `speech.md`가 존재해야 하며, 조립 과정에서 해당 노트를 PPT에 기록해야 한다.
- `slide_jobs.json`과 `slide_run_state.json`은 최종 상태를 반영해야 한다.
- 필수 원본 이미지는 실제 슬라이드에 보이게 반영하거나, 반영할 수 없다면 블로커를 보고해야 한다.
- 블로커가 있다면 최종 응답에 단계, 슬라이드 ID, 근거 경로, 미완료 사유를 명시한다. 덱이 완료되었다고 표현하지 않는다.

## 참조 맵

- `docs/workflow-gates-and-progress.md`: 승인 게이트, 진행 상태, 완료 근거.
- `docs/backend-selection.md`: 백엔드 결정 규칙 및 확인 문구.
- `docs/outline-style-and-sample.md`: 개요, 스타일, 샘플 규칙 및 프롬프트 예시.
- `docs/user-supplied-assets.md`: 필수 원본 자산의 엄격한 처리 규칙.
- `docs/slide-generation-and-subagents.md`: 작업 생성, 배정, 결과 기록, 블로커, 출처 추적.
- `docs/cli-api-fallback.md`: 폴백 런타임, 생성/편집 명령, 이미지 제한, 문제 해결.
- `docs/image-model-configuration.md`: API 키, 기본 URL, 모델, `.env`; 설정이 필요할 때만 읽는다.
- `docs/project-assembly-and-reporting.md`: 프로젝트 디렉터리, 노트, 조립, 최종 보고, 프롬프트 원칙.
- `prompts/slide-worker.md`: 슬라이드 서브에이전트 인계 템플릿.
- `references/*.md`: 내장 시각 스타일 레퍼런스. 사용자 지정 스타일은 `${CODEX_PPT_HOME:-~/.codex-ppt-skill}/references/`에 저장되며 이름이 같으면 내장 스타일보다 우선한다.

## 문서 및 업데이트

소스, 문서, 설치, 설정, 예시는 [ningzimu/codex-ppt-skill](https://github.com/ningzimu/codex-ppt-skill)을 참조한다.
