# Script Code Preview

> **역할**: 스크립트별 **역할·Public API·코드 발췌·Inspector** 정리.  
> **다이어그램**은 [ScriptDiagrams.md](./ScriptDiagrams.md)를 본다.

---

## 목차

| 섹션 | 스크립트 |
|------|----------|
| [1. Settings](#1-settings) | SettingSaveManager, SettingData, MenuSettingWindow |
| [2. Dialogue](#2-dialogue) | DialogueManager, PathResolver, Flow Controller |
| [3. Sound](#3-sound) | SFXManager, MusicDirector, MusicProfile, MusicNode |
| [4. Save / Progress](#4-save--progress) | MonoSingleton, GameSaveManager, GameSaveData |
| [5. Clue](#5-clue) | ClueInventoryManager, ClueData |
| [6. Scene Flow / UI](#6-scene-flow--ui) | SceneFlowManager, WorldMapToggleController |
| [7. Editor](#7-editor-참고) | 에디터 전용 도구 |

---
## 1. Settings

#### `SettingSaveManager` (`SettingSaveManager.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Settings/SettingSaveManager.cs` |
| **역할** | `settings.json` 저장·로드 |
| **붙는 곳** | Start_scene (또는 `EnsureExists()`로 런타임 생성) |
| **주요 의존** | `MonoSingleton<SettingSaveManager>`, `SettingData`, `Application.persistentDataPath` |

**Public API**

| 멤버 | 설명 |
|------|------|
| `Instance` | `MonoSingleton<T>`가 제공하는 싱글톤 참조 |
| `EnsureExists()` | `MonoSingleton<T>`가 제공 — 없으면 찾거나 새 GameObject 생성 |
| `SaveSetting()` | `settingData` → JSON 파일 |
| `LoadSetting()` | 파일 → `settingData` (없/손상 시 초기화) |
| `settingData` | 현재 설정 인스턴스 |

**코드 프리뷰**

```csharp
public class SettingSaveManager : MonoSingleton<SettingSaveManager>
{
    protected override void OnHostInstanceEstablished()
    {
        LoadSetting();
    }
}

// SaveFilePath = Path.Combine(Application.persistentDataPath, "settings.json")
```

**호출 흐름**: `MenuSettingWindow` → `EnsureExists()`(내부에서 `OnHostInstanceEstablished()` → `LoadSetting()`) → `settingData` 사용

**주의**: Windows 에디터 경로 예 — `AppData/LocalLow/DefaultCompany/SSAL_FIRST/settings.json`

---

#### `SettingData` (`SettingData.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Settings/SettingData.cs` |
| **역할** | 설정 직렬화 DTO |
| **붙는 곳** | 없음 (데이터 클래스) |

**필드**

| 필드 | 기본값 | 설명 |
|------|--------|------|
| `masterVolume`, `bgmVolume`, `sfxVolume` | 1f | 0~1 |
| `masterMuted`, `bgmMuted`, `sfxMuted` | false | 뮤트 |
| `resolutionIndex` | 0 | 해상도 드롭다운 인덱스 |

**코드 프리뷰**

```csharp
[Serializable]
public class SettingData
{
    public float masterVolume = 1f;
    public float bgmVolume = 1f;
    public float sfxVolume = 1f;
    public bool masterMuted = false;
    public bool bgmMuted = false;
    public bool sfxMuted = false;
    public int resolutionIndex = 0;
}
```

---

#### `MenuSettingWindow` (`MenuSettingScript.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Settings/MenuSettingScript.cs` |
| **역할** | 옵션 UI — 볼륨·뮤트·해상도, Mixer 반영, 저장 |
| **붙는 곳** | Start_scene — Option / Settings Panel |
| **주요 의존** | `SettingSaveManager`, `AudioMixer`, TMP, Slider |

**Public API**

| 멤버 | 설명 |
|------|------|
| `ToggleMasterMute()` / `ToggleBGMMute()` / `ToggleSFXMute()` | 뮤트 토글 + 슬라이더 UI 동기화 |
| `SetResolution(int index)` | `Screen.SetResolution` + `resolutionIndex` 저장 |
| `SaveSettingData()` | `SettingSaveManager.SaveSetting()` 후 패널 비활성 |
| `CloseMenu()` | 패널만 비활성 |

**코드 프리뷰**

```csharp
void TrySetMixerFloat(string parameterName, float dB)
{
    if (mixer == null || string.IsNullOrEmpty(parameterName)) return;
    mixer.SetFloat(parameterName, dB);
}

void ResolveMixerReference()
{
    if (mixer != null) return;
    mixer = Resources.Load<AudioMixer>("GameSettingsMixer");
    // 실제 프로젝트: Assets/Features/GameSettingsMixer.mixer → Inspector 할당 권장
}
```

**Inspector / 연결**

| 필드 | 타입 | 필수 |
|------|------|------|
| `mixer` | AudioMixer | Y (또는 Resources fallback) |
| `masterVolumeSlider` 등 | Slider ×3 | Y |
| `masterText` 등 | TMP_Text ×3 | Y |
| `masterImage` 등 + `soundWaves` | Image + Sprite List | Y |
| `resolutionDropdown` | TMP_Dropdown | Y |

**호출 흐름**: 슬라이더 드래그 → `PreviewVolume` → Mixer dB / 드래그 종료 → `SetVolume` → `SettingData` + `ApplyAllVolumes`

---

## 2. Dialogue

#### `DialogueManager` (`DialogueManager.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Json/DialogueManager.cs` |
| **역할** | 대화 JSON 로드·캐싱, 타이핑 출력, NextArrow, UI 토글 |
| **붙는 곳** | scene_1 — DialogueManager Object |
| **주요 의존** | Newtonsoft.Json, `DialoguePathResolver`, TMP, Input System |

**Public API**

| 멤버 | 설명 |
|------|------|
| `Instance` | 씬 내 싱글톤(독자 구현, `DontDestroyOnLoad` 아님) |
| `GetLine(character, event, id)` | `DialogueLine` 반환 |
| `GetLines(character, event)` | 전체 lines |
| `ShowLine(line)` | 타이핑 시작 |
| `CompleteTyping()` | 타이핑 즉시 완료 |
| `IsTyping()` / `CanProceed()` | Space 진행 조건 |
| `ToggleUI(bool)` | 대화 UI on/off (우클릭도 동일) |

> 예전에 있던 `LoadDialogue(...)` 레거시 호환 메소드는 아무 데서도 참조되지 않아 삭제됨(`GetLine` + `ShowLine`으로 대체).

**코드 프리뷰**

```csharp
public DialogueLine GetLine(string characterName, string eventName, int id)
{
    DialogueData data = GetDialogueData(characterName, eventName);
    if (data == null) return null;
    return data.lines.Find(item => item.id == id);
}

private TextAsset LoadDialogueTextAsset(string characterName, string eventName, out string resourcePath)
{
    foreach (string candidateEventName in DialoguePathResolver.GetEventNameCandidates(eventName))
        foreach (string folderName in DialoguePathResolver.GetFolderCandidates(characterName, candidateEventName))
        {
            string candidatePath = $"Dialogues/{folderName}/{candidateEventName}";
            TextAsset jsonText = Resources.Load<TextAsset>(candidatePath);
            if (jsonText != null) { resourcePath = candidatePath; return jsonText; }
        }
    return null;
}
```

**Inspector**

| 필드 | 필수 |
|------|------|
| `dialogueText` | Y |
| `dialogueUI` | Y |
| `nextArrow` | N (자식 `NextArrow` 자동 탐색) |

**호출 흐름**: `Scene Controller` → `GetLine` → `ShowLine` → `TypeLine` → Space → `CompleteTyping` 또는 다음 line

---

#### `DialoguePathResolver` (`DialoguePathResolver.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Json/DialoguePathResolver.cs` |
| **역할** | 한글 캐릭터명·이벤트명 → Resources 경로 후보 생성 |
| **붙는 곳** | 없음 (static) |

**Public API**

| 멤버 | 설명 |
|------|------|
| `GetFolderCandidates(characterName, eventName)` | 폴더 후보 리스트 (순서대로 시도) |
| `GetEventNameCandidates(eventName)` | 이벤트명 + `Patridge_` 보정 |

**코드 프리뷰**

```csharp
private static readonly Dictionary<string, string> characterFolderMap = new()
{
    { "모피어스", "Morpheus" },
    { "프레시아", "Presia" },
    { "상인A", "etc" },
    // ...
};

public static List<string> GetFolderCandidates(string characterName, string eventName)
{
    var folders = new List<string>();
    if (characterFolderMap.TryGetValue(characterName, out string mappedFolder))
        AddCandidate(folders, mappedFolder);
    AddCandidate(folders, characterName);
    AddCandidate(folders, GetEventPrefix(eventName));
    AddCandidate(folders, "etc");
    return folders;
}
```

---

#### 대화 데이터 타입 (`DialogueLine.cs`)

| 클래스 | 용도 | 주요 필드 |
|--------|------|-----------|
| `DialogueLine` | 한 줄 대사 | `id`, `characterName`, `dialogue_KR` |
| `DialogueData` | 캐릭터/이벤트 JSON | `chapterName`, `lines` |
| `DialogueSequence` | Flow 한 구간 | `characterName`, `eventName`, `startId`, `endId` |
| `DialogueFlowData` | 씬 Flow JSON | `flowName`, `sequences` |

> 선택지(분기) 관련 필드는 없음 — 진행은 항상 `sequences` 배열 순서대로만 이루어진다.

**코드 프리뷰**

```csharp
public class DialogueSequence
{
    public string characterName;
    public string eventName;
    public int startId = 1;
    public int endId = 1;
}

public class DialogueFlowData
{
    public string flowName;
    public List<DialogueSequence> sequences;
}
```

**리소스 예**: `Resources/Dialogues/Flows/Scene_1_Flow.json`, `Resources/Dialogues/Presia/Presia_C1Main.json`

---

#### `DialogueSceneControllerBase` (`DialogueSceneControllerBase.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/SceneController/DialogueSceneControllerBase.cs` |
| **역할** | Flow JSON 로드 후 sequence별 대화 자동 진행 + 진행상황 저장 + 종료 시 씬 전환 |
| **붙는 곳** | scene_1 등 — 씬별 Controller |

**확장 포인트**

| 멤버 | 설명 |
|------|------|
| `DefaultFlowResourcePath` | abstract — Resources 경로 |
| `ChapterName` | abstract — 이 씬이 속한 챕터 이름. `OnDialogueFlowFinished()`에서 `GameSaveManager.SetChapter()`에 씀 |
| `NextSceneName` | virtual, 기본 `null` — 값이 있으면 Flow 종료 시 그 씬으로 자동 전환 |
| `flowResourcePath` | Inspector override (비어 있으면 Default 사용) |

**코드 프리뷰**

```csharp
protected virtual void OnDialogueFlowFinished()
{
    DialogueManager.Instance.ToggleUI(false);

    GameSaveManager saveManager = GameSaveManager.EnsureExists();
    saveManager.SetChapter(ChapterName);
    saveManager.AutoSave();

    if (!string.IsNullOrWhiteSpace(NextSceneName))
        SceneFlowManager.EnsureExists().GoToScene(NextSceneName);
}

private void ShowNext()
{
    DialogueSequence sequence = flowData.sequences[sequenceIndex];
    DialogueLine line = DialogueManager.Instance.GetLine(sequence.characterName, sequence.eventName, currentLineId);
    if (line == null) return;

    DialogueManager.Instance.ShowLine(line);
    GameSaveManager.EnsureExists().SetDialogueProgress(FlowResourcePath, sequenceIndex, currentLineId);
    Advance();
}
```

**입력**: Space — 타이핑 스킵 / 다음 line

**진행상황 저장 방식**: `sequenceIndex`/`currentLineId`는 1부터 증가하는 전역 카운터가 아니라 "현재 활성 Flow(`flowResourcePath`) 안에서 몇 번째 시퀀스·몇 번 id인지"를 가리키는 좌표다. 매 줄 보여줄 때마다 메모리상의 `GameSaveData`만 갱신되고, 디스크 저장은 Flow가 끝날 때(`AutoSave()`)만 일어난다.

---

#### `Scene_1_Controller` (`Scene_1_Controller.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/SceneController/Scene_1_Controller.cs` |
| **역할** | 1장 Flow 전용 Controller |
| **붙는 곳** | scene_1 — Scene_1_Controller Object |

**코드 프리뷰**

```csharp
public class Scene_1_Controller : DialogueSceneControllerBase
{
    protected override string DefaultFlowResourcePath => "Dialogues/Flows/Scene_1_Flow";
    protected override string ChapterName => "Chapter01";
    protected override string LogPrefix => "[Scene_1]";
}
```

**새 씬 컨트롤러 추가 패턴**: `DialogueSceneControllerBase`를 상속해서 `DefaultFlowResourcePath`, `ChapterName`, `LogPrefix` 세 개를 채우고, 이 씬 끝나고 바로 다음 씬으로 넘어가야 하면 `NextSceneName`도 override한다. 그 외 로직은 부모 클래스가 전부 처리하므로 건드릴 필요 없음.

---

## 3. Sound

#### `SFXManager` (`SFXManager.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Sound/SFXManager.cs` |
| **역할** | 버튼 클릭 SFX, 씬 전환 후에도 유지 |
| **붙는 곳** | Start_scene — SFXManager Object |
| **주요 의존** | `MonoSingleton<SFXManager>` |

**Public API**

| 멤버 | 설명 |
|------|------|
| `Instance` / `EnsureExists()` | `MonoSingleton<T>` 제공 |
| `PlayButtonClick()` | `PlayOneShot(buttonClickClip)` |
| `BindButtonsInScene()` | 씬 내 모든 Button에 리스너 등록 |

**코드 프리뷰**

```csharp
public sealed class SFXManager : MonoSingleton<SFXManager>
{
    protected override void OnHostInstanceEstablished()
    {
        if (audioSource == null)
            audioSource = GetComponent<AudioSource>();
        audioSource.playOnAwake = false;
        audioSource.loop = false;
        audioSource.spatialBlend = 0f;
        if (outputMixerGroup != null)
            audioSource.outputAudioMixerGroup = outputMixerGroup;
    }

    public void BindButtonsInScene()
    {
        if (!bindButtonsAutomatically) return;
        foreach (Button button in FindObjectsByType<Button>(FindObjectsInactive.Include, FindObjectsSortMode.None))
        {
            if (button == null || boundButtons.Contains(button)) continue;
            button.onClick.AddListener(PlayButtonClick);
            boundButtons.Add(button);
        }
    }
}
```

**Inspector**: `audioSource`, `buttonClickClip` (`Resources/Sound/Sfx/button_click.mp3`)

---

#### `MusicDirector` / `MusicProfile` / `MusicNode` (`MusicDirector.cs`, `MusicProfile.cs`, `MusicNode.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Sound/` |
| **네임스페이스** | `SSAL.Sound` |
| **역할** | 배경음 재생 — `MusicProfile`(ScriptableObject)에 저장된 `MusicNode` 트리를 재귀 실행 |
| **붙는 곳** | Start_scene — MusicDirector Object (씬 로컬, 싱글톤 아님) |

**`MusicNode` 종류**

| 클래스 | 동작 |
|--------|------|
| `MusicClipNode` | AudioClip 하나 재생 (volume, pitch) |
| `MusicSequenceNode` | 자식들을 순서대로 재생 |
| `MusicLoopNode` | 자식을 지정 횟수(0=무한) 반복 |
| `MusicRandomNode` | 자식 중 무작위 선택 (`avoidImmediateRepeat` 옵션) |
| `MusicOverlayNode` | 여러 자식을 동시에 재생 |
| `MusicDelayNode` | 다음 노드 전 랜덤 범위만큼 대기 |

**코드 프리뷰**

```csharp
[CreateAssetMenu(fileName = "MusicProfile", menuName = "SSAL/Sound/Music Profile")]
public sealed class MusicProfile : ScriptableObject
{
    [SerializeReference] private MusicNode root;
    public MusicNode Root => root;
}

public void Play(MusicProfile profile)
{
    Stop();
    if (profile == null || profile.Root == null) return;

    double startTime = AudioSettings.dspTime + schedulingLeadTime;
    StartCoroutine(Execute(profile.Root, new PlaybackCursor(startTime), playbackVersion));
}
```

**Inspector**: `playOnStart`(MusicProfile), `outputMixerGroup`, `schedulingLeadTime`, `logPlayback`

**주의**: `MusicDirector`는 `DontDestroyOnLoad`가 없다. Start_scene에서 재생 중이던 배경음은 scene_1로 넘어가면 오브젝트째로 파괴되어 끊긴다 — 씬 간 배경음 연속 재생은 별도 작업 필요(현재 Script 브랜치 범위 밖, 다른 브랜치에서 진행 중).

---

## 4. Save / Progress

#### `MonoSingleton<T>` (`Core/MonoSingleton.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Core/MonoSingleton.cs` |
| **역할** | 씬에 하나만 있어야 하는 매니저들의 공통 싱글톤 베이스 — 중복 오브젝트 파괴, `DontDestroyOnLoad`, `EnsureExists()` 로직을 한 곳에 모음 |
| **붙는 곳** | 없음 (제네릭 추상 클래스, 상속 전용) |

**Public API**

| 멤버 | 설명 |
|------|------|
| `Instance` | 정적 싱글톤 참조 |
| `EnsureExists()` | 없으면 씬에서 찾거나 새 GameObject 생성 후 반환 |
| `OnHostInstanceEstablished()` | protected virtual — 이 인스턴스가 싱글톤 자리를 차지했을 때 한 번 호출. 하위 클래스는 초기 로드 로직을 여기 둠 |

**코드 프리뷰**

```csharp
public abstract class MonoSingleton<T> : MonoBehaviour where T : MonoSingleton<T>
{
    public static T Instance { get; private set; }

    public static T EnsureExists()
    {
        if (Instance != null) return Instance;
        var existing = FindFirstObjectByType<T>(FindObjectsInactive.Include);
        if (existing != null) { existing.TryBecomeHostInstance(); return Instance; }
        var go = new GameObject(typeof(T).Name);
        go.AddComponent<T>();
        return Instance;
    }

    protected virtual void Awake() => TryBecomeHostInstance();

    private void TryBecomeHostInstance()
    {
        if (Instance == this) return;
        if (Instance != null && Instance != this) { Destroy(gameObject); return; }
        Instance = (T)this;
        DontDestroyOnLoad(gameObject);
        OnHostInstanceEstablished();
    }

    protected virtual void OnHostInstanceEstablished() { }
}
```

**상속 클래스**: `GameSaveManager`, `ClueInventoryManager`, `SettingSaveManager`, `SFXManager`, `SceneFlowManager`

---

#### `GameSaveManager` (`GameSaveManager.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Save/GameSaveManager.cs` |
| **역할** | 챕터·대사 진행상황·단서 목록을 슬롯별 JSON 파일로 저장/로드 |
| **붙는 곳** | 없음(`EnsureExists()`로 런타임 생성) |
| **주요 의존** | `MonoSingleton<GameSaveManager>`, `GameSaveData` |

**Public API**

| 멤버 | 설명 |
|------|------|
| `MinSaveSlot` / `MaxSaveSlot` | 1 ~ 3 |
| `AutoSaveSlot` | 오토세이브 전용 슬롯 번호(=`MinSaveSlot`=1) |
| `Data` | 현재 활성 슬롯의 `GameSaveData` |
| `SetChapter(name)` | `currentChapter` 갱신(메모리만) |
| `SetDialogueProgress(flow, seq, lineId)` | 대사 진행 좌표 갱신(메모리만) |
| `AutoSave()` | `AutoSaveSlot`에 저장 — 수동 저장과 진입점 분리 |
| `ResetForNewGame(chapterName)` | `GameSaveData`를 완전히 새로 만들고 지정 챕터로 `AutoSave()` — "새 게임" 전용 |
| `SaveGameToSlot(slot)` / `LoadGameFromSlot(slot)` | 수동 슬롯 저장/로드 (아직 UI 미연결) |
| `HasSaveSlot(slot)` | 슬롯 파일 존재 여부 |
| `PeekSlotData(slot)` | 활성 슬롯 안 바꾸고 다른 슬롯 내용만 읽기 (저장 메뉴 미리보기용, 아직 UI 미연결) |
| `HasClue(id)` / `AddClue(id)` | 단서 목록 조회/추가 |

**코드 프리뷰**

```csharp
public const int AutoSaveSlot = MinSaveSlot;

public void AutoSave()
{
    SaveGameToSlot(AutoSaveSlot);
}

public void ResetForNewGame(string chapterName)
{
    gameSaveData = new GameSaveData();
    EnsureGameSaveData();
    gameSaveData.currentChapter = chapterName;
    AutoSave();
}

public GameSaveData PeekSlotData(int slot)
{
    string path = GetSaveFilePath(slot);
    if (!File.Exists(path)) return null;
    string json = File.ReadAllText(path);
    return string.IsNullOrEmpty(json) ? null : JsonUtility.FromJson<GameSaveData>(json);
}
```

**저장 파일 경로**: `Application.persistentDataPath/game_save_slot_{slot}.json`

**실제 확인된 저장 내용 예** (scene_1 대사 22턴 끝까지 진행 후):

```json
{
    "currentChapter": "Chapter01",
    "dialogueProgress": {
        "flowResourcePath": "Dialogues/Flows/Scene_1_Flow",
        "sequenceIndex": 21,
        "currentLineId": 11
    },
    "clueInventory": { "obtainedClueIds": [] }
}
```

**미구현**: 오토세이브 슬롯(1번)을 수동 저장으로부터 코드 레벨에서 막는 로직 — 저장 메뉴 UI 만들 때 같이 추가 예정.

---

#### `GameSaveData` (`GameSaveData.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Save/GameSaveData.cs` |
| **역할** | 세이브 파일 직렬화 DTO |

**코드 프리뷰**

```csharp
[Serializable]
public class GameSaveData
{
    public string currentChapter = "Chapter01";
    public DialogueProgressSaveData dialogueProgress = new DialogueProgressSaveData();
    public ClueInventorySaveData clueInventory = new ClueInventorySaveData();
}

[Serializable]
public class DialogueProgressSaveData
{
    public string flowResourcePath = "Dialogues/Flows/Scene_1_Flow";
    public int sequenceIndex = 0;
    public int currentLineId = 1;
}

[Serializable]
public class ClueInventorySaveData
{
    public List<string> obtainedClueIds = new List<string>();
}
```

---

## 5. Clue

#### `ClueInventoryManager` (`ClueInventoryManager.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Clue/ClueInventoryManager.cs` |
| **역할** | 단서 획득/조회 — 실제 저장은 `GameSaveManager`에 위임 |
| **붙는 곳** | 없음(`EnsureExists()`로 런타임 생성) |
| **주요 의존** | `MonoSingleton<ClueInventoryManager>`, `GameSaveManager` |

**Public API**

| 멤버 | 설명 |
|------|------|
| `AddClue(clueId)` | 새 단서면 `GameSaveManager.AddClue()` + `SaveGame()` |
| `HasClue(clueId)` | 보유 여부 |
| `GetObtainedClueIds()` | 보유 목록 |

**코드 프리뷰**

```csharp
public class ClueInventoryManager : MonoSingleton<ClueInventoryManager>
{
    private GameSaveManager gameSaveManager;

    protected override void OnHostInstanceEstablished()
    {
        gameSaveManager = GameSaveManager.EnsureExists();
    }

    public bool AddClue(string clueId)
    {
        bool isAdded = gameSaveManager.AddClue(clueId);
        if (!isAdded) return false;
        gameSaveManager.SaveGame();
        return true;
    }
}
```

**⚠ 미연결 상태**: 게임플레이/UI 어디서도 `AddClue`/`HasClue`를 호출하는 곳이 없다. 데이터 구조와 매니저만 준비되어 있고, 실제 "단서를 얻는 이벤트" 자체가 아직 없음.

---

#### `ClueData` (`ClueData.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Clue/ClueData.cs` |
| **역할** | 단서 표시용 데이터(POCO) — `clueId`로 저장 목록과 매칭해서 UI에 표시할 때 사용 예정 |

**코드 프리뷰**

```csharp
[Serializable]
public class ClueData
{
    public string clueId;
    public string displayName;
    [TextArea] public string description;
    public Sprite icon;
}
```

---

## 6. Scene Flow / UI

#### `SceneFlowManager` (`SceneFlowManager.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/SceneController/SceneFlowManager.cs` |
| **역할** | 씬 전환(새 게임 시작, 이어하기, 종료)의 단일 진입점 |
| **붙는 곳** | Start_scene — SceneFlowManager Object (버튼 OnClick에서 호출) |
| **주요 의존** | `MonoSingleton<SceneFlowManager>`, `GameSaveManager` |

**Public API**

| 멤버 | 설명 |
|------|------|
| `StartNewGame()` | `GameSaveManager.ResetForNewGame("Chapter01")` 후 `GoToScene("scene_1")` |
| `ContinueGame()` | 세이브 리셋 없이 `GoToScene("scene_1")` — **UI 버튼 아직 없음, 챕터별 씬 매핑도 아직 없어서 항상 scene_1로 감** |
| `QuitGame()` | 에디터 Play 중지 / 빌드 `Application.Quit()` |
| `GoToScene(sceneName)` | `SceneManager.LoadScene(sceneName)` — 모든 씬 전환이 이 메소드를 거침 |

**코드 프리뷰**

```csharp
public class SceneFlowManager : MonoSingleton<SceneFlowManager>
{
    private const string GameSceneName = "scene_1";
    private const string NewGameChapter = "Chapter01";

    public void StartNewGame()
    {
        GameSaveManager.EnsureExists().ResetForNewGame(NewGameChapter);
        GoToScene(GameSceneName);
    }

    public void GoToScene(string sceneName)
    {
        SceneManager.LoadScene(sceneName);
    }
}
```

**실제 씬 연결(Start_scene, 에디터에서 수동 확인 완료)**

| 버튼 | OnClick 대상 | CallState |
|------|-------------|-----------|
| Start | `SceneFlowManager.StartNewGame` | EditorAndRuntime — 정상 동작 확인됨 |
| Quit | `SceneFlowManager.QuitGame` | 정상 동작 확인됨 (한때 Off로 꺼져있던 버그 발견 후 수정됨) |

**Build Settings**: `scene_1`이 `ProjectSettings/EditorBuildSettings.asset`에 등록되어 있어야 `LoadScene("scene_1")`이 성공한다(과거엔 `Start_scene`만 등록되어 있었음 — 등록 완료).

---

#### `WorldMapToggleController` (`WorldMapToggleController.cs`)

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/UI/WorldMapToggleController.cs` |
| **역할** | M키/버튼으로 월드맵 패널 토글 |
| **붙는 곳** | scene_1 — Chatting_UI (권장) |

**Public API**

| 멤버 | 설명 |
|------|------|
| `ToggleWorldMap()` | 표시/숨김 전환 |
| `SetWorldMapVisible(bool)` | 명시적 설정 |
| `OnToggleButtonClicked()` | UI Button용 |
| `SetWorldMapSprite(Sprite)` | 런타임 지도 변경 |

**코드 프리뷰**

```csharp
public void SetWorldMapVisible(bool value)
{
    isWorldMapVisible = value;
    ApplyWorldMapVisibility();
}

private void ApplyWorldMapVisibility()
{
    if (isWorldMapVisible)
        worldMapPanel.transform.SetAsLastSibling();
    worldMapPanel.SetActive(isWorldMapVisible);
}
```

**Inspector**: `worldMapPanel`, `worldMapSprite` (선택), `toggleKey` (기본 M)

---

## 7. Editor (참고)

#### `SsalBootstrapAudioMixer`

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Editor/SsalBootstrapAudioMixer.cs` |

- 메뉴: `SSAL → Audio Mixer 설정 안내 (GameSettingsMixer)`
- AudioMixer **자동 생성하지 않음**. Inspector 할당 또는 `Assets/Resources/GameSettingsMixer.mixer` 수동 생성 안내.

#### `SettingSaveManagerJsonEditorTest`

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Editor/SettingSaveManagerJsonEditorTest.cs` |

- 메뉴: `SSAL → Test settings.json 저장·로드 (Editor)`
- `persistentDataPath/settings.json`에 테스트 데이터 쓰기/읽기 후 원본 복구.

#### `MusicProfileEditor` / `MusicNodeDrawer`

| 항목 | 내용 |
|------|------|
| **경로** | `Assets/Features/Scripts/Sound/Editor/` |

- `MusicProfile` ScriptableObject의 커스텀 인스펙터와, `SerializeReference` 기반 `MusicNode` 트리를 트리 형태로 그려주는 PropertyDrawer.

---

## 문서 동기화 규칙

| 변경 종류 | 수정 문서 |
|-----------|-----------|
| 씬 GameObject / Inspector / Mermaid 흐름도 | [ScriptDiagrams.md](./ScriptDiagrams.md) Part 1 |
| 클래스·모듈 관계 다이어그램 | [ScriptDiagrams.md](./ScriptDiagrams.md) Part 2 |
| public API·코드 발췌·Inspector 표 | **본 문서** 해당 섹션 |

**파일명 ↔ 클래스명**

| 파일 | 클래스 |
|------|--------|
| `MenuSettingScript.cs` | `MenuSettingWindow` |

---

## 스크립트 커버리지

| # | 파일 | 섹션 |
|---|------|------|
| 1 | SettingSaveManager.cs | 1 |
| 2 | SettingData.cs | 1 |
| 3 | MenuSettingScript.cs | 1 |
| 4 | DialogueManager.cs | 2 |
| 5 | DialoguePathResolver.cs | 2 |
| 6 | DialogueLine.cs | 2 |
| 7 | DialogueSceneControllerBase.cs | 2 |
| 8 | Scene_1_Controller.cs | 2 |
| 9 | SFXManager.cs | 3 |
| 10 | MusicDirector.cs | 3 |
| 11 | MusicProfile.cs | 3 |
| 12 | MusicNode.cs | 3 |
| 13 | Core/MonoSingleton.cs | 4 |
| 14 | Save/GameSaveManager.cs | 4 |
| 15 | Save/GameSaveData.cs | 4 |
| 16 | Clue/ClueInventoryManager.cs | 5 |
| 17 | Clue/ClueData.cs | 5 |
| 18 | SceneController/SceneFlowManager.cs | 6 |
| 19 | UI/WorldMapToggleController.cs | 6 |
| — | Editor 4종(SsalBootstrapAudioMixer, SettingSaveManagerJsonEditorTest, MusicProfileEditor, MusicNodeDrawer) | 7 |

**삭제된 파일** (더 이상 존재하지 않음, 과거 문서에 있었음): `StartGame.cs`(`StartMenu`), `Settings/MenuButtons.cs`, `ExitGame.cs`, `Data/GlobalData.cs` — 전부 씬 어디에도 연결 안 된 죽은 코드였거나(`StartGame.cs`, `MenuButtons.cs`), `SceneFlowManager`로 기능이 흡수됨(`ExitGame.cs`), `GameSaveManager` 중심 구조로 대체됨(`GlobalData.cs`).
