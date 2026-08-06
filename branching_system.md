# 분기 시스템 뼈대 — 설계 검토 (구현 아님, 2026-08-06)

코드는 아직 건드리지 않았음. 현재 대화/세이브 구조를 실제로 읽고, 그 위에 분기를 얹으려면 무엇이 걸리는지 정리.

## 지금 구조 (분기가 왜 하나도 없는지)

- `dialogue_line`(id, characterName, expression, dialogue_KR), `dialogue_sequence`(characterName, eventName, startId, endId), `dialogue_flow_data`(flowName, sequences) — `Assets/Features/Scripts/Dialogue/dialogue_line.cs:4-33`. 분기/조건/선택지 필드는 전혀 없음.
- `ShowNext()`(`dialogue_scene_controller_base.cs:143-160`)가 `(sequenceIndex, currentLineId)`로 줄을 찾고, `Advance()`(174-186)가 `currentLineId++` 하다가 `sequence.endId`에 닿으면 `sequenceIndex++` — **순수 산술 증가**. "다음 줄이 뭐냐"를 결정하는 지점이 이 두 곳뿐이고, 둘 다 `private`·비-virtual이라 분기용 후크가 없음.
- 재개(resume)는 `dialogue_progress_resolver.TryResolve`(15-43)가 `flowResourcePath` 일치 + `sequenceIndex` 유효 + `currentLineId`가 그 시퀀스의 `[startId,endId]` 안인지만 검증. **순차 진행을 강제하는 검증이 아니라 "이 위치가 유효한가"만 봄** — 이건 나중에 중요한 포인트(아래 3번).
- 세이브 파일(`game_save_data.cs`): `dialogueProgress`(flowResourcePath, sequenceIndex, currentLineId, completed, characterName, lineSummary) + `clueInventory`(obtainedClueIds) + `inventory`(ownedItemIds). **키-값 형태의 범용 플래그/상태 저장소가 없음.**
- 참고로 확인해본 것: `AddClue`/`AddItem`은 지금도 프로젝트 어디서도 호출되지 않음(여전히 트리거 미구현 상태) — [[project-cluebook-todo]]/[[project-inventory-todo]]에서 이미 나온 얘기 그대로.

---

## 1. 분기점이 어디서 발생하는가

물어본 두 가지(스토리 진행 중 vs 이벤트로 특정 함수 호출)는 **둘 다 필요하고, 서로 다른 레이어**라고 봄:

- **스토리 내장형**: `dialogue_line`에 분기 선택지가 붙는 경우. `ShowNext()`가 그 줄에 도달하면 평소처럼 자동 진행하지 않고 멈춰서 선택을 기다려야 함. 이건 대화 데이터 스키마 확장이 필요함 — 지금 `dialogue_line`엔 그런 필드가 아예 없음.
- **이벤트 트리거형**: 특정 함수 호출(미니게임 결과, 특정 조건 충족 등)로 대화 흐름과 무관하게 분기 팝업이 뜨는 경우. 이건 대화 스키마와 무관하게, 그냥 "N개 선택지를 보여주고 고른 걸 콜백으로 알려주는" 범용 UI 하나만 있으면 됨.

**뼈대 단계 추천**: 이벤트 트리거형(범용 팝업)을 먼저 만드는 게 낫다고 봄 — 스토리 내장형은 결국 이 팝업을 "대화 진행 로직이 자동으로 호출해주는" 특수 케이스일 뿐이라서, 범용 팝업 → 대화 진입점 순으로 가면 재작업이 줄어듦.

**놓치기 쉬운 문제 — 분기 대상 주소 지정 방식**: 지금 시퀀스는 배열 인덱스(`sequenceIndex`)로만 식별됨. 나중에 시퀀스를 하나 끼워넣거나 순서를 바꾸면 인덱스가 다 밀려서, 하드코딩된 분기 타겟이 조용히 다른 곳을 가리키게 됨. 분기 타겟을 도입하기 전에 **시퀀스에 안정적인 문자열 id(라벨)를 붙이고 그걸로 점프하게 할지, 지금처럼 배열 인덱스로 할지**를 먼저 정해야 함 — 나중에 바꾸면 이미 작성된 분기 데이터를 전부 다시 손봐야 해서 지금 결정하는 게 훨씬 쌈.

## 2. 선택 시 이벤트 처리 방식

이미 있는 관례를 그대로 확장하는 게 제일 자연스러움: `confirm_popup_controller.Show(message, Action onConfirm, Action onCancel)`(2개 고정 슬롯)를 N개로 일반화한 `branch_popup_controller.Show(message, List<branch_option>)` 같은 형태 — 각 옵션은 `{ text, Action onSelected }`. 팝업 자체는 "선택하면 그 옵션의 Action을 실행한다"만 알고, **그 Action이 실제로 뭘 하는지(함수 호출/씬 전환/패널 토글)는 호출하는 쪽이 결정** — `confirm_popup_controller`가 "확인"의 의미를 모르는 것과 동일한 원칙.

두 갈래 선택지가 있음:
- **코드 기반** (뼈대 단계 추천): 분기별 Action을 C# 코드에서 직접 구성. 새 분기 하나 추가할 때마다 코드 수정 필요하지만, 지금 있는 유일한 유사 패턴(confirm popup)과 결이 같고 별도 인터프리터가 안 필요함.
- **데이터 기반**: JSON에 `effectType: "ChangeScene" / "GrantClue" / "SetFlag"` 같은 문자열 + 파라미터를 넣고, 문자열→실제 Action으로 매핑하는 작은 디스패치 테이블을 만듦. 분기가 많아지고 기획자가 코드 없이 추가해야 하는 시점이 오면 이쪽으로 옮기는 게 맞음. 지금 뼈대 단계에서는 과할 수 있음.

**놓치기 쉬운 문제 — 취소 가능 여부**: `ConfirmPopup`은 ESC/"아니오"로 취소가 정상 동작(=`ui_overlay_state`에 등록돼서 ESC가 취소 콜백을 호출). 스토리 분기 선택은 보통 **취소가 없어야 함** — 뭐라도 하나는 골라야 진행되는 게 정상이라서, `ui_overlay_state`의 "ESC로 top overlay 닫기" 관례를 그대로 재사용하면 안 될 가능성이 높음. 이벤트 트리거형(예: "정말 나가시겠습니까?")은 취소 가능해야 할 수도 있어서, 팝업 자체를 "취소 가능/불가능"을 옵션으로 가지게 설계하는 게 좋아 보임.

## 3. 세이브 파일 영향

**좋은 소식**: 분기로 점프한 위치 자체는 지금 스키마로도 충분함 — `dialogue_progress_resolver.TryResolve`는 순차 진행을 강제하지 않고 "이 `(sequenceIndex, currentLineId)`가 유효한 위치인가"만 검증하기 때문에, 분기로 어디로 튀든 그 지점의 `(sequenceIndex, currentLineId)`만 저장하면 재개는 지금 코드 그대로 동작함. **위치 저장용 필드 추가는 필요 없음.**

**진짜 빈 자리**: "이 분기에서 뭘 선택했는지"를 나중에 참조해야 하는 순간(다른 대사가 그 선택에 따라 달라지거나, 한 번 고른 분기를 다시 못 고르게 막거나) 저장할 **플래그/상태 저장소가 세이브 파일에 전혀 없음**. 이건 새로 만드는 게 아니라 — 이미 [[project-cluebook-todo]](clue 보상 필드)와 [[project-inventory-todo]](item 보상 필드), [[project-worldmap-marker-todo]](`setsLocationId`)에서 각각 따로 논의됐던 **같은 빈 자리**임. 분기까지 오니 세 번째로 같은 요구가 나온 셈 — `game_save_data`에 범용 `Dictionary<string,string> flags` 하나 + `game_save_manager.SetFlag/GetFlag/HasFlag`를 만들어서 네 기능(단서 보상/아이템 보상/위치 갱신/분기 선택 기록)이 전부 이걸 같이 쓰는 게, 각자 필드를 따로 만드는 것보다 훨씬 나을 것 같음. 이 넷 중 어느 걸 먼저 만들든 이 결정(범용 flags 저장소를 만들지 말지)부터 정하는 게 순서상 맞다고 봄.

## 4. 그 외 놓치기 쉬운 것들 (직접 안 물어봤지만 관련됨)

- **Skip과의 충돌**: `SkipChapter()`(`dialogue_scene_controller_skip.cs`)는 줄 단위로 도는 게 아니라 `EndScene()`을 바로 호출함 — 즉 지금 구조에서 스킵은 **분기 줄을 아예 거치지 않고 통과**함. 분기점이 있는 챕터를 스킵하면 어떻게 할지(기본 옵션 자동 선택? 분기가 남아있으면 스킵 자체를 막을지?) 결정 필요.
- **Auto-play / 배속과의 충돌**: `A`(Auto)와 왼쪽 Ctrl(배속)은 둘 다 타이머로 `ShowNext()`를 자동 호출함 — 분기 줄에 도달했을 때 이 둘이 멈추고 선택을 기다리게 만드는 가드가 새로 필요함(지금 `ui_overlay_state.IsAnyOverlayOpen` 가드랑 비슷한 위치에 `isChoicePending` 같은 플래그 추가하는 식이 자연스러워 보임).
- **파일 구조**: `_skip.cs`/`_autoplay.cs`/`_fastforward.cs`처럼 partial class로 관심사를 나눠온 관례가 있으니, `dialogue_scene_controller_branch.cs` 하나 추가하는 게 일관성 있어 보임.

## 뼈대 단계에서 먼저 정해야 할 것 (요약)

1. 분기 타겟 주소 지정: 시퀀스 배열 인덱스 vs 안정적 문자열 라벨
2. 선택 효과: 코드 기반 Action vs 데이터 기반 effect-string — 뼈대는 코드 기반 추천
3. 취소 가능 여부를 팝업 옵션으로 둘지 (스토리 분기=불가, 이벤트 팝업=가능할 수도)
4. 범용 `flags` 저장소를 세이브에 만들지 — 만든다면 클루/아이템 보상, 월드맵 위치와 같이 쓰는 것 전제로 설계
5. Skip이 분기를 만났을 때 동작, Auto/배속을 분기 줄에서 멈추는 가드
