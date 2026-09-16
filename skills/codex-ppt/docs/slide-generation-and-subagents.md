# 슬라이드 생성 및 서브에이전트

전체 덱 이미지를 생성하거나 슬라이드 작업을 준비하거나 서브에이전트에 배정하거나 결과/블로커를 기록하기 전에 읽는다.

## 최종 슬라이드 이미지 생성

선택된 이미지 백엔드로 슬라이드당 이미지 한 장을 생성한다. 최종 `slide_XX.png`는 반드시 내장 이미지 도구 또는 `scripts/image_gen.py`가 직접 생성해야 한다. 프로그램 렌더링이나 텍스트 오버레이 혼합 방식은 최종 슬라이드 이미지 생성에 사용할 수 없다.

개요, 시각 스타일, 이미지 백엔드, 샘플 슬라이드가 모두 승인된 뒤 다음 최종 산출물을 만든다.

- `deck_spec.json`
- `prompts/slide_XX.json`
- `speech.md`

개요 승인 전에는 최종 산출물을 만들지 않는다. 승인 전 계획 파일이 꼭 필요하면 `.draft.` 파일명을 사용하고 승인 후 동기화한다.

`prepare_slide_prompts.py`를 실행하기 전에 `deck_spec.json`에 승인된 샘플의 `sample_generation_method`를 넣는다. 도우미 스크립트가 이 정보를 각 `prompts/slide_XX.json`과 `slide_jobs.json`에 복사해 작업자가 샘플과 정확히 같은 백엔드, 도구, 모드, 이미지 컨텍스트 준비 방식, 출력 제약을 따르게 한다.

전체 생성 전에는 다음 명령으로 슬라이드별 구조화 작업을 준비하는 것을 우선한다.

```bash
~/.codex-ppt-skill/.venv/bin/python {skill_root}/scripts/prepare_slide_prompts.py \
  --spec {base_dir}/{deck_name}/deck_spec.json \
  --out-dir {base_dir}/{deck_name} \
  --selected-backend "<확정된 백엔드 이름>" \
  --force
```

생성 파일:

```text
{base_dir}/{deck_name}/
├── prompts/
│   ├── slide_01.json
│   ├── slide_02.json
│   └── ...
├── slide_jobs.json
└── slide_run_state.json
```

각 `prompts/slide_XX.json`은 독립적으로 실행 가능한 단일 슬라이드 작업이어야 한다. 슬라이드 번호, 제목, 출력 파일명, 입력 이미지 목록, 컨텍스트 이미지 필요 여부, 전체 프롬프트를 포함한다. 별도의 작업 매니페스트는 사용자가 요청하지 않는 한 만들지 않는다.

부모 에이전트는 배정 전에 필요한 컨텍스트를 완전하게 포장해야 한다. 서브에이전트는 배정된 한 장의 작업, 명시적으로 전달된 이미지, 인계 문구만 본다고 가정한다. 원문 전체, 전체 개요, 앞뒤 슬라이드, 부모 대화에만 있는 개념을 알고 있다고 가정하지 않는다.

여러 슬라이드나 원문 전체의 의미에 의존하는 정보는 프롬프트 준비 전에 `deck_spec.json`에 명시한다.

- `deck_context`: 원본 요약, 핵심 주장, 용어 목록, 분류 체계, 등장인물, 정의, 연표, 고정 명칭처럼 여러 슬라이드가 공유하는 기준 정보
- `local_context`: 특정 슬라이드에만 필요한 사실, 목록, 비교 항목, 이전 결론의 구체적 내용 등
- `여섯 가지 특징`, `위 프레임워크`, `이전 결론` 같은 암시적 참조는 실제 목록이나 정의로 풀어 쓴다.

`slide_jobs.json`은 작업 배정 상태 파일이다. 슬라이드별 프롬프트 작업, 최종 출력 경로, 상태, 선택 백엔드, 샘플 생성 방식, 서브에이전트 배정 메타데이터, 결과 출처, 블로커 상태를 기록한다. 상태값을 손으로 수정하지 말고 번들 상태 스크립트를 사용한다.

`deck_spec.json`의 `required_images`는 구조화 객체 또는 Markdown 이미지 참조 문자열을 사용할 수 있다. 원본 이미지의 역할과 보존 조건이 작업 프롬프트에 명확히 남아야 한다.

각 슬라이드 프롬프트는 캔버스, 스타일, 레이아웃, 텍스트, 시각 요소, 제약 조건을 구분한 구조화된 시각 브리프로 만든다. 덱 전체의 시각 정체성은 유지하되 페이지 의미에 따라 레이아웃을 바꾼다.

권장 페이지 유형:

- 표지 / 섹션 구분
- 배경 / 문제 정의
- 프로세스 / 타임라인
- 비교 / 트레이드오프
- 데이터 / 근거 / KPI
- 아키텍처 / 워크플로 다이어그램
- 요약 / 결론 / 다음 단계

모든 슬라이드를 똑같은 카드형 레이아웃으로 만들지 않는다. 각 슬라이드의 `layout.intent`에 해당 구성을 선택한 이유를 적는다.

수동으로 프롬프트를 준비하더라도 생성 전 `{base_dir}/{deck_name}/prompts/slide_XX.json`에 전체 작업을 저장해야 한다. 작업에는 `prompt`, `out`, `input_images`가 포함되어야 하고 승인 샘플 스타일 참고와 슬라이드별 필수 원본 이미지의 역할도 명시한다.

## 서브에이전트를 이용한 병렬 슬라이드 생성

샘플 슬라이드가 승인되고 사용자가 전체 생성에 동의한 뒤, 현재 런타임이 서브에이전트를 생성할 수 있다면 남은 슬라이드마다 서브에이전트를 사용하는 것이 필수다. 편의를 이유로 메인 에이전트가 나머지 덱을 순차 생성하지 않는다. 서브에이전트를 만들 수 없다면 배정 단계에서 멈추고 블로커를 보고한다. 사용자가 워크플로 변경을 명시적으로 허용한 경우만 예외다.

상태 스크립트를 작업 배정 계약으로 사용한다. 실제 스크립트 기록이 있어야 배정/완료된 것으로 본다.

### 부모 에이전트 책임

- `outline.md`, `deck_spec.json`, `prompts/`, `origin_image/`, QA, `speech.md`, 최종 PPT 조립을 소유한다.
- 배정 전 `prepare_slide_prompts.py`를 실행하거나 동등한 완전한 작업 JSON과 `slide_jobs.json`을 작성한다.
- 각 배치 전에 `slide_job_status.py`로 배정 가능한 슬롯과 대기 중 슬라이드 ID를 확인한다.
- 승인 샘플 슬라이드를 가능한 모든 비샘플 작업에 스타일 전용 입력으로 넣는다.
- 각 작업이 독립적으로 실행 가능하도록 `deck_context`와 `local_context`를 충분히 채운다.
- `sample_generation_method`가 `deck_spec.json`, 모든 `prompts/slide_XX.json`, `slide_jobs.json`에 존재하도록 한다.
- 승인 샘플을 다시 만들지 않을 경우 `deck_spec.json`에서 `sample_approved: true` 또는 `approved_sample: true`로 표시해 도우미가 최종 이미지 존재 시 `accepted`로 기록하게 한다.
- 내장 이미지 모드에서는 위임 전에 필수 로컬 원본 이미지를 부모가 먼저 실제로 검사한다.
- CLI/API 폴백에서 입력 이미지를 붙일 수 없는 슬라이드는 텍스트 전용 대체물로 위임하지 않는다.
- `dispatch_slots_available` 범위에서 슬라이드당 작업자 한 명을 생성한다.
- 작업자 생성 직후 실제 agent id와 프롬프트 경로를 `record_slide_dispatch.py`로 기록한다.
- 작업자 반환 후 선택 결과를 시각 검토하고 `record_slide_result.py`로 `origin_image/slide_XX.png`에 복사 및 백엔드 출처 기록을 수행한다.
- 백엔드나 필수 입력 이미지를 사용할 수 없으면 `record_slide_blocker.py`로 기록한다.

### 서브에이전트 책임

- 배정된 `prompts/slide_XX.json` 하나만 읽는다.
- 선택된 이미지 백엔드만 사용한다.
- 작업에 포함된 `sample_generation_method`를 그대로 따른다.
- 최종 슬라이드는 이미지 생성 백엔드로 직접 만든다. 로컬 드로잉, HTML/SVG/canvas 스크린샷, Pillow, python-pptx/PptxGenJS 레이아웃, 수동 합성으로 만들지 않는다.
- 승인 샘플은 스타일 참고로만 사용한다.
- 필수 원본 이미지는 엄격한 입력 자산으로 취급하고 프롬프트의 보존 규칙을 지킨다.
- 반환 전에 텍스트 품질, 스타일 일관성, 필수 이미지 포함 여부, 레이아웃 문제를 검사한다.
- 선택된 원본 생성 이미지 경로, 사용 백엔드, 한 문장 QA 메모만 반환한다.

서브에이전트는 `outline.md`, `deck_spec.json`, 다른 작업 파일, `origin_image/`, `speech.md`, 최종 `.pptx`를 수정하지 않는다.

### 배정/결과/블로커 기록

```bash
~/.codex-ppt-skill/.venv/bin/python {skill_root}/scripts/slide_job_status.py \
  {base_dir}/{deck_name}

~/.codex-ppt-skill/.venv/bin/python {skill_root}/scripts/record_slide_dispatch.py \
  {base_dir}/{deck_name} \
  --slide slide_02 \
  --agent-id <agent id> \
  --agent-nickname "<nickname if available>" \
  --prompt-file prompts/slide_02.json
```

```bash
~/.codex-ppt-skill/.venv/bin/python {skill_root}/scripts/record_slide_result.py \
  {base_dir}/{deck_name} \
  --slide slide_02 \
  --agent-id <agent id> \
  --backend-used "built-in image tool" \
  --selected-source /absolute/path/to/generated/slide_02.png \
  --qa-note "한글이 선명하고 승인 샘플과 스타일이 일치함."
```

```bash
~/.codex-ppt-skill/.venv/bin/python {skill_root}/scripts/record_slide_blocker.py \
  {base_dir}/{deck_name} \
  --slide slide_02 \
  --agent-id <agent id> \
  --reason "작업자 환경에서 선택된 이미지 백엔드를 사용할 수 없음"
```

인계 템플릿은 `../prompts/slide-worker.md`를 사용한다.

최종 이미지는 다음 위치와 이름 규칙을 따른다.

```text
{base_dir}/{deck_name}/origin_image/slide_01.png
{base_dir}/{deck_name}/origin_image/slide_02.png
...
```

- 슬라이드 순서대로 `slide_01.png`, `slide_02.png`, ... 형태로 이름을 붙인다.
- 일반적인 덱에서는 두 자리 0 패딩을 사용한다.
- 승인 샘플은 이미 올바른 `slide_XX.png` 이름이어야 하며 그대로 재사용한다.
- 거부된 변형, 초안, 참고 이미지는 `origin_image/` 밖에 둔다.
- 조립 전에 예상되는 모든 `slide_XX.png`가 존재하고 누락/초과 파일이 없으며 `slide_job_status.py`에서 모든 비샘플 작업이 `recorded`인지 확인한다.

한글 덱에서는 이미지 백엔드 프롬프트에 한글 텍스트를 정확하고 선명하게 렌더링하고 글자 깨짐을 피하라고 명시한다.
