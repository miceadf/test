# 단축키 정리 (코드 기준, 2026-08-06)

프로젝트 전체 `Assets/Features/Scripts`를 `Keyboard.current` 사용처 기준으로 grep해서 정리한 것. 버튼 클릭으로만 되는 기능(단축키 없음)도 빠짐없이 표시함.

## 전역

| 키 | 동작 | 코드 |
|---|---|---|
| **ESC** | 열려있는 오버레이가 있으면 가장 최근에 연 것부터 닫음(LIFO). 아무것도 안 열려있으면 대신 **옵션(일시정지) 패널을 엶** | `ui_escape_manager.cs` |

ESC는 한 번 누르면 정확히 하나의 동작만 하도록 `ui_escape_manager`가 입력을 전담해서 처리함(`ui_overlay_state.TryCloseTopOverlay()` 먼저 시도 → 실패하면 옵션 패널 오픈).

## 대화 화면 (`dialogue_scene_controller_base` 및 partial 파일들)

오버레이 패널이 하나라도 열려있으면(`ui_overlay_state.IsAnyOverlayOpen`) 아래 전부 비활성화됨.

| 입력 | 동작 | 비고 |
|---|---|---|
| **Space** / **마우스 좌클릭** | 타이핑 중이면 즉시 완성, 아니면 다음 대사로 진행 | Auto가 켜져 있으면 먼저 Auto를 끄기만 함(중복 진행 방지). 클릭 가능한 UI(Selectable) 위에서는 무시됨 |
| **A** | Auto(자동 진행) 켜기/끄기 | 대사가 끝나면 지연시간(기본 1.2초, `setting_data.autoPlayDelaySeconds`로 옵션에서 조절 가능) 후 자동으로 다음 줄 |
| **S** | 챕터 스킵 요청 | `RequestSkipChapter()` — 되돌릴 수 없는 동작이라 `confirm_popup_controller` 확인 팝업이 먼저 뜨고, 확인해야 실제 스킵(`SkipChapter()`) 실행 |
| **왼쪽 Ctrl (누르고 있는 동안)** | 배속 | 타이핑 속도가 빨라지고(`dialogue_manager_typing.GetEffectiveCharacterInterval()`), 타이핑이 끝난 다음 줄로도 0.05초 간격으로 자동 진행(`dialogue_scene_controller_fastforward.cs`) |

## 오버레이 패널 토글 (`UI_Toggles`의 각 컨트롤러 — 키보드 + 아이콘바 버튼 둘 다 가능)

| 키 | 패널 | 컨트롤러 |
|---|---|---|
| **Y** | 단서함 (ClueBook) | `clue_book_toggle_controller.cs` |
| **I** | 인벤토리 (Inventory) | `inventory_toggle_controller.cs` |
| **M** | 월드맵 (WorldMap) | `world_map_toggle_controller.cs` |
| **L** | 대사 로그 (Backlog) | `dialogue_backlog_toggle_controller.cs` |
| *(없음, 버튼 전용)* | 세이브 슬롯 패널 | `save_slot_toggle_controller.cs` |
| *(없음, 버튼 전용 — ESC로는 열림)* | 옵션 | `options_toggle_controller.cs` |

전부 `ui_overlay_state.CanOpen()`을 통과해야 열림 — 즉 **다른 오버레이가 이미 열려있으면 이 키들도 눌러도 안 열림**(예: Options가 열려있을 때 Y를 눌러도 ClueBook은 안 열림).

## 확인 팝업 (`confirm_popup_controller`)

키보드 단축키 없음 — 마우스로 예/아니오 버튼만 클릭 가능. 단 ESC는 `ui_overlay_state`에 등록돼 있어서 "아니오"와 동일하게 취소 처리됨.

## 테스트/개발용 — 배포 대상 아님

| 키 | 동작 | 비고 |
|---|---|---|
| **1** / **Numpad 1** | `instance_guide_controller`의 안내 패널 토글 | `[Header("Test")] enableKeyboardTest` 플래그로 켜져 있음. Inspector에서 끌 수 있고, 정식 노출은 `OnToggle()`/`OnToggle(string id)`를 버튼 등에서 호출하는 방식 |

## 미니게임 전용

| 키 | 동작 | 씬/스크립트 |
|---|---|---|
| **R** | 사다리 재생성 | `MiniGameTest_scene`, `ladder_game_controller.cs` (에디터 `[ContextMenu]`로도 가능) |
| **Z** | 왼쪽 플리퍼 | PinBall, `pinball_flipper_controller.cs` |
| **/ (Slash)** | 오른쪽 플리퍼 | PinBall, `pinball_flipper_controller.cs` |
| **Space (누르고 있는 동안 충전 → 떼면 발사)** | 발사 핀 충전/발사 | PinBall, `pinball_shooting_pin_controller.cs` |

PinBall 미니게임도 Space를 쓰지만 대화 화면의 Space(다음 대사 진행)와는 완전히 별개 씬/컨트롤러라 충돌 없음.
