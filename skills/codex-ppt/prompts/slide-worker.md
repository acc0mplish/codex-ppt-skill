# 슬라이드 작업자 프롬프트

샘플 슬라이드가 승인되고 전체 덱 생성을 허가받은 뒤 슬라이드 서브에이전트에 작업을 배정할 때 이 템플릿을 사용한다.

```text
이 codex-ppt 덱의 <N>번 슬라이드를 생성한다.

덱 디렉터리: <absolute deck dir>
슬라이드 작업 파일: <absolute deck dir>/prompts/slide_<NN>.json
부모 에이전트가 소유하는 최종 출력 위치: <absolute deck dir>/origin_image/slide_<NN>.png
선택된 이미지 백엔드: <built-in image tool OR CLI/API fallback>
승인된 샘플에서 복사한 생성 방식:
- backend_used: <부모가 기록한 정확한 백엔드 이름>
- tool_name: <image_gen OR image_generate OR scripts/image_gen.py>
- mode: <generate OR edit>
- model/config: <model, size, quality 또는 노출되지 않으면 "built-in default">
- prompt_source: <승인 샘플 프롬프트 출처>
- input_context_preparation: <로컬 이미지를 표시하거나 첨부한 방식>
- approved_sample_path: <승인된 origin_image/slide_XX.png 절대 경로>
- handoff_rule: 동일한 backend/tool/mode를 사용하고 사용할 수 없으면 blocker를 반환
부모가 이미 준비한 입력 이미지:
- <absolute path> - 승인 샘플 슬라이드 스타일 참고; 스타일만 맞추고 레이아웃은 복사하지 않음
- <absolute path> - 필수 입력 자산; 라벨/데이터/화살표/내용을 보존

JSON 작업 파일을 읽고 `prompt` 필드를 정확히 따른다. 선택된 이미지 백엔드와 기록된 샘플 생성 방식만 사용한다.
최종 슬라이드 후보는 반드시 선택된 이미지 생성 백엔드를 실제로 호출해 만든다.
- 내장 모드: 내장 이미지 생성/편집 도구 사용
- CLI/API 폴백 모드: 저장된 작업 프롬프트와 필수 입력 이미지를 사용해 `scripts/image_gen.py` 실행

최종 슬라이드 이미지 생성에 금지되는 방식:
- 로컬 드로잉/렌더링 스크립트
- Pillow로 생성한 슬라이드
- SVG, HTML/CSS, canvas 스크린샷
- python-pptx/PptxGenJS/네이티브 PPT 레이아웃 스크린샷
- 텍스트, 카드, 차트, 이미지의 수동 합성 오버레이

선택된 이미지 백엔드를 사용할 수 없으면 저품질 대체물을 만들지 말고 `blocker=<reason>`을 반환한다.
기록된 샘플 생성 방식을 따를 수 없으면 도구를 바꾸지 말고 `blocker=<reason>`을 반환한다.
슬라이드 작업 파일, origin_image, speech.md를 수정하거나 PPT를 조립하지 않는다.

반환 전에 눈으로 확인한다.
- 한글 텍스트가 읽기 쉽고 깨지지 않았는지
- 스타일이 승인 샘플과 일치하는지
- 필수 원본 이미지가 실제로 포함되고 비슷하게 다시 그린 이미지로 대체되지 않았는지
- 중요한 내용이 겹치거나 잘리지 않았는지

다음만 반환한다.
backend_used=<built-in image tool OR scripts/image_gen.py>
selected_source=/absolute/path/to/$CODEX_HOME/generated_images/.../ig_*.png
qa_note=<한 문장>
```
