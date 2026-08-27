# PageFinder


<p align="center">
  <img src="./gifs/gameplay.gif" alt="Gameplay">
  <img src="./gifs/Reward.gif" alt="Gameplay">
  <img src="./gifs/Diary.gif" alt="Gameplay">
</p>

`PageFinder`는 3D 캐주얼 액션 게임으로, 플레이어의 행동을 통해 전장에 '잉크'를 남기고, 축적된 잉크를 활용해 다양한 특수 효과를 발동시켜 주어진 스테이지를 클리어하는 게임입니다.

<br>

## Project Overview

| 항목 | 내용 |
|---|---|
| 개발 기간 | 2024.03 ~ 2025.08 |
| 개발 인원 | 6명 팀 프로젝트(기획1, 프로그래머2, 그래픽3)|
| 플랫폼 | PC |
| 담당 역할 | Player, Reward, Growth, HUD|

<br>

## Tech Stack

- **Engine**: Unity 6(6000.0.34f1)
- **Language**: C#

<br>

## My Contributions

### 1. Player / Combat
- 플레이어 이동 및 연속 공격, 대쉬, 스킬, 피격 로직 구현
- 우선순위 기반 입력 처리로 이동, 공격, 대쉬 흐름 제어

### 2. Ability System
- 강화 단계에 따라 동작이 변경되는 Ability System 구현
- CSV 기반 Ability 데이터 관리를 통한 로직과 데이터 분리

### 3. Buff / Shield System
- 버프 적용 및 해제 흐름 구현
- Shield 상태 변경을 Event 기반으로 HUD와 연동

### 4. Ink System
- 플레이어 행동에 따라 전장에 잉크가 생성되는 흐름 구현
- 서로 다른 Ink간 합성 가능 여부를 판정하는 조건 시스템 설계

### 5. HUD / UI
- 플레이어 상태 변화에 따른 HUD 갱신
- 재사용 가능한 UI 컴포넌트 구성

### 6. 강화 카드 장착/해제 UI 및 데이터 관리 시스템 구현
- Drag &  Drop 기반 카드 장착/해제 구현
- 슬롯 상태와 장착 데이터 동기화
- 카드 데이터와 UI 표현 분리

<br>

## Architecture

```text
Input
  ↓
PlayerInputInvoker
  ↓
PlayerController
  ↓
ScriptBehaviour
  ↓
Damage / Generate Ink
  ↓
Enemy
  ↓
Reward
  ↓
Growth / UI
```

<br>

### 주요 클래스

| 클래스 | 역할 |
|---|---|
| [`PlayerInputInvoker.cs`](./PageFinder/Assets/02.Scripts/Entity/Player/PlayerInputInvoker.cs) | 플레이어 입력을 받고 우선순위에 따라 행동 Command 실행 요청 처리 |
| [`ScriptSystemManager.cs`](./PageFinder/Assets/02.Scripts/Script/ScriptSystemManager.cs) | 보상(Script) 데이터 파싱 및 생성, UI 연동 흐름 관리|
| [`EventManager.cs`](./PageFinder/Assets/02.Scripts/Mananger/EventManager.cs) | Observer 기반 이벤트 구독•해제 및 이벤트 전달 관리 |
| [`GeometryUtils.cs`](./PageFinder/Assets/02.Scripts/Utils/GeometryUtils.cs) | 잉크 합성 조건 판별 |
| [`Buff.cs`](./PageFinder/Assets/02.Scripts/Buff/Buffs.cs), [`CommandInvoker.cs`](./PageFinder/Assets/02.Scripts/Buff/CommandInvoker.cs) | Buff의 등록 및 실행 흐름 관리|
| [`UIManager.cs`](./PageFinder/Assets/02.Scripts/UI/UIManager/NewUIManager.cs) | UI Panel 등록•전환 및 관리, 게임 이벤트에 따른 UI 상태 전환 |

<br>

## Technical Highlights

### 1. Strategy 패턴 기반 Ability System 설계

#### 문제
초기에는 기본 공격, 대쉬, 스킬의 실행 로직을 각각의 Controller에서 직접 처리했습니다. 
그러나 기능이 확장되면서 Controller 내부의 상태 검사와 실행 로직이 증가했고, 강화 시스템이 추가된 이후에는 수치 변화뿐 아니라 Ability의 실행 방식 자체가 변경되는 요구사항도 발생했습니다.
이 구조를 유지할 경우 새로운 Ability나 강화 행동을 추가할 때마다 기존 Controller의 조건 분기와 실행 로직을 수정해야 해, 시스템 간 결합도와 유지보수 비용이 증가하는 문제가 있었습니다.

#### 접근
Player Controller가 구체적인 Ability 실행 방식을 직접 처리하지 않고, 현재 장착된 
[`BaseScript`](./PageFinder/Assets/02.Scripts/Script/BaseScript.cs)에 행동 실행을 위임하도록 구조를 변경했습니다.
BaseScript 내부에서는 Ability의 데이터와 실행 로직을 분리하고, 실행 행동을 [`IScriptBehaviour`](./PageFinder/Assets/02.Scripts/Script/IScriptBehaviour.cs) 인터페이스로 추상화했습니다. [`DashBehaviour`](./PageFinder/Assets/02.Scripts/Script/DashBehaviour.cs)와 같은 개별 Behaviour가 실제 행동을 담당하도록 구성하여, Controller는 구체적인 구현을 알지 않고 동일한 인터페이스를 통해 Ability를 실행할 수 있도록 했습니다.

```text
Player Controller
        ↓ 실행 요청
    BaseScript
        ↓ 행동 위임
 IScriptBehaviour
   ├─ AttackBehaviour
   ├─ DashBehaviour
   └─ SkillBehaviour
```

또한 플레이 중 보상 및 강화에 따라 Controller가 참조하는 `BaseScript`를 교체할 수 있도록 구성하여, Player 로직을 수정하지 않고 현재 사용 중인 Ability를 동적으로 변경할 수 있도록 했습니다.

#### 결과

- 신규 Ability 및 행동 추가 시 기존 Player Controller 수정 최소화
- 강화에 따른 행동 변경을 조건 분기 증가 없이 독립적으로 처리
- Ability별 변경 사항의 영향 범위를 개별 Behaviour 내부로 제한

<br>

### 2. 우선순위 기반 Input/Command 처리

#### 문제

공격, 대쉬, 스킬과 같은 여러 행동 입력 짧은 시간 안에 동시에 발생할 수 있고, 플레이어의 현재 상태에 따라 입력 시점에 즉시 실행할 수 없는 경우가 있었습니다.
각 입력 콜백에서 행동을 바로 실행할 경우 행동 간 우선순위를 일관되게 제어하기 어렵고, 실행 불가능한 순간에 입력된 조작이 그대로 소실될 수 있었습니다.

#### 접근

각 플레이어 입력을 [`InputCommand`](./PageFinder/Assets/02.Scripts/Buff/Command.cs) 객체로 캡슐화하고, 입력 발생 시 즉시 행동을 실행하지 않고 [`PlayerInputInvoker`](./PageFinder/Assets/02.Scripts/Entity/Player/PlayerInputInvoker.cs)에 전달하도록 설계했습니다.
각 Command는 실행 가능 여부(`IsExecutable`), 우선순위(`Priority`), 입력 시점(`Timestamp`), 입력 유효 시간(`ExpirationTimeSec`)을 가지며, [`PlayerInputInvoker`](./PageFinder/Assets/02.Scripts/Entity/Player/PlayerInputInvoker.cs)가 매 프레임 다음 조건을 검사하도록 구현했습니다.

```text
Input Action 
      ↓ 
InputCommand 생성 
      ↓ 
PlayerInputInvoker 
      ↓ 
유효 시간 검사 
      ↓ 
실행 가능 여부 검사 
      ↓ 
우선순위 비교 
      ↓ 
최우선 Command 실행
```

실행 가능한 입력이 여러 개 존재할 경우 우선순위가 가장 높은 Command를 선택하며, 현재 실행할 수 없는 입력은 설정된 유효 시간 동안 유지하여 이후 실행 가능 여부를 다시 검사하도록 설계했습니다.

#### 결과

- 공격, 스킬, 대쉬 입력간 실행 우선순위를 일관된 규칙으로 관리
- 행동 불가능 시점에 입력된 조작을 일정 시간 보관하여 입력 누락 완화
- 입력 감지와 실제 Player 행동 실행 책임을 Command 단위로 분리
- 신규 Player Action 추가 시 독립적인 Command 구현을 통해 기존 입력 처리 로직의 변경 최소화

<br>

## Links

- [Gameplay Video](https://www.youtube.com/watch?v=APIvWz_owCs)
- [Original Team Repository](https://github.com/marshmar/PageFinder)

