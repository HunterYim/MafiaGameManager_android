# Mafia Game Manager Android 설계

## 1. 프로젝트 방향

기존 `MafiaGameAlgorithm_java` 프로젝트의 게임 규칙과 알고리즘을 기반으로 새로운 Android 애플리케이션을 개발한다.

기존 Java 프로젝트는 알고리즘 프로젝트로 유지하고, Android 앱은 별도 Repository에서 개발한다.

예상 구조:

```text
MafiaGameManager_android/
├─ docs/
│  └─ ANDROID_APP_DESIGN.md
├─ core/
│  └─ Java 기반 게임 엔진
└─ app/
   └─ Kotlin + Jetpack Compose Android UI
```

초기 Android 전환 과정에서는 기존 Java 게임 로직을 최대한 활용하고, Android/UI는 Kotlin + Jetpack Compose로 구현한다. Java → Kotlin 전체 변환은 초기 목표가 아니다.

---

## 2. 핵심 설계 원칙

### 2.1 엔진은 규칙만 강제하고 전략은 강제하지 않는다

게임 엔진은 물리적으로 또는 규칙상 불가능한 행동만 막는다. 합리적인 플레이를 강요하거나, 아군이므로 선택 불가와 같은 전략적 제한을 과도하게 두지 않는다.

예:

- 접선한 정보원/늑대인간도 마피아 공격 대상으로 선택 가능
- 다른 마피아도 생존 중이면 화면에 표시하고 선택 가능하게 둘 수 있음
- 아군 정보는 선택 제한이 아니라 팀 컬러, 직업명, 아이콘 등 추가 정보로 제공
- 복잡한 동률 상황에서 엔진이 플레이어의 이해관계를 대신 판단하지 않음

### 2.2 정보 비대칭은 UI 흐름으로 보호한다

모든 플레이어의 차례 자체가 정보가 될 수 있으므로, 능력이 없거나 행동할 수 없는 플레이어도 동일하게 휴대폰을 전달받고 자신의 상태를 확인한다.

예:

```text
3번 플레이어 행동 완료
→ "4번 플레이어에게 휴대폰을 넘겨주세요."
→ 4번 플레이어가 [다음]
→ 개인 화면 표시
```

건달에게 협박당한 플레이어도 자동 스킵하지 않는다.

```text
4번 플레이어에게 휴대폰 전달
→ 개인 화면
→ "건달의 협박으로 이번 투표에 참여할 수 없습니다."
→ [확인]
→ 다음 플레이어에게 전달
```

### 2.3 UI와 게임 규칙을 분리한다

- UI는 게임 규칙을 직접 계산하지 않는다.
- GameEngine이 게임 규칙의 Source of Truth이다.
- Compose는 `GameUiState`를 표시하고 사용자 입력을 `GameEvent`로 전달한다.
- 실제 비밀 정보는 `GameState`에 존재하지만, UI에는 현재 플레이어가 볼 수 있는 정보만 Projection하여 전달한다.
- 반대로 이미 모든 플레이어에게 공개된 정보는 다시 숨기지 않는다. 공개된 직업/상태는 대상 선택 화면 등 관련 UI에서 표시해 플레이어의 기억 부담과 오인을 줄인다.

---

## 3. 전체 아키텍처

기본 데이터 흐름:

```text
GameConfig
    ↓
GameEngine
    ↓
GameState
    ↓
GameViewModel
    ↓
GameUiState
    ↓
Compose
```

사용자 입력은 반대 방향으로 전달한다.

```text
Compose
    ↓
GameEvent
    ↓
GameViewModel
    ↓
GameEngine
    ↓
GameState 변경
```

GameEngine은 사용자의 입력을 기다리며 스레드를 블로킹하지 않는다. 입력이 필요한 지점에서 `PendingAction`을 만든 뒤 즉시 반환한다.

---

## 4. GameConfig

게임 시작 시 필요한 설정 정보.

```kotlin
data class GameConfig(
    val mode: GameMode,
    val playerNames: List<String>
)
```

`playerCount`는 `playerNames.size`에서 계산한다.

---

## 5. GameState

### 5.1 원칙

`GameState`는 현재 게임을 중단했다가 다시 시작해도 동일한 지점에서 복원할 수 있을 정도의 실제 게임 사실만 저장한다.

화면 이름, 버튼 활성화 등 UI 표현 정보는 저장하지 않는다.

### 5.2 구조

```text
GameState
│
├─ config: GameConfig
├─ day: Int
├─ phase: GamePhase
│
├─ players: List<PlayerState>
│
├─ progress: PhaseProgress
│   ├─ actorOrder
│   ├─ currentIndex
│   └─ pendingAction
│
├─ nightState: NightState
│   ├─ actions
│   ├─ privateResults
│   └─ delayedPublicEvents
│
├─ dayState: DayState
│   ├─ votes
│   └─ publicEvents
│
├─ history: GameHistory
│   ├─ eliminations
│   ├─ privateResults
│   └─ publicEvents
│
└─ result: GameResult?
```

중복 상태는 가능한 한 저장하지 않는다.

- `playerCount` → `config.playerNames.size`
- `isGameOver` → `result != null`
- `winningTeam` → `GameResult`
- 늑대인간의 `slaughterEnabled` → 저장하지 않고 현재 상태로 계산

`day`는 밤/낮 한 사이클의 번호로 사용한다. 게임 시작 시 `day = 1`이며 `day = 1`의 밤이 첫날 밤이다. 같은 `day = 1`의 낮이 끝난 뒤 `day = 2`로 증가하여 두 번째 밤으로 진입한다. 따라서 첫날 밤 전용 능력은 `day == 1`, 간첩의 홀수 밤 포섭은 `day % 2 == 1`처럼 판정할 수 있다.

---

## 6. PlayerState / JobState

```kotlin
data class PlayerState(
    val id: Int,
    val name: String,
    val job: JobState,
    val initialTeam: Team,
    val currentTeam: Team,
    val isAlive: Boolean,
    val canVoteToday: Boolean
)
```

`canVoteToday`는 영구 정보가 아니라 **해당 낮의 투표에만 유효한 상태**다. 새로운 낮이 시작되면 생존 플레이어를 기본적으로 `true`로 초기화하고, 직전 밤 건달에게 협박당한 대상만 `false`로 적용한다. 이후 다음 낮이 시작될 때 다시 계산한다.

팀 변경 예:

```text
경찰
initialTeam = CITIZEN
currentTeam = CITIZEN

↓ 간첩에게 포섭

initialTeam = CITIZEN
currentTeam = SPY
```

직업별 가변 상태는 `PlayerState`에 boolean을 계속 추가하지 않고 `JobState`가 가진다.

예:

```text
InformantState
└─ contactedMafia

WerewolfState
└─ contactedMafia

SoldierState
└─ defenseUsed

PoliticianState
└─ executionEvadeUsed

ReporterState
└─ reportUsed

UndertakerState
└─ autopsyUsed
```

도굴꾼이 직업을 획득할 때는 기존 Job 객체를 공유하지 않고 새로운 JobState를 생성한다.

---

## 7. PhaseProgress / PendingAction

### 7.1 PhaseProgress

기존 `currentPlayerIndex`의 역할을 현재 Phase의 진행 상태로 확장한다.

```kotlin
data class PhaseProgress(
    val actorOrder: List<Int>,
    val currentIndex: Int,
    val pendingAction: PendingAction?
)
```

### 7.2 PendingAction의 의미

`GamePhase`가 전체 진행 단계를 표현한다면 `PendingAction`은 엔진이 현재 정확히 어떤 사용자 입력을 기다리는지 표현한다.

```text
GamePhase
= 게임 전체가 어느 단계인가?

PendingAction
= 그 단계에서 지금 누구의 어떤 입력을 기다리는가?
```

`GamePhase`는 유한하고 상호 배타적인 진행 단계이므로 enum으로 관리한다. V1에서는 저장/복구할 가치가 있는 사용자 진행 단계 중심으로 둔다.

```kotlin
enum class GamePhase {
    NIGHT_ACTION,
    NIGHT_PRIVATE_RESULT,
    DAY_PUBLIC_RESULT,
    DISCUSSION,
    VOTING,
    EXECUTION,
    GAME_OVER
}
```

`NIGHT_RESOLUTION`처럼 사용자 입력 없이 엔진이 즉시 계산하고 지나가는 과정은 별도 장기 상태로 유지하지 않고 `advanceUntilInputRequired()` 내부 처리 단계로 둔다.

V1 구조:

```text
PendingAction
│
├─ AcknowledgePrivateStep
│   ├─ 밤 직업/능력 안내 확인
│   ├─ 개인 결과 확인
│   ├─ 협박으로 투표 불가 확인
│   └─ 사용 가능한 능력 없음 확인
│
├─ SelectPlayerTarget
│   ├─ 각종 밤 능력 대상 선택
│   └─ 테러 대상 선택
│
├─ CastVote
│
├─ AcknowledgePublicResult
│
└─ FinishDiscussion
```

예시:

```kotlin
sealed interface PendingAction {
    val id: Long
}

data class AcknowledgePrivateStep(
    override val id: Long,
    val actorId: Int,
    val type: PrivateStepType
) : PendingAction

data class SelectPlayerTarget(
    override val id: Long,
    val actorId: Int,
    val purpose: TargetSelectionPurpose,
    val targetOptions: List<TargetOption>,
    val canSkip: Boolean
) : PendingAction

data class CastVote(
    override val id: Long,
    val voterId: Int,
    val targetOptions: List<TargetOption>,
    val canAbstain: Boolean
) : PendingAction
```

### 7.3 휴대폰 전달 화면

`PASS_PHONE`은 게임 규칙 입력이 아니므로 `PendingAction`으로 만들지 않는다.

개인용 PendingAction이 존재하면 UI는 공통적으로 먼저 다음과 같은 Hand-off 화면을 표시한다.

```text
"4번 플레이어에게 휴대폰을 넘겨주세요."
[다음]
```

`[다음]`은 GameState를 변경하지 않고 UI 화면만 전환한다.

Hand-off 화면에는 다음 차례의 **생존 플레이어 이름/번호 정도만** 표시하고, 직업·진영·조사 결과·접선/포섭 정보 같은 비밀 정보는 노출하지 않는다. 사망한 플레이어는 actor order에서 제외한다. 능력을 사용할 수 없는 생존 플레이어도 정보 대칭을 위해 자신의 차례에는 휴대폰을 전달받고 Private 안내를 확인한다.

---

## 8. GameEvent

### 8.1 원칙

GameEvent는 사용자가 실제 게임 상태에 가한 입력만 표현한다.

휴대폰 전달 화면에서 `[다음]`을 누르는 것처럼 게임 규칙에 영향을 주지 않는 UI 동작은 GameEvent에 포함하지 않는다.

### 8.2 구조

```text
GameEvent
│
├─ PendingActionEvent
│   ├─ ConfirmPrivateStep
│   ├─ SelectPlayerTarget
│   ├─ SkipPlayerTarget
│   ├─ CastVote
│   ├─ AbstainVote
│   ├─ ConfirmPublicResult
│   └─ FinishDiscussion
│
└─ ConfirmManualDraw
```

예:

```kotlin
sealed interface GameEvent

sealed interface PendingActionEvent : GameEvent {
    val pendingActionId: Long
}

data class SelectPlayerTarget(
    override val pendingActionId: Long,
    val targetId: Int
) : PendingActionEvent
```

직업별 이벤트를 만들지 않는다.

경찰 조사도 UI는 다음만 전송한다.

```text
SelectPlayerTarget(
    pendingActionId = 127,
    targetId = 4
)
```

현재 PendingAction의 `purpose = POLICE_INVESTIGATE`를 보고 GameEngine이 의미를 해석한다.

`pendingActionId`를 통해 오래된 이벤트나 중복 입력을 방어한다.

---

## 9. GameEngine 진행 모델

### 9.1 핵심 원칙

Android에서는 사용자 입력을 기다리는 블로킹 루프를 사용하지 않는다.

```text
이벤트 하나 받음
↓
가능한 게임 로직 실행
↓
다음 사용자 입력이 필요한 지점까지 진행
↓
PendingAction 생성
↓
즉시 반환
```

### 9.2 기본 API

```kotlin
class GameEngine {

    fun createGame(config: GameConfig): GameState

    fun handleEvent(
        state: GameState,
        event: GameEvent
    ): GameState

    private fun advanceUntilInputRequired(
        state: GameState
    ): GameState

    private fun advanceOneStep(
        state: GameState
    ): GameState
}
```

`advanceUntilInputRequired()`의 내부 while은 사용자 입력을 기다리지 않고 짧은 순수 계산만 반복하므로 허용한다.

```text
PendingAction != null
→ 사용자 입력 필요
→ GameEngine 반환

PendingAction == null
AND result == null
→ 자동 진행
```

엔진 자체는 `suspend`일 필요가 없다. DB 저장, 파일 저장, 네트워크, 긴 지연 등이 필요할 때 ViewModel/Repository 계층에서 Coroutine을 사용한다.

금지:

```text
Thread.sleep()
사용자 입력을 기다리는 while
GameEngine 내부 UI 호출
```

---

## 10. GameManager 리팩터링 방향

기존 `GameManager` 클래스는 그대로 유지하지 않고 책임을 분해한 뒤 핵심 진행 제어 역할을 `GameEngine`으로 대체한다.

```text
기존 GameManager
├─ 상태 보관
├─ UI 입력/출력
├─ 게임 루프
├─ 직업 배정
├─ 밤 능력 처리
├─ 투표
├─ 추방
└─ 승리 판정

↓ 분해

GameState
→ 상태 보관

GameEngine
→ 게임 진행 제어

JobDistributionProvider
→ 직업 구성/배정

NightResolver
→ 밤 행동 처리

VoteResolver
→ 투표 집계

ExecutionResolver
→ 추방 및 관련 효과

VictoryResolver
→ 승리/교착 판정

GameViewModel / Compose
→ UI
```

기존 필드 매핑:

```text
GameUI ui                         → 제거, Android UI 계층
players                           → GameState.players
playerCount                       → config.playerNames.size
currentPhase                      → GameState.phase
dayCount                          → GameState.day
isGameOver                        → result != null
winningTeam                       → GameResult
currentPlayerIndex                → PhaseProgress
nightAbilityTargets               → NightState.actions
nightResultsForPrivateConfirmation→ NightState.privateResults
voteRecords                       → DayState.votes
publicAnnouncements               → PublicEvent
intimidatedPlayers                → PlayerState.canVoteToday 등
executedPlayersToday              → GameHistory.eliminations
mafiaNominations                  → NightAction(MAFIA_NOMINATE)
```

---

## 11. 직업 구조 재설계

### 11.1 세 가지 책임

```text
JobDefinition = 변하지 않는 직업 정보
JobState      = 플레이어마다 가지는 직업의 현재 상태
JobRule       = 직업의 순수 행동 규칙
```

### 11.2 JobDefinition

```kotlin
data class JobDefinition(
    val type: JobType,
    val name: String,
    val baseTeam: Team,
    val description: String,
    val hasNightAbility: Boolean
)
```

### 11.3 JobState

직업별 가변 상태만 가진다.

### 11.4 JobRule

JobRule은 UI 입력을 요청하거나 GameState를 직접 변경하지 않는다.

개념적인 책임:

```text
- 현재 능력을 사용할 수 있는가?
- 행동 종류는 무엇인가?
- 어떤 플레이어를 대상으로 보여줄 것인가?
- 규칙상 실제 선택 불가능한 대상은 누구인가?
- 투표 가중치는 얼마인가?
```

예:

```kotlin
interface JobRule {
    fun canUseNightAbility(
        state: GameState,
        actor: PlayerState
    ): AbilityAvailability

    fun getTargetOptions(
        state: GameState,
        actor: PlayerState
    ): List<TargetOption>

    fun getVoteWeight(
        state: GameState,
        player: PlayerState
    ): Int
}
```

밤 행동의 처리 순서와 상호작용 우선순위는 `JobRule`이 아니라 `NightResolver`/세부 Resolver가 담당한다. UI 입력 요청은 별도 `NightActionRequest` 모델을 중복해서 만들지 않고 `PendingAction`을 사용한다.

능력 사용 불가 이유 예:

```text
NO_NIGHT_ABILITY
ALREADY_USED
EMBARGO
CONDITION_NOT_MET
```

### 11.5 Resolver

직업 간 상호작용은 Job 객체끼리 직접 호출하지 않고 Resolver가 처리한다.

```text
NightResolver
AttackResolver
RecruitmentResolver
VoteResolver
ExecutionResolver
VictoryResolver
```

예:

```text
MAFIA_FINAL_ATTACK
↓ AttackResolver

대상이 늑대인간?
→ 공격 무효 + 접선

의사 치료 성공?
→ 공격 무효

군인 방탄 남음?
→ 방탄 소모 + 생존

그 외
→ 사망
```

---

## 12. Action / Resolver / 결과 모델

직업별 상호작용 자체는 실제 구현 단계에서 작성하고, 설계 단계에서는 책임 경계만 고정한다.

```text
JobRule
= 현재 어떤 행동을 할 수 있는가

PendingAction
= 엔진이 지금 어떤 입력을 기다리는가

GameEvent
= 사용자가 무엇을 선택했는가

NightAction
= 밤에 실제로 제출된 행동 기록

Resolver
= 제출된 행동을 해석하여 실제 게임 상태를 변경

PrivateResult / PublicEvent
= 처리 결과를 누가 알게 되는가

GameState
= 처리 후 현재 게임의 사실
```

### 12.1 NightAction

기존 `Map<Player, Player> nightAbilityTargets`를 의미 있는 행동 객체로 대체한다.

```kotlin
data class NightAction(
    val actorId: Int,
    val type: NightActionType,
    val targetId: Int?
)
```

능력을 포기한 경우에는 `NightAction` 자체를 생성하지 않는다.
따라서 탐정은 대상 플레이어가 그날 밤 하나 이상의 능동적인 `NightAction`을 실제로 제출했는지만 확인하면 된다.

테러리스트의 동반 처형처럼 밤 행동이 아닌 입력은 `NightAction`에 넣지 않고 해당 Phase의 `PendingAction`과 Resolver 흐름에서 처리한다.

### 12.2 Resolver

`NightResolver`가 밤 행동 처리의 진입점이 된다.

```text
NightState.actions
    ↓
NightResolver
    ↓
필요한 세부 Resolver 호출
    ↓
GameState 변경
+ PrivateResult
+ PublicEvent
```

초기에는 모든 직업별 Resolver를 따로 만들지 않는다.
복잡한 상호작용만 별도 Resolver로 분리하고, 단순한 행동은 `NightResolver` 내부에서 처리할 수 있다.

```text
NightResolver
├─ AttackResolver
└─ RecruitmentResolver

VoteResolver
ExecutionResolver
VictoryResolver
```

실제 공격 우선순위, 치료/방탄/접선 충돌, 포섭 상호작용 등의 세부 처리 순서는 구현 단계에서 확정한다.

### 12.3 결과 모델

개인 결과와 공개 결과를 분리한다.

```text
PrivateResult
→ 특정 플레이어에게만 알려지는 게임 사실

PublicEvent
→ 모든 플레이어에게 공개되는 게임 사실
```

결과 객체에는 완성된 UI 문장보다 가능한 한 게임 사실을 저장한다.
문장, 색상, 아이콘 등 표현은 `GameUiState`를 만드는 UI 계층에서 결정한다.

---

## 13. 마피아 지명 및 최종 공격

### 13.1 기본 흐름

생존한 실제 마피아를 플레이어 순서대로 정렬한다.

예:

```text
생존 마피아 = [2번, 5번, 8번]

2번 → MAFIA_NOMINATE
5번 → MAFIA_NOMINATE
8번 → MAFIA_FINAL_ATTACK
```

마피아가 한 명뿐이면 해당 플레이어가 바로 `MAFIA_FINAL_ATTACK`을 수행한다.

### 13.2 지명 정보는 누적 공개

앞선 마피아의 지명은 뒤의 마피아에게 비공개 정보로 전달한다.

```text
2번 마피아 → 4번 지명

5번 마피아 차례
→ "앞선 지명: 2번 → 4번" 확인
→ 본인 지명

8번 최종 공격자
→ 앞선 모든 지명 확인
→ 최종 공격 대상 결정
```

별도 `mafiaNominations` Map을 만들지 않고 `NightState.actions`에서 `MAFIA_NOMINATE` 기록을 조회한다.

### 13.3 선택을 과도하게 제한하지 않는다

마피아 공격 화면에는 원칙적으로 모든 플레이어를 표시한다.

- 생존 플레이어: 전략적으로 자유롭게 선택 가능
- 접선한 정보원/늑대인간: 선택 가능
- 같은 마피아: 선택 가능
- 사망 플레이어: UI에 표시할 경우 선택 불가

접선/아군 정보는 선택을 막는 대신 해당 마피아에게 팀 컬러, 직업명 등으로 표시한다.

```text
3번 플레이어
늑대인간 · 마피아팀

4번 플레이어
정보원 · 마피아팀
```

이 정보는 현재 화면을 보는 플레이어가 실제로 알고 있는 범위에서만 보여준다.

### 13.4 실제 공격 처리

`MAFIA_NOMINATE`는 의견 기록만 하고 게임 효과를 만들지 않는다.

`MAFIA_FINAL_ATTACK` 하나만 `AttackResolver`에 전달하여 실제 공격을 처리한다.

---

## 14. Dynamic Visibility / Target UI

UI는 `GameState`의 Player 객체를 그대로 받지 않는다.

```kotlin
data class PlayerUi(
    val id: Int,
    val name: String,
    val isAlive: Boolean,
    val visibleJobName: String? = null,
    val visibleTeam: Team? = null
)
```

대상 선택 UI는 개념적으로 다음과 같은 정보를 받을 수 있다.

```kotlin
data class TargetOptionUi(
    val playerId: Int,
    val name: String,
    val isAlive: Boolean,
    val enabled: Boolean,
    val visibleJob: JobType?,
    val visibleTeam: Team?
)
```

동일 플레이어라도 관찰자에 따라 `visibleJob`, `visibleTeam` 값이 달라질 수 있다.

---

## 15. 기본 게임 흐름

### 밤

밤의 사용자 행동 단계는 별도 `ROLE_CONFIRMATION` Phase를 두지 않고 `NIGHT_ACTION` 하나로 통일한다. 첫날 밤과 이후 밤 모두 살아있는 플레이어를 차례대로 순회한다.

```text
HANDOFF
→ 밤 직업/능력 안내
→ 대상 선택 / 능력 포기 / 사용 가능한 능력 없음 확인
→ 다음 생존 플레이어 HANDOFF
```

휴대폰을 받은 직후 각 플레이어에게 매일 밤 자신의 현재 직업과 능력을 다시 안내한다. 표현 예시는 다음과 같다.

```text
"플레이어 4님, 당신은 경찰입니다. 다른 생존 플레이어를 조사할 수 있습니다."
"플레이어 6님, 당신은 기자입니다. 취재할 플레이어를 선택하거나 이번 밤에는 사용하지 않을 수 있습니다."
```

첫날 밤도 동일한 `NIGHT_ACTION` 흐름을 사용하되 기자 엠바고, 도굴꾼 첫날 판정 등 `day == 1` 전용 규칙만 적용한다. 모든 생존 플레이어의 밤 행동 입력이 끝나면 Resolver가 행동을 처리하고 `NIGHT_PRIVATE_RESULT`로 이동한다.

### 개인 결과 확인

밤 처리 후에는 **결과 유무와 관계없이 모든 생존 플레이어를 다시 한 번 순회**한다. 결과가 있는 플레이어에게만 휴대폰을 전달하면 그 자체가 비밀 정보가 될 수 있기 때문이다.

```text
HANDOFF
→ 개인 결과 있음: 결과 표시
→ 개인 결과 없음: "이번 밤에 확인할 개인 결과가 없습니다."
→ 다음 생존 플레이어 HANDOFF
```

사망한 플레이어는 이 순회에서 제외한다.

### 낮

1. 공개 결과 발표
2. 토론
3. 투표
4. 추방
5. 테러리스트 등 추방 연계 효과 처리
6. 승리 조건 검사
7. 다음 밤

### 투표 기본 규칙

- 생존하고 투표 가능한 플레이어는 다른 플레이어뿐 아니라 **자기 자신에게도 투표 가능**하다.
- 투표는 **기권/패스 가능**하다.
- 건달의 협박 등으로 투표권이 없는 플레이어는 휴대폰 전달 순서는 유지하되 실제 투표는 하지 못한다.
- 최다 득표자가 한 명이면 해당 플레이어를 추방 대상으로 처리한다.
- **최다 득표가 동률이면 아무도 추방하지 않고 바로 다음 밤으로 이동**한다. 재투표는 하지 않는다.
- 전원 기권 등으로 유효 득표가 없을 때도 아무도 추방하지 않고 다음 밤으로 이동한다.
- 정치인의 2표 효과는 `VoteResolver`에서 투표 가중치로 반영한다.

---

## 16. 직업별 확정 규칙

### 16.0 능동 능력 공통 원칙

- 플레이어가 직접 대상을 선택하거나 행동 여부를 결정하는 **능동 능력은 기본적으로 포기 가능**하게 통일한다. (`canSkip = true`)
- 예외적으로 **테러리스트의 추방 연계 대상 선택은 반드시 수행**한다. (`canSkip = false`) 테러리스트가 낮 투표로 실제 추방되면 살아있는 다른 플레이어 1명을 반드시 선택해야 한다.
- 능력을 포기한 경우 해당 `NightAction` 또는 연계 행동은 제출되지 않은 것으로 처리하며, 탐정에게도 `사용하지 않음`으로 판정한다.
- 한 직업이 한 밤에 여러 능동 행동을 가질 경우 각 행동은 서로 독립적으로 포기할 수 있다.
- 군인 방탄/경계, 정치인 투표 가중치/추방 방어, 도굴꾼의 조건 충족 시 직업 획득처럼 사용자 대상 선택이 없는 패시브·자동 효과는 이 `canSkip` 원칙의 대상이 아니다.

### 마피아

- 밤마다 공격 대상을 정할 수 있음
- 여러 마피아 생존 시 앞선 마피아는 지명, 마지막 순서의 마피아가 최종 공격 결정
- 다음 마피아는 앞선 마피아들의 지명 내역을 확인할 수 있음
- 최종 공격자는 앞선 지명과 다른 대상을 자유롭게 선택 가능
- 지명과 최종 공격 모두 포기 가능 (`canSkip = true`)
- 생존 플레이어는 같은 마피아 진영이어도 전략적으로 선택 가능하며, 접선/동료 정보는 선택 제한 대신 UI 정보로 제공
- **자기 자신도 지명/최종 공격 대상으로 선택 가능**하다. 엔진은 전략적으로 불리하다는 이유로 제한하지 않는다.
- 실제 공격 효과는 `MAFIA_FINAL_ATTACK`에 대해서만 `AttackResolver`가 처리

### 정보원

- 밤마다 **조사**와 **추천**을 각각 사용할 수 있으며, 두 행동은 서로 독립적이다.
- 조사: 살아있는 다른 플레이어 1명을 선택해 해당 플레이어의 실제 `JobType`을 정보원만 확인한다. 조사 자체를 포기할 수 있다. (`canSkip = true`)
- 추천: 살아있는 다른 플레이어 1명을 선택하며, 조사 대상과 같아도 되고 달라도 된다. 추천 자체를 포기할 수 있다. (`canSkip = true`)
- 조사와 추천은 각각 독립적으로 포기할 수 있으므로 둘 중 하나만 사용하거나 둘 다 포기할 수도 있다.
- 추천 대상은 **그날 밤의 마피아 공격 선택 전에 제공하지 않는다.** 모든 밤 행동 처리 후 개인 결과 확인(Private) 단계에서 생존 마피아들에게 `정보원의 추천 대상`으로 표시한다. 따라서 같은 밤 공격 판단에는 사용할 수 없고 이후 플레이 판단용 정보가 된다. 추천은 참고 정보일 뿐 마피아의 지명/최종 공격을 강제하지 않는다.
- 조사 결과 자체는 마피아에게 자동 공유하지 않는다.
- 조사 대상의 실제 직업이 `MAFIA`이면 접선이 성립하고, 이후 모든 생존 마피아와 정보원이 서로의 존재를 인지한다.
- 간첩에게 포섭되어 `currentTeam = SPY`가 되어도 정보원의 조사/추천 능력과 UI 흐름은 동일하게 유지한다.
- 포섭 이후에도 추천은 계속 마피아에게 전달되며, 어떤 대상을 조사하고 어떤 대상을 추천할지는 정보원 플레이어가 현재 승리 목표에 맞게 전략적으로 판단한다.
- 시스템은 포섭 여부에 따라 추천 내용을 자동 변경하거나 가짜 정보를 생성하지 않는다.
- 직접 제거 능력은 없다.

### 늑대인간

- 처음에는 마피아와 서로의 존재를 알지 못한다.
- 마피아의 최종 공격 대상이 되면 해당 공격은 무효가 되고 마피아와 접선한다.
- 접선이 성립하면 늑대인간과 모든 생존 마피아가 서로의 존재를 인지한다.
- 살육은 다음 두 조건을 모두 만족할 때 사용할 수 있다.
  - 마피아와 접선 완료 (`contactedMafia = true`)
  - 현재 생존 마피아 0명
- 살육 가능 여부는 현재 상태에서 동적으로 계산하며 별도 `slaughterEnabled`를 저장하지 않는다.
- 살육 시 살아있는 다른 플레이어 한 명을 선택할 수 있다.
- 살육은 포기 가능하다. (`canSkip = true`)
- 살육은 의사의 치료로 막을 수 없다.
- 군인은 방탄이 남아 있다면 살육을 한 번 방어할 수 있다.
- 간첩에게 포섭되어 `currentTeam = SPY`가 되어도 직업 능력은 유지한다. 접선 완료 + 생존 마피아 0명 조건을 만족하면 간첩팀 상태에서도 살육할 수 있다.
- 접선하지 못한 늑대인간은 생존 마피아가 0명이 되어도 살육할 수 없다.

### 경찰

- 생존 중이면 매일 밤 다른 생존 플레이어 한 명을 조사할 수 있다.
- 조사 능력은 포기 가능하다. (`canSkip = true`)
- 판정 기준은 `currentTeam`이 아니라 대상의 실제 직업이 `JobType.MAFIA`인지 여부
- 정보원/늑대인간이 접선했거나 다른 플레이어가 간첩팀으로 포섭되었더라도 실제 직업이 `MAFIA`가 아니면 "마피아 아님"으로 판정
- 조사 선택은 `NightAction(POLICE_INVESTIGATE)`로 기록
- 조사 결과는 경찰 개인에게만 `PrivateResult`로 제공

### 의사

- 생존 중이면 매일 밤 생존 플레이어 한 명을 치료할 수 있다.
- 자기 자신 치료 가능
- 치료 능력은 포기 가능하다. (`canSkip = true`)
- 치료 선택은 `NightAction(DOCTOR_HEAL)`로 기록
- 마피아의 최종 공격은 막을 수 있음
- 늑대인간 살육은 막지 못함
- 실제 공격 여부와 치료 성공 여부는 밤 행동을 모두 수집한 뒤 `AttackResolver`가 판정
- 치료가 실제로 마피아 공격을 막은 경우 다음날 치료 대상 이름까지 공개
- 마피아가 공격을 포기했거나 치료 대상이 공격받지 않은 경우 치료 행동은 기록되지만 공개 효과는 발생하지 않음

### 군인

- 시민 진영의 패시브 직업
- 마피아 공격 또는 늑대인간 살육을 게임 중 한 번 방어하며, 두 공격 유형이 하나의 방어 횟수를 공유한다.
- 방어가 실제로 발동하면 방탄을 소모하고 생존한다.
- 방어 성공 직후 군인 본인에게는 공격을 막았다는 정도의 Private 결과만 제공한다.
- 공격한 마피아 또는 늑대인간에게는 공격에 실패했다는 사실만 Private 결과로 제공하며, 그 시점에는 군인 정체를 직접 알려주지 않는다.
- 다음 낮 공개 결과에서 해당 플레이어가 군인이며 방어에 성공했다는 사실을 전체에게 발표한다.
- 간첩 포섭은 경계 능력으로 무효화한다. 이 포섭 방어는 공격 방어 1회를 소모하지 않는다.
- 간첩은 포섭 실패 시 대상이 군인임을 알게 된다.
- 군인의 방탄/경계는 패시브 효과이므로 탐정의 능동적 NightAction 감지 대상이 아니다.

### 정치인

- 시민 진영의 패시브 직업이다.
- `논객/권력`: 살아있는 동안 본인의 투표는 항상 2표로 계산한다.
- `처세/조작`: 낮 투표 결과 본인이 추방 대상으로 확정되면 게임 중 한 번 자동 발동하여 추방을 무효화한다.
- `처세/조작`은 선택형 능력이 아니며, 사용 가능 상태에서 추방 대상이 되면 반드시 자동 발동한다.
- 추방 무효화가 발동하면 정치인이라는 사실을 전체 공개한다.
- 해당 날에는 차순위 득표자를 대신 추방하지 않고 **아무도 추방되지 않은 채** 다음 진행으로 넘어간다.
- 마피아 공격, 늑대인간 살육 등 밤 공격에는 추방 방어가 적용되지 않는다.
- 간첩에게 포섭되어 `currentTeam = SPY`가 되어도 투표 2표와 1회 추방 방어 능력은 그대로 유지한다.
- 두 능력 모두 패시브이므로 탐정의 능동적 NightAction 감지 대상이 아니다.

### 장의사

- 시민 진영의 1회성 밤 능력 직업이다.
- 게임 중 한 번 이미 사망한 플레이어 한 명을 선택해 부검할 수 있다.
- 능력 사용 가능 상태에서도 부검을 포기할 수 있다. (`canSkip = true`)
- 사망 원인과 관계없이 이미 사망한 플레이어라면 모두 대상이 될 수 있다.
- 이미 직업이 공개된 사망자도 다시 부검 대상으로 선택할 수 있으며, 엔진이 이를 전략적으로 제한하지 않는다.
- 부검한 대상의 실제 직업은 다음 낮 전체에게 공개한다.
- 장의사가 능력을 사용한 같은 밤에 사망하면 부검 결과는 취소된다.
- 실제로 부검 대상을 선택해 제출한 밤에는 탐정의 능동적 NightAction 감지 대상이 된다.
- 간첩에게 포섭되어 `currentTeam = SPY`가 되어도 남아 있는 부검 능력은 그대로 유지한다.
- 대상 선택 UI에서는 이미 모두에게 공개된 직업 정보가 있다면 함께 표시한다. 예를 들어 군인의 방어 성공 공개, 정치인의 추방 방어 공개, 기자 취재로 공개된 직업, 이전 장의사 부검 결과 등 공개적으로 확정된 정보는 숨기지 않는다.
- 이 표시는 선택을 막기 위한 제한이 아니라, 플레이어가 이미 알려진 정보를 다시 확인해 오인하지 않도록 돕기 위한 UI 정보다.

### 건달

- 시민 진영의 반복 사용 가능한 밤 능력 직업이다.
- 살아있는 동안 매일 밤 자신을 제외한 살아있는 플레이어 한 명을 협박할 수 있다.
- 협박 능력은 사용을 포기할 수 있다. (`canSkip = true`)
- 같은 플레이어를 여러 밤 연속으로 협박할 수 있으며, 엔진이 연속 선택을 제한하지 않는다.
- 실제로 협박 대상을 선택해 제출한 밤에는 탐정의 능동적 NightAction 감지 대상이 된다.
- 협박당한 플레이어는 해당 밤의 Private 결과 확인 단계에서 자신이 협박당했다는 사실을 확인한다.
- 협박 효과는 다음 낮 투표에 적용되며, 대상은 그날 투표할 수 없다.
- 협박당한 플레이어도 낮 투표 순서에서 자동으로 건너뛰지 않는다. 정상적으로 휴대폰을 전달받고 Private 화면에서 `건달의 협박으로 이번 투표에 참여할 수 없습니다.`라는 안내를 확인한 뒤 다음 플레이어에게 넘어간다.
- 협박 대상이 밤 사이 사망했다면 다음 낮 투표 제한은 실질적인 효과가 없다.
- 간첩에게 포섭되어 `currentTeam = SPY`가 되어도 건달의 능력은 그대로 유지하며, 누구를 협박할지는 플레이어의 전략적 선택에 맡긴다.

### 기자

- 시민 진영의 1회성 밤 능력 직업이다.
- 첫날 밤에는 엠바고로 능력을 사용할 수 없다.
- 둘째 밤부터 살아있는 다른 플레이어 한 명을 선택해 취재할 수 있다.
- 능력 사용 가능 상태에서도 취재를 포기할 수 있다. (`canSkip = true`)
- 취재한 대상의 실제 직업은 다음 낮 전체에게 공개한다.
- 취재 대상이 같은 밤에 사망하더라도 기자 본인이 살아 있다면 예정대로 다음 낮에 해당 대상의 실제 직업을 공개한다.
- 기자가 취재한 같은 밤에 사망하면 취재 결과는 취소된다.
- 실제로 취재 대상을 선택해 제출한 밤에는 탐정의 능동적 NightAction 감지 대상이 된다.
- 간첩에게 포섭되어 `currentTeam = SPY`가 되어도 아직 사용하지 않은 취재 능력은 그대로 유지한다.
- 기자 취재로 공개된 직업은 이후 관련 플레이어 선택 UI에서 모두가 아는 공개 정보로 표시한다.

### 탐정

- 시민 진영의 반복 사용 가능한 밤 능력 직업이다.
- 살아있는 동안 매일 밤 자신을 제외한 살아있는 플레이어 한 명을 조사할 수 있다.
- 조사 능력은 사용을 포기할 수 있다. (`canSkip = true`)
- 조사 결과는 대상이 그날 밤 하나 이상의 능동적인 `NightAction`을 실제로 제출했는지만 알려준다.
- 행동의 종류나 대상, 성공 여부는 알려주지 않는다.
- 능동 행동을 시도했다면 효과가 실패하더라도 `사용함`으로 판정한다. 예를 들어 간첩의 군인 포섭 실패나 군인에게 막힌 마피아 공격도 사용으로 본다.
- 포기 가능한 능력을 실제로 포기했다면 `사용하지 않음`으로 판정한다.
- 정보원의 조사+추천, 홀수 밤 간첩의 포섭+투표 지령처럼 한 플레이어가 여러 능동 행동을 수행할 수 있는 경우 하나라도 실제 제출했다면 `사용함`으로 표시한다.
- 군인의 방탄/경계, 정치인의 투표 가중치/추방 방어 같은 패시브 효과는 감지하지 않는다.
- 간첩에게 포섭되어 `currentTeam = SPY`가 되어도 탐정 능력은 그대로 유지한다.

### 도굴꾼

- 도굴 능력은 **게임의 첫날 밤에만 한 번 자동 판정**한다. 이후 밤에는 다시 발동하지 않는다.
- 첫날 밤에 다른 플레이어가 사망하면, 그 밤에 **가장 먼저 사망한 플레이어의 실제 직업**을 획득한다.
- 직업 획득 시 도굴꾼의 직업과 `currentTeam`을 새 직업에 맞게 변경한다.
- 기존 플레이어의 `JobState` 인스턴스나 능력 사용 이력은 상속하지 않고, 획득한 직업의 **새로운 JobState**를 생성한다. 따라서 1회성 능력/방어 횟수 등도 새 상태에서 시작한다.
- 첫날 밤에 아무도 사망하지 않으면 도굴 능력은 종료되고, 도굴꾼은 **일반 시민 직업 + 시민팀**으로 전환한다. 이후 도굴 기회는 다시 생기지 않는다.
- 도굴꾼 본인이 첫날 밤에 사망하면 직업 획득이나 일반 시민 전환 등 추가 효과 없이 그대로 사망 처리한다.
- 직업을 획득했는지, 어떤 직업으로 바뀌었는지는 전체 공개하지 않는다. 생존한 도굴꾼 본인에게만 Private 결과로 새 직업을 알려준다.
- 사용자 입력 없이 자동으로 처리되는 효과이므로 탐정의 능동 `NightAction` 감지 대상이 아니다.

### 테러리스트

- 낮 투표로 실제 추방되면 살아있는 다른 플레이어 한 명을 **반드시 선택**해 함께 탈락시킨다.
- 테러 대상 선택은 포기할 수 없다. (`canSkip = false`)
- 테러 대상의 실제 `JobType`이 `MAFIA`인 경우에만 해당 플레이어가 마피아였음을 전체 공개한다. 정보원/늑대인간 등 마피아 진영의 다른 직업은 공개하지 않는다.
- 테러 대상 선택은 `PendingAction.SelectPlayerTarget(purpose=TERROR_RETALIATION, canSkip=false)`를 사용한다.
- 대상 선택과 동반 탈락 처리가 모두 끝난 뒤 승리 조건을 검사한다.

### 간첩

- 생존 중이면 **매일 밤 투표 지령**을 사용할 수 있으며, 사용하지 않고 포기할 수도 있다. (`canSkip = true`)
- 투표 지령을 사용하면 다음 낮에 투표할 플레이어 한 명을 선택해 살아있는 간첩팀 플레이어들의 Private 단계에서 공유한다.
- 투표 지령은 강제가 아닌 추천 정보이며, 실제 투표 대상 선택은 각 플레이어의 자유다.
- 홀수 밤에는 투표 지령에 더해 **포섭**도 사용할 수 있어 최대 두 번 행동한다. 두 행동은 각각 독립적으로 포기할 수 있다.
- 홀수 밤에 두 행동을 모두 사용하는 경우 처리 순서는 `포섭 → 투표 지령`으로 한다. 포섭에 성공한 플레이어는 즉시 `currentTeam = SPY`가 되므로 같은 밤의 투표 지령도 확인한다.
- 포섭은 살아있는 다른 플레이어 한 명을 대상으로 하며 포기할 수 있다. (`canSkip = true`)
- 포섭 성공 시 간첩은 대상의 실제 직업을 확인하고 대상의 `currentTeam`을 `SPY`로 변경한다. 기존 직업과 직업 능력은 그대로 유지한다.
- 포섭된 플레이어 본인에게는 자신이 간첩팀으로 전환되었다는 사실을 Private 결과로 알려준다.
- 간첩은 자신이 포섭한 대상을 알고 있지만, 포섭된 플레이어에게 실제 간첩의 정체를 알려주지 않는다. 포섭된 간첩팀 플레이어끼리도 서로의 정체를 알 수 없다.
- 정보원/늑대인간을 포섭할 수 있으며 기존 접선 상태와 직업별 상태도 유지한다.
- 실제 직업이 `MAFIA`인 대상은 포섭할 수 없으며, 실패 결과에서 대상이 마피아였음을 간첩에게 알려준다.
- 군인은 경계 능력으로 포섭을 막으며, 이 경우에도 실패 결과에서 대상이 군인이었음을 간첩에게 알려준다.
- 따라서 포섭 실패는 단순한 실패 메시지가 아니라 실패 원인(마피아/군인)을 구분하여 Private 결과로 제공한다.
- 포섭에 성공하면 대상 신원은 숨긴 채 `누군가 간첩에게 포섭되었습니다` 수준의 `PublicEvent`를 발생시켜 포섭 성공 사실만 전체에게 공개한다. 공개 이벤트 객체에는 포섭 대상 `targetId` 자체를 넣지 않는다. 대상 신원은 간첩에게 제공되는 Private 정보/게임 상태에만 존재하게 하여 UI Projection 실수로 공개되는 것을 방지한다.

---

## 17. 승리 / 자동 무승부 / 수동 무승부

### 17.1 기본 철학

기본 승리는 적대 진영의 섬멸로 정의한다.

자동 무승부는 일반적인 미래 행동을 모두 탐색하는 복잡한 교착 탐지기가 아니라, **최종 생존자가 정확히 2명인 1:1 상황에서 명시적으로 정의한 영구 교착 조합**에만 적용한다.

자동 판정 순서:

```text
1. 테러리스트 보복 등 필수 후속 처리 완료
2. 명확한 섬멸 승리 판정
3. 생존자 2명 + 서로 다른 currentTeam이면 최종 1:1 교착 검사
4. 명시된 영구 교착 조합이면 자동 무승부
5. 그 외에는 게임 계속
```

교착 여부를 판단할 때는 각 플레이어가 **자기 진영의 승리를 위해 정상적으로 플레이한다는 전제**를 사용한다.

따라서 다음과 같은 행동은 교착 해소 수단으로 계산하지 않는다.

- 상대 진영을 이롭게 하기 위한 자기 투표
- 고의적인 기권으로 상대에게 승리를 넘기는 행동
- 자신이 이길 수 있는 제거/방어/포섭 능력을 의도적으로 포기하는 행동

반대로 정치인의 2표, 건달의 투표 봉쇄, 실제 간첩의 포섭, 접선 늑대인간의 살육처럼 **스스로 판을 깨고 승리로 진행할 수 있는 수단**이 남아 있으면 자동 무승부가 아니다.

`1:1:1`, `2:2`, 기타 다인 동률은 정보 비대칭, 협상, 투표 선택에 따라 상태가 달라질 수 있으므로 자동 무승부로 판정하지 않는다.

### 17.2 클래식 모드 승리

시민팀 승리:

- 마피아 진영의 모든 생존 플레이어 제거
- 마피아 진영 = 마피아 + 정보원 + 늑대인간
- 접선 여부와 무관하게 보조직업도 섬멸 대상

마피아팀 승리:

- 모든 시민팀 플레이어 제거

### 17.3 클래식 모드 자동 무승부

다음 최종 1:1은 자동 무승부로 처리한다.

#### A. 정보원 vs 제거 수단 없는 1표 시민

정보원은 직접 제거 능력이 없으므로 다음 시민 직업과의 최종 1:1은 영구 교착으로 본다.

```text
정보원
vs
일반 시민 / 경찰 / 의사 / 군인 / 장의사 / 기자 / 탐정 / 테러리스트
→ Draw(AUTO_STALEMATE)
```

정치인은 2표로 투표 우위를 만들 수 있고, 건달은 상대의 투표권을 막을 수 있으므로 이 목록에서 제외한다.

#### B. 미접선 늑대인간 vs 제거 수단 없는 1표 시민

미접선 늑대인간은 생존 마피아가 0명이 되어도 살육할 수 없으므로 다음 조합을 자동 무승부로 처리한다.

```text
미접선 늑대인간
vs
일반 시민 / 경찰 / 의사 / 군인 / 장의사 / 기자 / 탐정 / 테러리스트
→ Draw(AUTO_STALEMATE)
```

접선된 늑대인간은 생존 마피아가 0명이면 살육할 수 있으므로 교착이 아니다.

#### C. 마피아 vs 의사

```text
마피아
vs
자기치료 가능한 의사
→ Draw(AUTO_STALEMATE)
```

의사는 정상적인 자기 진영 플레이에서 매일 자기 자신을 치료해 마피아의 공격을 계속 막을 수 있다.

군인은 방탄이 1회뿐이므로 `마피아 vs 군인`은 영구 교착이 아니다.

### 17.4 간첩 모드 승리

기본은 3팀 섬멸전이다.

시민팀 승리:
- 마피아팀과 간첩팀 모두 섬멸

마피아팀 승리:
- 시민팀과 간첩팀 모두 섬멸

간첩팀 승리:
- 시민팀과 마피아팀 모두 섬멸

승리 판정은 `currentTeam` 기준이다.

### 17.5 간첩 모드 자동 무승부

자동 무승부는 간첩 모드에서도 **최종 생존자 2명의 1:1**에만 적용한다.

#### A. 마피아팀 vs 시민팀

간첩팀이 전멸하고 마피아팀과 시민팀만 남은 최종 1:1은 클래식 모드의 교착 규칙을 동일하게 적용한다.

```text
정보원 vs 제거 수단 없는 1표 시민
미접선 늑대인간 vs 제거 수단 없는 1표 시민
마피아 vs 의사
→ Draw(AUTO_STALEMATE)
```

#### B. 실제 간첩 vs 시민 군인

```text
간첩
vs
군인
→ Draw(AUTO_STALEMATE)
```

군인은 간첩의 포섭을 영구적으로 막고, 간첩과 군인 모두 상대를 직접 제거하거나 투표 우위를 만들 수 없다.

실제 간첩은 군인을 제외한 대부분의 시민 직업을 포섭할 수 있으므로 다른 일반적인 `간첩 vs 시민` 1:1은 교착으로 보지 않는다.

#### C. 실제 간첩 사망 후 포섭된 간첩팀원 vs 시민팀

실제 간첩이 이미 사망해 더 이상 포섭 능력이 없고, 포섭된 간첩팀원과 시민팀 플레이어가 모두 **제거/포섭/투표 우위 수단이 없는 1표 직업**이라면 자동 무승부로 처리한다.

간첩팀 쪽 대표 비공격 1표 직업:

```text
일반 시민 / 경찰 / 의사 / 정보원 / 미접선 늑대인간 /
장의사 / 기자 / 탐정 / 테러리스트
```

시민팀 쪽 대표 비공격 1표 직업:

```text
일반 시민 / 경찰 / 의사 / 군인 /
장의사 / 기자 / 탐정 / 테러리스트
```

예:

```text
간첩팀 경찰 vs 시민 의사
간첩팀 정보원 vs 시민 군인
간첩팀 테러리스트 vs 시민 탐정
→ Draw(AUTO_STALEMATE)
```

정치인, 건달, 살육 가능한 접선 늑대인간처럼 스스로 판을 깰 수 있는 직업이 한쪽에 있으면 자동 무승부가 아니다.

#### D. 마피아팀 vs 간첩팀

다음 조합은 최종 1:1에서 영구 교착으로 본다.

```text
실제 마피아 vs 간첩팀 의사
→ 의사의 자기치료로 마피아 공격을 영구 방어

실제 마피아 vs 간첩팀 늑대인간
→ 마피아 공격은 늑대인간에게 무효화되고 접선됨
→ 실제 마피아가 살아 있으므로 늑대인간은 살육할 수 없음

마피아팀 정보원 또는 미접선 늑대인간
vs
실제 간첩이 아닌 비공격 1표 간첩팀원
→ 양쪽 모두 제거/포섭/투표 우위 수단 없음
```

위 조합은 모두 `Draw(AUTO_STALEMATE)`로 처리한다.

실제 간첩이 살아 있고 상대가 포섭 가능한 정보원/늑대인간이라면 교착이 아니며, 실제 마피아가 상대를 정상적으로 공격해 제거할 수 있는 경우도 교착이 아니다.

### 17.6 자동 무승부로 처리하지 않는 상황

다음은 자동 교착 판정을 하지 않고 게임을 계속 진행한다.

```text
1 : 1 : 1
2 : 2
3 : 3
기타 다인 동률
세 진영이 동시에 생존한 상황
포섭된 간첩팀 플레이어가 여러 명 남아 서로의 정체를 모르는 상황
정보/협상/투표 선택으로 결과가 달라질 수 있는 상황
명시된 최종 1:1 교착 목록에 없는 상황
```

특히 포섭된 간첩팀 플레이어끼리는 서로의 정체를 모르므로, 팀 숫자가 같더라도 실제 투표가 반드시 진영별 동수로 모인다고 가정하지 않는다.

### 17.7 수동 무승부

자동 무승부 대상이 아니더라도 실제 플레이에서 참가자들이 더 이상 게임을 진행할 필요가 없다고 합의하면 언제든 메뉴에서 수동 무승부 종료를 선택할 수 있다.

```text
게임 메뉴
→ 게임 종료
→ 무승부로 종료
→ 최종 확인
```

```kotlin
GameResult.Draw(
    reason = MANUAL_AGREEMENT
)
```

`1:1:1`, `2:2`, 반복되는 전략적 동률 등은 엔진이 자동으로 판정하지 않고 이 수동 무승부 기능으로 종료할 수 있다.

---

## 18. GameResult

```kotlin
sealed interface GameResult {

    data class Victory(
        val team: Team,
        val reason: VictoryReason
    ) : GameResult

    data class Draw(
        val reason: DrawReason
    ) : GameResult
}
```

예:

```text
Victory(MAFIA, ELIMINATION)
Draw(AUTO_STALEMATE)
Draw(MANUAL_AGREEMENT)
```

V1의 `DrawReason`은 다음 두 값을 사용한다.

```text
AUTO_STALEMATE
MANUAL_AGREEMENT
```

---

## 19. 직업 구성

### 공통

필수 시민 직업:

```text
경찰 1
의사 1
```

보조 직업:

```text
정보원 또는 늑대인간 중 랜덤 1명
```

특수 시민 후보는 중복 없이 랜덤 선택:

```text
군인
정치인
장의사
건달
기자
탐정
도굴꾼
테러리스트
```

### 클래식 모드

| 인원 | 마피아 | 보조 | 경찰 | 의사 | 특수 시민 | 일반 시민 |
|---:|---:|---:|---:|---:|---:|---:|
| 4 | 1 | 0 | 1 | 1 | 1 | 0 |
| 5 | 1 | 0 | 1 | 1 | 2 | 0 |
| 6 | 1 | 1 | 1 | 1 | 2 | 0 |
| 7 | 1 | 1 | 1 | 1 | 3 | 0 |
| 8 | 2 | 1 | 1 | 1 | 3 | 0 |
| 9 | 2 | 1 | 1 | 1 | 4 | 0 |
| 10 | 2 | 1 | 1 | 1 | 4 | 1 |
| 11 | 3 | 1 | 1 | 1 | 5 | 0 |
| 12 | 3 | 1 | 1 | 1 | 5 | 1 |

### 간첩 모드

| 인원 | 마피아 | 보조 | 간첩 | 경찰 | 의사 | 특수 시민 | 일반 시민 |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 9 | 2 | 1 | 1 | 1 | 1 | 3 | 0 |
| 10 | 2 | 1 | 1 | 1 | 1 | 4 | 0 |
| 11 | 3 | 1 | 1 | 1 | 1 | 4 | 0 |
| 12 | 3 | 1 | 1 | 1 | 1 | 4 | 1 |

직업 구성 규칙은 core의 단일 Source of Truth로 관리하며 UI에 중복 하드코딩하지 않는다.

---

## 20. GameUiState / GameScreen 및 UI Projection

### 20.1 기본 원칙

UI는 `GameState`와 `NightAction`을 직접 해석하지 않는다.

데이터 흐름은 다음과 같다.

```text
사용자 입력
→ GameEvent
→ GameEngine
→ NightAction / VoteRecord 등 실제 행동 기록
→ Resolver 처리
→ GameState 변경
→ PrivateResult / PublicEvent 생성 및 기록
→ GameViewModel이 현재 사용자에게 보이는 정보만 Projection
→ GameUiState
→ Compose 렌더링
```

`GameUiState`는 화면 표시용 Projection이며 Source of Truth가 아니다.

따라서 이미 공개되었거나 특정 플레이어가 알고 있는 정보가 이후 화면에서도 계속 보여야 한다면 그 근거는 `GameState` 쪽에 남아 있어야 한다.

V1에서는 별도의 복잡한 Knowledge DB를 만들지 않고 다음 기록을 활용한다.

```text
PrivateResult
= 특정 플레이어만 알게 된 사실

PublicEvent
= 모두에게 공개된 사실
```

현재 밤/낮 진행에 필요한 임시 결과는 `NightState`/`DayState`에 두고, 해당 페이즈가 종료된 뒤에도 다시 참조해야 하는 결과는 `GameHistory`에 누적한다. V1에서는 복잡한 별도 Knowledge DB를 만들지 않고 `GameHistory.privateResults`, `GameHistory.publicEvents`, `GameHistory.eliminations` 정도로 유지한다.

즉 `NightState`/`DayState`는 **현재 사이클의 작업 공간**, `GameHistory`는 **이미 확정되어 이후에도 참조할 사실의 기록** 역할을 한다. `GameViewModel`은 현재 viewer에 맞는 History만 필터링하여 `GameUiState`로 Projection한다.

예:

```text
기자의 취재로 4번의 직업 공개
→ PublicEvent 기록
→ 이후 대상 선택 UI에서도 4번의 공개 직업 표시

경찰이 5번을 조사
→ 해당 경찰의 PrivateResult 기록
→ 이후 해당 경찰의 화면에서는 이미 확인한 정보를 표시할 수 있음
```

### 20.2 GamePhase와 GameScreen 분리

`GamePhase`는 게임 규칙상 현재 진행 단계를 의미하고, `GameScreen`은 그 단계를 사용자에게 어떻게 보여주는지를 의미한다.

한 개의 Phase 안에서도 여러 Screen을 순차적으로 사용할 수 있다.

예:

```text
NIGHT_ACTION
→ HANDOFF
→ PRIVATE_INFO 또는 TARGET_SELECTION
→ 다음 플레이어 HANDOFF
```

### 20.3 GameScreen

V1 화면 종류는 다음 정도로 둔다.

```text
GameScreen
├─ HANDOFF
├─ NIGHT_ROLE_GUIDE
├─ PRIVATE_INFO
├─ TARGET_SELECTION
├─ PRIVATE_RESULT
├─ PUBLIC_RESULT
├─ DISCUSSION
├─ VOTE
└─ GAME_RESULT
```

의미:

- `HANDOFF`
  - 다음 플레이어에게 휴대폰을 넘기기 위한 UI 전용 화면
  - GameState를 변경하지 않는다.
- `NIGHT_ROLE_GUIDE`
  - 매일 밤 `NIGHT_ACTION`에서 Hand-off 직후 현재 플레이어의 직업과 능력을 개인 안내
  - 별도 GamePhase가 아니라 UI 화면/Private step이다.
- `PRIVATE_INFO`
  - 사용 가능한 능력 없음, 협박으로 투표 불가 등 개인 상태 확인
- `TARGET_SELECTION`
  - 밤 능력 대상, 테러리스트 동반 대상 등 플레이어 선택
- `PRIVATE_RESULT`
  - 경찰/탐정/정보원 조사 결과, 군인 방어 성공 등 개인 결과 확인
- `PUBLIC_RESULT`
  - 밤 사망자, 의사 치료 성공, 군인 공개, 기자/장의사 공개 등 전체 공개 결과
- `DISCUSSION`
  - 낮 토론 진행 및 토론 종료 입력
- `VOTE`
  - 생존 플레이어의 투표 입력
- `GAME_RESULT`
  - 승리 또는 무승부 결과 표시

버튼 확인, 다이얼로그, 최종 선택 재확인처럼 규칙을 바꾸지 않는 UI 요소는 별도 `GamePhase`나 `PendingAction`으로 만들지 않는다.

### 20.4 GameUiState

초기 구현은 거대한 nullable 필드 하나보다 화면별 Content를 분리한다.

```kotlin
data class GameUiState(
    val day: Int,
    val phase: GamePhase,
    val screen: GameScreen,
    val players: List<PlayerUi>,
    val content: ScreenContent
)
```

```kotlin
sealed interface ScreenContent {

    data class Handoff(
        val nextPlayerId: Int
    ) : ScreenContent

    data class NightRoleGuide(
        val actorId: Int,
        val job: JobType,
        val team: Team
    ) : ScreenContent

    data class PrivateInfo(
        val actorId: Int,
        val type: PrivateInfoType
    ) : ScreenContent

    data class TargetSelection(
        val actorId: Int,
        val purpose: TargetSelectionPurpose,
        val targets: List<TargetOptionUi>,
        val canSkip: Boolean
    ) : ScreenContent

    data class PrivateResult(
        val actorId: Int,
        val results: List<PrivateResultUi>
    ) : ScreenContent

    data class PublicResult(
        val events: List<PublicEventUi>
    ) : ScreenContent

    data class Discussion(
        val day: Int
    ) : ScreenContent

    data class Vote(
        val voterId: Int,
        val targets: List<TargetOptionUi>,
        val canAbstain: Boolean
    ) : ScreenContent

    data class GameResult(
        val result: GameResultUi
    ) : ScreenContent
}
```

정확한 UI 문자열, 아이콘, 색상 등은 Compose/ViewModel 계층에서 결정하고 core에는 저장하지 않는다.

### 20.5 PlayerUi / TargetOptionUi

UI에는 전체 `PlayerState`를 그대로 전달하지 않는다.

현재 화면을 보는 플레이어에게 허용된 정보만 전달한다.

```kotlin
data class PlayerUi(
    val id: Int,
    val name: String,
    val isAlive: Boolean,
    val visibleJob: JobType? = null,
    val visibleTeam: Team? = null,
    val publicMarkers: List<PlayerMarkerUi> = emptyList()
)
```

대상 선택은 별도 Projection을 사용한다.

```kotlin
data class TargetOptionUi(
    val playerId: Int,
    val name: String,
    val isAlive: Boolean,
    val enabled: Boolean,
    val visibleJob: JobType? = null,
    val visibleTeam: Team? = null,
    val markers: List<PlayerMarkerUi> = emptyList()
)
```

예를 들어 다음과 같은 이미 알려진 사실은 UI에 표시할 수 있다.

- 모두에게 공개된 군인/정치인 직업
- 기자 또는 장의사에 의해 공개된 직업
- 현재 플레이어가 개인 조사로 확인한 정보
- 마피아 및 접선된 보조직업 사이에서 서로 알고 있는 정보

단, 정보가 보인다는 이유로 전략적 선택을 자동 제한하지 않는다. 실제 선택 가능 여부는 게임 규칙상 가능한지에 대해서만 결정한다.

### 20.6 Phase 전환 골격

큰 흐름은 다음으로 확정한다.

```text
게임 시작 / 직업 확인
↓
밤 행동 입력
↓
밤 행동 Resolve
↓
개인 결과 확인
↓
낮 공개 결과
↓
토론
↓
투표
↓
추방 및 후속 처리
↓
승리 판정
↓
다음 밤 또는 GAME_RESULT
```

테러리스트처럼 추방 후 추가 입력이 필요한 경우에는 승리 판정을 잠시 보류한다.

```text
투표
→ 테러리스트 추방
→ TARGET_SELECTION
→ 동반 사망 처리
→ 승리 판정
```

직업 간 구체적인 처리 우선순위와 상호작용은 실제 Resolver 구현 단계에서 작성한다.

---

## 21. Repository / 모듈 / 패키지 구조

### 21.1 Repository

새 Android 애플리케이션은 기존 알고리즘 Repository와 분리한다.

```text
Repository: MafiaGameManager_android
```

기존 `MafiaGameAlgorithm_java`는 알고리즘/참고 구현으로 유지하고, 새 Repository가 Android 앱의 실제 개발 저장소가 된다.

새 Repository가 생성된 이후에는 버전 관리되는 `docs/ANDROID_APP_DESIGN.md`를 구현 설계의 기준 문서로 사용하고, 현재 Google Drive 문서는 초기 설계 원본/백업 역할로 둔다.

### 21.2 최상위 구조

```text
MafiaGameManager_android/
├─ README.md
├─ AGENTS.md
├─ .gitignore
├─ settings.gradle.kts
├─ build.gradle.kts
├─ gradle.properties
├─ docs/
│  └─ ANDROID_APP_DESIGN.md
├─ core/
└─ app/
```

Gradle Wrapper 및 IDE 생성 파일은 Android Studio 프로젝트 생성 시 추가한다.

### 21.3 모듈 책임

```text
:core
→ Android에 의존하지 않는 순수 JVM 게임 엔진
→ 게임 규칙, 상태, 직업, 행동, Resolver, 승리/무승부 판정

:app
→ Android Application
→ Kotlin + Jetpack Compose
→ Navigation / ViewModel / GameUiState Projection / UI
→ :core 의존
```

의존 방향은 단방향으로 유지한다.

```text
app → core
core -X→ app / Android SDK
```

초기 `core`는 기존 Java 게임 로직을 참고하여 Java 중심으로 구현해도 되며, 필요한 경우 순수 JVM 범위 안에서 Kotlin을 혼용할 수 있다. Java → Kotlin 전체 변환은 초기 목표로 두지 않는다.

### 21.4 core 패키지

과도하게 세분화하지 않고 다음 정도로 시작한다.

```text
core/src/main/java/com/mafiagame/manager/core/
├─ config/
├─ engine/
├─ state/
├─ player/
├─ job/
├─ action/
├─ result/
└─ resolver/
```

대략적인 책임:

```text
config     → GameConfig, GameMode, 직업 구성 규칙
engine     → GameEngine, 진행 제어
state      → GameState, NightState, DayState, PhaseProgress, GameHistory
player     → PlayerState, Team
job        → JobDefinition, JobState, JobRule, JobType
action     → PendingAction, GameEvent, NightAction, VoteRecord
result     → PrivateResult, PublicEvent, GameResult
resolver   → Night/Vote/Execution/Victory/Stalemate 등 실제 규칙 처리
```

직업별 Resolver를 처음부터 모두 분리하지 않는다. 복잡성이 실제로 생길 때만 세부 클래스로 추출한다.

### 21.5 app 패키지

```text
app/src/main/java/com/mafiagame/manager/
├─ MainActivity.kt
├─ navigation/
├─ viewmodel/
└─ ui/
   ├─ screen/
   ├─ component/
   ├─ model/
   └─ theme/
```

대략적인 책임:

```text
navigation  → 화면 이동
viewmodel   → GameEngine 호출, GameState → GameUiState Projection
ui/model    → GameUiState, ScreenContent, PlayerUi, TargetOptionUi
ui/screen   → 실제 Compose 화면
ui/component→ 공통 UI 컴포넌트
ui/theme    → Compose Theme
```

초기에는 별도 Repository/Data 계층을 만들지 않는다. 실제 영속 저장이 필요해질 때 추가한다.

### 21.6 Git 운용

개인 개발 프로젝트이므로 복잡한 Git Flow는 사용하지 않는다.

```text
main
→ 항상 실행 가능한 기준 브랜치

feat/*, fix/*
→ 기능 단위 작업이 필요할 때만 생성
```

Codex 작업도 필요하면 feature branch에서 진행한 뒤 `main`으로 합치는 정도로 유지한다.

---

## 22. Codex 인계 기준

Codex가 구현을 시작할 때 우선 읽어야 할 파일:

```text
AGENTS.md
README.md
docs/ANDROID_APP_DESIGN.md
```

`AGENTS.md`에는 다음 원칙을 간단히 고정한다.

```text
- docs/ANDROID_APP_DESIGN.md가 게임 규칙/구조의 기준이다.
- core는 Android에 의존하지 않는다.
- UI는 게임 규칙을 계산하지 않는다.
- GameEngine / Resolver가 Source of Truth다.
- 플레이어의 전략적으로 불리한 선택을 과도하게 막지 않는다.
- 모든 능동 능력은 기본적으로 skip 가능하며 테러리스트 보복만 필수다.
- 자동 무승부는 문서에 명시된 최종 1:1 영구 교착에만 적용한다.
- 아직 문서에서 구현 단계로 미룬 직업 간 세부 상호작용은 구현 시 테스트와 함께 결정한다.
```

초기 구현 권장 순서:

```text
1. Gradle / :core / :app 프로젝트 뼈대
2. core 기본 enum/data model
3. GameState / PendingAction / GameEvent
4. GameEngine 진행 골격
5. 일반 시민 → 마피아 → 경찰 → 의사
6. 보조직업 및 특수직업
7. 간첩 모드
8. VictoryResolver / 1:1 StalemateResolver
9. GameUiState Projection
10. Compose 화면 연결
```

실제 코드 구현 전 설계 단계에서 필요한 핵심 결정은 여기까지 완료된 것으로 본다.

---

## 23. 최종 점검에서 확정된 보완 사항

- `GamePhase`는 enum으로 관리하고, 밤 행동은 별도 `ROLE_CONFIRMATION` Phase 없이 `NIGHT_ACTION` 하나로 통일한다. Hand-off 직후 매일 밤 `NIGHT_ROLE_GUIDE` 화면에서 현재 직업/능력을 안내한다. 사용자 입력 없이 즉시 계산되는 내부 Resolve 과정은 `advanceUntilInputRequired()` 안에서 처리한다.
- `GameHistory`는 제거 기록뿐 아니라 이후 화면에서 다시 참조할 `PrivateResult`와 `PublicEvent`를 누적한다.
- 밤 처리 우선순위는 `JobRule`이 아니라 Resolver 책임으로 둔다. 별도 `NightActionRequest`를 만들지 않고 `PendingAction`을 UI 입력 요청의 단일 모델로 사용한다.
- 투표는 기권 가능, 자기 투표 가능, 동률/유효표 없음이면 추방 없이 다음 밤으로 이동한다.
- 정보원 추천은 같은 밤 마피아 공격 전에 보이지 않고 그날 밤 Private 결과 확인 단계에서 마피아에게 전달한다.
- 간첩은 자신이 포섭한 대상을 알지만 포섭 대상은 간첩 정체를 알 수 없고, 포섭된 간첩팀 플레이어끼리도 서로를 모른다. 포섭 성공 시 대상 신원은 숨기고 성공 사실만 전체에 익명 공지한다.
- 마피아는 자기 자신도 지명/최종 공격 대상으로 선택할 수 있다.
- `HANDOFF`는 UI 전용이며 다음 생존 플레이어의 순서 안내만 표시한다. 사망자는 순서에서 제외하며 비밀 정보는 Hand-off 화면에 노출하지 않는다.
- `day = 1`의 밤을 첫날 밤으로 정의하고 같은 번호의 낮이 끝난 뒤 `day`를 증가시킨다.
- 밤 처리 후 개인 결과 확인은 결과 유무와 관계없이 모든 생존 플레이어를 순회한다. 결과가 없으면 결과 없음 안내를 표시한다.
- `canVoteToday`는 해당 낮에만 유효한 임시 상태로 매 낮 시작 시 다시 계산한다.
- 간첩 포섭 성공 `PublicEvent`에는 대상 `targetId`를 포함하지 않고 익명 성공 사실만 저장한다.
- 자동 무승부는 최종 생존자가 정확히 2명인 1:1에서 문서에 명시한 영구 교착 조합에만 적용한다. 그 외 다인 동률/복잡한 교착은 자동 판정하지 않고 필요 시 수동 무승부로 종료한다.
- Repository 이름은 `MafiaGameManager_android`, 모듈은 `:core` + `:app`으로 확정한다. `app → core` 단방향 의존을 유지한다.
- Codex 구현 시작 시 `AGENTS.md`, `README.md`, `docs/ANDROID_APP_DESIGN.md`를 우선 컨텍스트로 사용한다.
