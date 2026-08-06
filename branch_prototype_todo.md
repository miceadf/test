# 분기 시스템 프로토타입 (Prototype_Select) — Todo

실제 통합 지점이 아직 정해지지 않아서, 실제 프로젝트 파일은 하나도 건드리지 않고
`Assets/Prototype_Select/`에 완전히 격리해서 구현/검증함. [branching_system.md](./branching_system.md) 설계 검토를 코드로 옮긴 것.

## 현재 어떻게 작동하는가 (구현 완료, Play 모드에서 검증됨)

**위치** (전부 `Assets/Prototype_Select/`, 실제 `Assets/Features/`와 같은 하위 경로):
- `Features/Scripts/Dialogue/dialogue_line.cs` — `dialogue_choice_proto`/`dialogue_line_proto`/`dialogue_sequence_proto`/`dialogue_flow_data_proto`. 실제 `dialogue_line`/`dialogue_sequence`(`Assets/Features/Scripts/Dialogue/dialogue_line.cs`)를 복제하고 분기용 필드(`choices`, `sequenceId`)만 추가한 것. 이름에 `_proto`가 붙은 이유는 실제 클래스와 이름이 겹치면 컴파일이 깨지기 때문 — **실제 게임 코드와는 전혀 연결되어 있지 않음.**
- `Features/Scripts/Save/game_save_manager.cs` — `game_save_flags_prototype`(static). 범용 `Dictionary<string,string>` 플래그 저장소의 프로토타입. 메모리에만 있고 세이브 파일엔 안 씀.
- `Features/Scripts/UI/branch_popup_controller.cs` — `confirm_popup_controller`를 N개 선택지로 일반화한 팝업. 옵션 버튼은 `instance_guide_controller.BuildCurrent()`처럼 프리팹 없이 코드로 동적 생성. `cancelable` 플래그로 ESC 취소 가능/불가능을 나눔.
- `Features/Scripts/SceneController/dialogue_scene_controller_branch.cs` — `dialogue_scene_controller_branch_prototype`(독립 MonoBehaviour). 하드코딩된 2줄+분기 하나짜리 테스트 대화를 직접 돌린다. 실제로는 `dialogue_scene_controller_base`의 partial 파일이 될 자리지만, 통합 방식이 안 정해진 지금은 완전히 별도 클래스로 분리해서 실제 대화 시스템에 어떤 영향도 주지 않게 했다.
- 테스트용 씬: `Scenes/BranchPrototype_scene.unity` (Canvas/EventSystem + LineText/StatusText/BranchPopupPanel).

**Play 모드에서 실제로 확인된 것**:
1. Space로 대사 진행 → 선택지 있는 줄에 도달하면 자동으로 멈추고 팝업 표시 (2개 버튼 동적 생성, 한글 폰트 정상 표시).
2. 선택 대기 중에는 시뮬레이션한 Auto-play(`A`)와 Skip(`S`) 둘 다 실제로 막힘 — `isChoicePending` 가드가 의도대로 동작.
3. 옵션 클릭 → 팝업 닫힘 → 지정한 flag(`branch_intro_choice=go`) 저장 → 목표 시퀀스/라인으로 점프해서 이어서 보여줌.
4. 분기 종료 지점에서 더 진행해도 다른 분기(`path_b`)로 새지 않고 올바르게 종료됨 — 아래 "실제로 발견한 문제" 참고.

**실제로 발견한 문제 (예상 못 했던 것)**: 기존 `Advance()`는 한 시퀀스가 끝나면 "배열상 다음 시퀀스로" 그냥 넘어가는데, 분기로 도착한 시퀀스(`path_a`/`path_b`)에도 이 로직을 그대로 적용하면 두 분기가 잘못 이어붙는다(A 선택 후 끝까지 가면 B 대사가 이어서 나옴). 지금 프로토타입은 `sequenceId != "intro"`면 무조건 종료하는 임시 방편으로 막아놨는데, 시퀀스가 여러 개면 이 방식은 안 맞음 — 실제 통합 때 "분기 도착지는 명시적으로 종료 지점을 표시해야 한다"는 설계가 필요함.

**프로토타입이라 단순화한 것** (실제 통합 때 다시 채워야 함):
- JSON/`Resources.Load` 안 씀 — 대화 데이터를 코드에 하드코딩.
- `characterName`/`eventName` 매칭 없이 `id`로만 줄을 찾음 (실제 `dialogue_manager.GetLine`은 캐릭터+이벤트+id로 찾음).
- 세이브 파일에 실제로 안 씀 — flags가 메모리 static Dictionary에만 있음.

## Todo — 실제 통합 시 해야 할 일

- [ ] **어디에 연결할지 결정** — 이번 프로토타입은 의도적으로 완전히 격리됨. 스토리 내장형(대화 데이터에 선택지)과 이벤트 트리거형(팝업만 직접 호출) 중 뭘 먼저 실제로 붙일지 결정 필요.
- [ ] `dialogue_line`/`dialogue_sequence`(실제 클래스)에 `choices`/`sequenceId` 필드 실제로 추가 — 지금은 `_proto` 클래스로 분리되어 있어서 실제 클래스는 그대로임.
- [ ] `dialogue_scene_controller_branch.cs`를 실제 `dialogue_scene_controller_base`의 partial 파일로 옮기기(`_skip.cs`/`_autoplay.cs`와 같은 패턴). 지금은 완전히 별도 MonoBehaviour.
- [ ] `Update()`/`UpdateAutoPlay()`/`UpdateFastForwardAdvance()`/`SkipChapter()`(전부 실제 파일)에 `isChoicePending` 가드를 실제로 추가 — 프로토타입에서는 이 네 가지를 흉내낸 자체 로직으로만 검증함.
- [ ] `game_save_data`/`game_save_manager`(실제 파일)에 `flags` 저장소를 실제로 추가하고 세이브 파일 직렬화 대상에 포함. [[project-cluebook-todo]]/[[project-inventory-todo]]/[[project-worldmap-marker-todo]]와 같이 쓰는 것 전제로.
- [ ] `branch_popup_controller`를 실제 씬(`UI_Toggles` 등)에 배치하고, 다른 오버레이(`ClueBookPanel` 등)와 동시에 열렸을 때 상호작용 재검증.
- [ ] **분기 종료 지점 표시 방법 재설계** — 위 "실제로 발견한 문제" 참고. `sequenceId=="intro"` 같은 하드코딩이 아니라, 시퀀스마다 "다음이 뭔지"를 명시하는 방식(예: 명시적 `nextSequenceId` 필드, 또는 없으면 종료)으로 가야 함.
- [ ] 코드 기반(현재 프로토타입: C#에 하드코딩된 2개 선택지) vs 데이터 기반(JSON에서 선택지 정의) — 분기가 많아지면 후자로 옮길지 결정.
- [ ] `characterName`/`eventName` 기반 `GetLine` 조회 방식과 합치기 — 지금은 `id`만으로 조회하는 단순화된 상태.
