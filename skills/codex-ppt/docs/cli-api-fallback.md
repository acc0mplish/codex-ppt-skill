# CLI/API 폴백

CLI/API 폴백을 선택하고 사용자에게 확인받은 뒤에만 이 문서를 사용한다. 백엔드 선택 규칙은 `SKILL.md`가 담당하고, 이 문서는 폴백 명령, 런타임 준비, 이미지 입력 제한, 편집, 투명 배경, 문제 해결을 다룬다.

`{skill_root}`는 `SKILL.md`가 있는 디렉터리를 의미한다.

## 런타임 준비

CLI/API 폴백 명령은 공유 런타임 환경을 사용한다. `scripts/assemble_ppt.py` 또는 폴백 이미지 명령을 실행하기 전에 `~/.codex-ppt-skill/.venv/bin/python`이 존재하고 스크립트 의존성을 import할 수 있는지 확인한다. 없거나 실패하면 다음으로 환경을 생성/갱신한다.

```bash
python3 {skill_root}/scripts/codex_ppt_runtime.py bootstrap
```

이는 스킬 내부 준비 단계다. 의존성 설치 실패로 사용자의 승인이나 문제 해결이 필요한 경우가 아니라면 사용자에게 직접 실행을 요구하지 않는다.

폴백 CLI는 `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `CODEX_PPT_IMAGE_MODEL`을 위해 `~/.codex-ppt-skill/.env`를 자동으로 읽는다. `.env`를 수동 파싱하지 않는다. 폴백 CLI가 실제 설정 오류를 반환했거나 사용자가 설정 변경을 요청했거나 실제 API 호출에서 인증/권한/기본 URL/모델 가용성 오류가 발생했을 때만 `image-model-configuration.md`를 읽는다.

## 슬라이드 한 장 생성

기본 명령:

```bash
~/.codex-ppt-skill/.venv/bin/python {skill_root}/scripts/image_gen.py generate \
  --model gpt-image-2.5-flare \
  --prompt-file {prompt_file} \
  --size 2560x1440 \
  --quality medium \
  --out {base_dir}/{deck_name}/origin_image/slide_01.png
```

폴백 CLI 기본 모델은 `gpt-image-2.5-flare`다. Sunburst를 쓰려면 `--model gpt-image-2.5-sunburst`를 지정한다. 공급자 접두사가 붙은 모델명과 이전 GPT Image 모델도 받을 수 있지만 사용 전에 공급자 지원 여부를 확인한다.

저장된 `prompts/slide_XX.json`에서 생성할 때는 작업에 입력 이미지가 필요하지 않은 경우에만 `prompt` 필드만 사용한다.

```bash
python3 -c 'import json, pathlib; print(json.loads(pathlib.Path("{base_dir}/{deck_name}/prompts/slide_01.json").read_text())["prompt"])' | \
~/.codex-ppt-skill/.venv/bin/python {skill_root}/scripts/image_gen.py generate \
  --prompt-file - \
  --size 2560x1440 \
  --quality medium \
  --out {base_dir}/{deck_name}/origin_image/slide_01.png
```

이 텍스트 전용 경로를 쓰기 전에 해당 `prompts/slide_XX.json`을 확인한다. `input_images`가 비어 있지 않거나 `requires_context_images`가 `true`라면 위 명령만으로는 충분하지 않다. 내장 이미지 도구처럼 필요한 이미지가 컨텍스트에 보이거나 모든 원본 이미지를 전달할 수 있는 편집/이미지 입력 경로를 사용한다. 가능한 경로가 없으면 중단하고 백엔드 전환 여부를 사용자에게 확인한다. 필수 입력 자산을 무시한 텍스트 전용 대체물을 생성하지 않는다.

## 기능 및 크기

폴백 CLI는 다음을 지원한다.

- `generate`: 프롬프트에서 이미지 생성
- `edit`: 기존 이미지 하나 이상을 선택적으로 마스크와 함께 편집

기본 출력은 2K 16:9 가로형 `2560x1440`, 품질 `medium`이다. GPT Image 2.5는 `xhigh`, `max`도 지원한다. 4K 가로 슬라이드는 사용자가 4K를 요청했거나 텍스트가 많은 슬라이드에서 더 선명한 결과가 필요하거나 기본 결과가 흐릴 때만 `--size 3840x2160 --quality high`를 사용한다. 세로형은 사용자가 요청한 경우에만 `--size 2160x3840`을 사용한다. GPT Image 2.5에서 `2560x1440` 초과 출력은 실험적일 수 있으므로 실제 크기와 품질을 검사한다.

## 슬라이드 편집

슬라이드 대부분이 맞고 국소적인 문제만 있다면 선택한 백엔드의 편집 기능을 사용할 수 있을 때 사용한다. CLI/API 폴백에서는 다음을 사용한다.

```bash
~/.codex-ppt-skill/.venv/bin/python {skill_root}/scripts/image_gen.py edit \
  --image {slide_path} \
  --prompt {edit_prompt} \
  --out {new_slide_path}
```

편집 결과를 검증한 뒤에만 최종 슬라이드를 교체한다.

## 투명 배경

- GPT Image 2.5 Flare/Sunburst는 호환 API에서 `--background transparent --output-format png` 또는 `webp`를 지원한다. JPEG는 투명도를 보존하지 못한다.
- 현재 AtlasCloud 어댑터는 `background`를 전달하지 않고 PNG/JPEG만 허용하므로 해당 어댑터에서 네이티브 투명 배경을 보장하지 않는다.
- `gpt-image-2`는 네이티브 투명 배경을 지원하지 않지만 GPT Image 1/1.5는 지원한다. 합의 없이 선택 모델을 바꾸지 않는다.
- 내장 모드에서는 도구가 네이티브 투명도를 지원하면 사용하고, 그렇지 않으면 필요할 때 단색 크로마키 배경과 `scripts/remove_chroma_key.py`를 사용한다.

## 조립 및 진단

`assemble_ppt.py`는 `16:9`와 `4:3`을 지원한다. 사용자가 다르게 요청하지 않으면 `16:9`를 사용한다.

폴백 API 접근 문제를 해결할 때만 다음 진단 명령을 실행한다.

```bash
python3 {skill_root}/scripts/codex_ppt_runtime.py doctor --check-api
```
