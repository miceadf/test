# 분기 시스템 프로토타입 (Prototype_Branch) — Todo

실제 통합 지점이 아직 정해지지 않아서, 실제 프로젝트 파일은 하나도 건드리지 않고
`Assets/Prototype_Branch/`에 완전히 격리해서 구현/검증함. [branching_system.md](./branching_system.md) 설계 검토를 코드로 옮긴 것.

**2026-08-06 개정**: 초기 버전은 대사/시퀀스를 C# 리스트에 하드코딩해서 실제 진행 방식과
너무 달랐음 — 데이터 로딩 경로까지 실제 시스템(Resources JSON + Newtonsoft)과 동일하게
다시 만들고, 테스트 대사도 실제 `Presia_C1Main.json`에서 그대로 가져왔고, `ChangeScene`/세이브를
로그만 남기던 시뮬레이션에서 실제 동작(진짜 씬 이동, 진짜 파일 저장/불러오기)으로 바꿈.

## 현재 어떻게 작동하는가 (구현 완료, Play 모드에서 검증됨)

**위치** (전부 `Assets/Prototype_Branch/`, 실제 `Assets/Features/`와 같은 하위 경로):
- `Features/Scripts/Dialogue/dialogue_line.cs` — `branch_action_type` enum + `dialogue_choice_proto`/`dialogue_line_proto`/`dialogue_sequence_proto`/`dialogue_flow_data_proto`/`dialogue_data_proto`. 실제 `dialogue_line`/`dialogue_sequence`/`dialogue_data`(`Assets/Features/Scripts/Dialogue/dialogue_line.cs`)를 복제하고 분기용 필드(`choices`, `sequenceId`, `actionType` 등)만 추가한 것. 이름에 `_proto`가 붙은 이유는 실제 클래스와 이름이 겹치면 컴파일이 깨지기 때문 — **실제 게임 코드와는 전혀 연결되어 있지 않음.**
- `Features/Scripts/Save/game_save_manager.cs` — `game_save_flags_prototype`(static, 메모리 전용 flags) + `branch_test_save_manager_prototype`(실제 파일 I/O). 후자는 실제 `game_save_manager.SaveToPath/TryLoadFromPath`와 동일한 방식(`JsonUtility` + `persistentDataPath`)이지만, 실제 세이브 파일과 안 섞이게 `persistentDataPath/branch_test_save/branch_test_save.json` 하나에만 쓴다.
- `Features/Scripts/UI/branch_popup_controller.cs` — `confirm_popup_controller`를 N개 선택지로 일반화한 팝업. 옵션 버�튼은 `instance_guide_controller.BuildCurrent()`처럼 프리팹 없이 코드로 동적 생성. `cancelable` 플래그로 ESC 취소 가능/불가능을 나눔. (이번 개정에서 변경 없음.)
- `Features/Scripts/SceneController/dialogue_scene_controller_branch.cs` — `dialogue_scene_controller_branch_prototype`(독립 MonoBehaviour). 실제 `dialogue_scene_controller_base.LoadFlow/ShowNext/Advance`와 같은 구조로, `Resources.Load<TextAsset>` + `JsonConvert`로 flow/대사 JSON을 읽어온다. 실제로는 `dialogue_scene_controller_base`의 partial 파일이 될 자리지만, 통합 방식이 안 정해진 지금은 완전히 별도 클래스로 분리해서 실제 대화 시스템에 어떤 영향도 주지 않게 했다.
- `Resources/Dialogues_Proto/Flows/BranchTest_Flow.json` — 실제 `Dialogues/Flows/*.json`과 같은 형식(`flowName`+`sequences[{sequenceId,characterName,eventName,startId,endId}]`). `sequenceId`는 실제 스키마엔 없는 필드지만, 분기 타겟이 배열 인덱스가 아니라 안정적인 라벨을 가리키게 하려고 추가함.
- `Resources/Dialogues_Proto/BranchTest_Presia.json` — 실제 `Dialogues/Presia/Presia_C1Main.json`의 id 1~4, 8~9 대사를 그대로 복사(자연스러운 흐름일 필요는 없다고 확인받음). id=4 줄에 `choices` 필드로 분기 두 개를 데이터로 직접 선언.
- 테스트용 씬: `Scenes/BranchPrototype_scene.unity` (Canvas/EventSystem + LineText/StatusText/BranchPopupPanel).

**선택지가 실제로 뭘 하는지는 `branch_action_type` enum으로 명시적으로 구분한다** (`dialogue_choice_proto.actionType`, JSON에는 문자열로 저장) — null인 target 필드로 의도를 추론하는 대신, `JumpToLine`/`ChangeScene`/`FireEvent` 중 하나를 선언하고 `ResolveChoice()`가 그 값으로 switch한다. 새 분기 방식이 필요해지면 enum에 값 하나 추가 + switch에 케이스 하나만 추가하면 됨:
- `JumpToLine` — 같은 흐름 안의 다른 지점(`targetSequenceId`+`targetLineId`)으로 이동. 테스트 데이터: id=4에서 "더 알아본다" 선택 → `path_a` 시퀀스의 id=8로 점프.
- `ChangeScene` — 다른 씬으로 이동(`targetSceneName`). **실제로 `scene_flow_manager.EnsureExists().GoToScene(sceneName)`을 호출한다** (로그만 남기던 이전 버전과 다름). 테스트 데이터: id=4에서 "그냥 마을로 돌아간다" 선택 → 실제로 `Feature_scene`이 로드됨(Play 모드에서 `manage_scene get_active`로 확인).
- `FireEvent` — 다른 시스템에 알리기만 하고(`OnBranchEventFired` static 이벤트) 대화는 계속 진행. 지금 테스트 데이터는 안 쓰지만 switch 케이스는 계속 살려둠(확장성 스켈레톤 유지).

**Play 모드에서 실제로 확인된 것** (이번 개정, 전부 리플렉션으로 재검증):
1. Awake에서 `Dialogues_Proto/Flows/BranchTest_Flow.json` 로드 → id 1~3 순차 진행(각 줄 텍스트가 실제 Presia_C1Main.json 내용과 일치) → id=4(분기 줄)에서 자동으로 멈추고 팝업 표시, 버튼 라벨이 JSON의 `choices[].text`와 일치.
2. "더 알아본다" 클릭 → `flags["branch_test_choice"]="investigate"` 저장 → `path_a`(seq=1)의 id=8로 점프해서 이어서 보여줌 → id=9에서 `sequenceId != "intro"`라 정상 종료(다른 시퀀스로 새지 않음).
3. F5로 저장 → 실제로 `%AppData%/LocalLow/DefaultCompany/SSAL_FIRST/branch_test_save/branch_test_save.json`에 `{flowResourcePath, sequenceIndex, currentLineId, flags}` 기록됨(디스크에서 직접 확인).
4. 진행 상태를 다른 지점으로 바꾼 뒤 F9로 불러오기 → 저장했던 지점(seq/line)으로 정확히 복원되고 그 줄부터 이어서 보여줌 — `dialogue_progress_resolver.TryResolve`와 동일한 유효성 검증(시퀀스 범위 + startId~endId) 통과.
5. "그냥 마을로 돌아간다" 클릭 → `scene_flow_manager.GoToScene("Feature_scene")`이 실제로 호출되어 `Feature_scene`이 로드됨(액티브 씬이 실제로 바뀐 것을 확인).
6. R키로 언제든 리셋 가능(변경 없음, 이번 개정에서 재검증 안 함 — 로직 자체가 안 바뀜).
7. **분기 종료 지점을 `isBranchEndpoint` 필드로 명시** (2026-08-06 추가) — 예전엔 `sequenceId != "intro"`라는 이름 기반 임시방편이었음. `dialogue_sequence_proto.isBranchEndpoint`(기본값 false)를 추가하고 `Advance()`가 이 필드로 판단하도록 교체. 검증을 위해 flow에 `path_a` 바로 뒤에 `unreachable_tail`(id=10, 대사: "(오류) 이 대사가 보이면 분기 종료 처리가 깨진 것입니다")이라는 3번째 시퀀스를 추가 — `path_a.isBranchEndpoint=true`로 끝났을 때 배열상 다음 칸인 이 시퀀스로 새지 않고 `ended=true`로 정상 종료되는 것을 실제로 확인함(리플렉션으로 id=10 텍스트가 절대 안 나오는 것까지 검증).

**프로토타입이라 여전히 단순화한 것**:
- `dialogue_manager.GetLine`의 캐릭터-폴더 후보 탐색(`dialogue_path_resolver`)은 재현하지 않음 — 고정 경로 `Dialogues_Proto/{eventName}`로 단순화(캐릭터 폴더 매핑은 이 프로토타입이 검증하려는 부분이 아님).
- 세이브는 1개 파일만 씀(실제 시스템의 3슬롯+오토세이브 구조는 안 따라함) — 분기 위치가 저장/복원되는지만 검증하는 게 목적.
- flags는 세이브 파일에 포함되긴 하지만, 실제 `game_save_data`에 통합된 게 아니라 이 프로토타입 전용 별도 파일에만 있음.

## Todo — 실제 통합 시 해야 할 일

- [ ] **어디에 연결할지 결정** — 이번 프로토타입은 의도적으로 완전히 격리됨. 스토리 내장형(대화 데이터에 선택지)과 이벤트 트리거형(팝업만 직접 호출) 중 뭘 먼저 실제로 붙일지 결정 필요.
- [ ] `dialogue_line`/`dialogue_sequence`(실제 클래스)에 `choices`/`sequenceId` 필드 실제로 추가 — 지금은 `_proto` 클래스로 분리되어 있어서 실제 클래스는 그대로임.
- [ ] `dialogue_scene_controller_branch.cs`를 실제 `dialogue_scene_controller_base`의 partial 파일로 옮기기(`_skip.cs`/`_autoplay.cs`와 같은 패턴). 지금은 완전히 별도 MonoBehaviour.
- [ ] `Update()`/`UpdateAutoPlay()`/`UpdateFastForwardAdvance()`/`SkipChapter()`(전부 실제 파일)에 `isChoicePending` 가드를 실제로 추가 — 프로토타입에서는 이 네 가지를 흉내낸 자체 로직으로만 검증함.
- [ ] `game_save_data`/`game_save_manager`(실제 파일)에 `flags` 저장소를 실제로 추가하고 세이브 파일 직렬화 대상에 포함. [[project-cluebook-todo]]/[[project-inventory-todo]]/[[project-worldmap-marker-todo]]와 같이 쓰는 것 전제로. (지금 `branch_test_save_manager_prototype`은 이 통합 전까지만 쓰는 임시 경로.)
- [ ] `branch_popup_controller`를 실제 씬(`UI_Toggles` 등)에 배치하고, 다른 오버레이(`ClueBookPanel` 등)와 동시에 열렸을 때 상호작용 재검증.
- [x] **분기 종료 지점 표시 방법 재설계** — `isBranchEndpoint` 불리언 필드로 구현 완료(위 "실제로 확인된 것" 7번 참고). 실제 `dialogue_sequence`에도 이 필드만 추가하면 됨 — 기본값 false라 기존 Flow JSON(Scene_1~6)은 하나도 안 고쳐도 그대로 동작함.
- [ ] `dialogue_manager.GetLine`의 캐릭터-폴더 후보 탐색(`dialogue_path_resolver`)과 합치기 — 지금은 고정 경로로 단순화된 상태.
- [ ] `OnBranchEventFired`를 실제로 구독하는 시스템 연결(아이템 지급/퀘스트 갱신 등) — 지금은 이벤트가 발행되긴 하지만 아무도 구독하지 않음.
- [ ] 실제 통합 시 세이브 슬롯 구조(3슬롯+오토세이브)에 분기 위치가 어떻게 들어갈지 결정 — 지금 프로토타입은 슬롯 개념 없이 파일 하나만 씀.
