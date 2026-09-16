# 이미지 모델 설정

로컬 API/CLI 폴백이 필요하고 런타임 설정이 없거나 변경해야 할 때만 이 문서를 사용한다.

`.env`를 수동으로 파싱하지 않는다. 폴백 CLI가 공유 설정을 자동으로 읽으므로 먼저 폴백 명령을 실행하고, CLI가 설정 누락이나 오류를 보고할 때만 이 문서를 사용한다.

다음 경우에만 사용자에게 설정을 추가하거나 변경하도록 요청한다.

- 폴백 CLI가 `OPENAI_API_KEY` 누락을 보고함
- 사용자가 API 키, 기본 URL, 모델 변경을 명시적으로 요청함
- 실제 API 호출이 인증, 권한, 기본 URL 또는 모델 없음 오류로 실패함

## 설정이 필요한 경우

이미지 API 설정은 API/CLI 폴백 이미지 생성에만 필요하다.

대표 사례:

- Codex에서 서드파티 API 또는 OpenAI 호환 프록시를 사용함
- Claude Code, OpenClaw, Hermes Agent 등 Codex 내장 이미지 도구가 없는 환경에서 스킬을 사용함

Codex 내장 이미지 도구를 사용할 수 있다면 `gpt-image-2.5-flare` 설정을 요구하지 않는다.

## 필수 및 선택 값

- `OPENAI_API_KEY`: 실제 API/CLI 폴백 호출에 필수
- `OPENAI_BASE_URL`: 선택. 미설정 시 공식 OpenAI API 사용, 설정 시 해당 서드파티 공급자 기본 URL 사용
- `CODEX_PPT_IMAGE_MODEL`: 선택. 기본값 `gpt-image-2.5-flare`; Sunburst는 `gpt-image-2.5-sunburst`, 또는 공급자가 지원하는 모델명 사용

제공된 API 설정은 `scripts/codex_ppt_runtime.py config --api-key`로 저장한다. 설정 명령은 `~/.codex-ppt-skill/.env`를 작성한다.

## 공식 OpenAI 예시

```bash
python3 {skill_root}/scripts/codex_ppt_runtime.py config \
  --api-key "your-api-key" \
  --model gpt-image-2.5-flare
```

## OpenAI 호환 공급자 예시

```bash
python3 {skill_root}/scripts/codex_ppt_runtime.py config \
  --api-key "your-provider-api-key" \
  --base-url "https://xxxx.example.com/v1" \
  --model gpt-image-2.5-flare
```

이는 다음과 같은 런타임 설정을 만든다.

```env
OPENAI_API_KEY=your-provider-api-key
OPENAI_BASE_URL=https://xxxx.example.com/v1
CODEX_PPT_IMAGE_MODEL=gpt-image-2.5-flare
```

OpenAI 호환 공급자의 `OPENAI_BASE_URL`은 일반적으로 공급자의 `/v1` 루트에서 끝나야 한다. `/images/generations`, `/images/edits` 같은 최종 엔드포인트를 넣지 않는다. 폴백 CLI가 OpenAI SDK를 통해 이미지 생성/편집 경로를 덧붙인다.

공급자가 별도 모델명을 문서화한 경우에만 그 이름을 사용하고, 그렇지 않으면 `gpt-image-2.5-flare`를 우선한다.

## AtlasCloud 예시

AtlasCloud에서는 `--model`에 기본 모델명을 지정한다. CLI가 생성/편집 라우트를 내부에서 선택한다. 아래 예시는 알려진 `gpt-image-2` 경로를 유지한 것이므로 2.5 모델을 선택하기 전에는 공급자 문서를 확인한다.

```bash
python3 {skill_root}/scripts/codex_ppt_runtime.py config \
  --api-key "your-atlascloud-api-key" \
  --base-url "https://api.atlascloud.ai/api/v1/model" \
  --model openai/gpt-image-2
```

## 런타임 설정 파일

설정은 다음에 저장된다.

```text
~/.codex-ppt-skill/.env
```

파일 권한은 `0600`으로 생성되며 Codex, Claude Code, OpenClaw, Hermes Agent 등 로컬 에이전트가 공유한다.

프로세스 환경변수는 `.env` 값을 덮어쓴다. 명령줄의 `--model`은 해당 명령에서만 `CODEX_PPT_IMAGE_MODEL`을 덮어쓴다.
