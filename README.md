# No-Face

쿼터뷰 로그라이크 RPG 게임  
절차적 콘텐츠 생성 알고리즘(PCG)을 활용하여 던전을 자동 생성하고, 플레이어가 탐험하는 형식의 게임입니다.

<br>

## 프로젝트 개요

- 프로젝트명: No-Face  
- 개발 기간: 2024.09.06 ~ 2024.12.04  
- 개발 인원: 4명  
- 엔진: Unreal Engine 5.4  
- 플랫폼: PC (Windows)  

<br>

## 게임 소개

NoFace는 실행 시마다 절차적으로 생성된 던전을 탐험하며 몬스터를 처치하고, 보스를 물리쳐 클리어하는 로그라이크 RPG입니다.  
게임이 끝난 후 재시작하면 매번 다른 구조의 맵이 생성됩니다.

<br>

PCG(Procedural Content Generation)는 일련의 규칙에 따라 콘텐츠를 자동 생성하는 알고리즘으로,  
맵 지형뿐 아니라 적, 스토리 등 다양한 콘텐츠 생성에 응용됩니다.

<br>

### 조작 방법

<table>
  <tr>
    <th>키보드</th>
    <td>Q, W, E, R (스킬)</td>
    <td>Z, X (무기 교체)</td>
    <td>Tab (월드맵)</td>
    <td>Space Bar (대쉬)</td>
    <td>T (스킬 강화창)</td>
  </tr>
  <tr>
    <th>마우스</th>
    <td>좌 클릭 (기본 공격)</td>
    <td>우 클릭 (이동)</td>
    <td>마우스 휠 (줌 인 / 아웃)</td>
    <td></td>
    <td></td>
  </tr>
</table>

<br>


# 아키텍처 요약
<h4> 코어 클래스 </h4>

| 클래스                                | 상속/구현                                                                                                               | 핵심 책임                                                        | 대표 메서드                                                                                                                                                                                 |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ACharacterBase`                   | `ACharacter`                                                                                                        | 입력 바인딩, 이동/회전/줌, 기본공격 트리거, 스킬 트리거, 무기 전환/장착, 피해/상태 처리, UI 토글 | `SetupPlayerInputComponent`, `OnAttackStart`, `Q/W/E/R_Skill`, `Next/PrevWeapon`, `ChangeWeapon`, `EquipSword/Bow/Staff`, `TakeDamage`, `CancelCasting`, `RotateToTarget/UpdateRotate` |
| `UCharacterDefaultAttackComponent` | `UActorComponent` + `ISwordInterface`, `IBowInterface`, `IStaffInterface`                                           | 무기별 **기본 공격**(검/활/스태프), 콤보/투사체/애님 제어, 무기전환 가드                | `BeginAttack`, `SetWeaponType`, `SwordDefaultAttackHitCheck`, `SetBow`, `StartAnimation/EndAnimation`, `StaffDefaultAttack`, `CanChangeWeapon`                                         |
| `USkillComponent`                  | `UActorComponent` + `ISwordSkillInterface`, `IBowSkillInterface`, `IStaffSkillInterface`, `IPlayerSkillUIInterface` | **Q/W/E/R 스킬**, 캐스팅 큐, 쿨다운+HUD 갱신, 모션워핑, 패링/실드, 스킬 레벨업       | `PlaySkill_Q/W/E/R`, `BeginDash`, `SetWeaponType`, `SetupSkillUIWidget`, `Get/SetCastingFlag`, `GetSkillState`, `StartCooldown`, `ParryingSuccess`, `SetShieldAmount`                  |



<br>

## 캐릭터/전투/스킬 시스템 요약 (UE5/C++)

본 문서는 다음 3개 클래스군의 역할/흐름/상태/확장 포인트를 기술합니다.

- **ACharacterBase** — 입력·이동·무기 전환·피해 처리·HUD 연동
- **UCharacterDefaultAttackComponent** — 기본 공격(검/활/지팡이), 콤보/투사체
- **USkillComponent** — Q/W/E/R 스킬, 캐스팅/쿨다운/UI, 모션 워핑/패링/실드

> 엔진: Unreal Engine 5.4 / 입력: Enhanced Input


<br>

## 1) 추가 자료형

### 열거형



```
enum class EWeaponType { Sword = 0, Bow = 1, Staff = 2 };
enum class EPlayerStateType { Common, Shield, Stun, Dead };
enum class ESkillState { Progress, CanSkill }; // 스킬 실행 중/가능
```



### 델리게이트 / 이벤트



```
DECLARE_DELEGATE(FTakeItemDelegate); // 무기 장착용
FSignedChangeWeapon(int32 CurrentWeaponIndex); // 무기 변경 브로드캐스트
FOnWarpNextMap(const FVector&); // 맵 이동 신호
FParryingSign, FShieldSign; // 패링/실드 신호(스킬 컴포넌트→캐릭터)
```



### 보조 구조체

- `FTakeItemDelegateWrapper` — 무기별 장착 함수 래핑

<br>

## 2) 클래스별 역할

```markdown
ACharacterBase 
├─ 입력 바인딩(Enhanced Input) 
├─ 이동/회전/줌/대시 
├─ 공격 입력 분배 → UCharacterDefaultAttackComponent 
├─ 스킬 입력 분배 → USkillComponent 
├─ 무기 전환/장착(검/활/지팡이), 애님 인덱스/브로드캐스트 
├─ 피해 처리(패링/실드 상태 반영) 
└─ HUD/스킬UI/월드맵 생성/토글 

UCharacterDefaultAttackComponent 
├─ 현재 무기 타입 반영(SetWeaponType) 
├─ 기본 공격 BeginAttack() 멀티 디스패치 
│ ├─ 검: 콤보 타이머 기반 섹션 점프, 부채꼴 판정 
│ ├─ 활: 활 인터페이스/화살 스폰, 당기기/놓기 애니 
│ └─ 지팡이: 콤보 + 투사체 스폰 
└─ 무기 전환 가능 여부 가드(bCanChangeWeapon) 

USkillComponent 
├─ PlaySkill_Q/W/E/R → 무기별 Begin* 라우팅 
├─ 캐스팅 플래그/큐(TQueue>) 
├─ 쿨다운/가능 여부 플래그 & 타이머 
├─ UI 연동(UHUDWidget*: Start/Update/MaxCooldown) 
├─ 모션 워핑(Target 명시적 세팅) 
├─ 패링 성공 처리/실드 누적·임계 처리 
└─ 스킬 레벨업(StatData 반영)
```

<br>

## 3) 입력 바인딩 & 흐름

- **바인딩 위치**: `ACharacterBase::SetupPlayerInputComponent`
- **우클릭 이동**: `OnClickStart/OnClicking/OnRelease`
- **좌클릭 공격**: `OnAttackStart`
- **Q/W/E/R**: `Q_Skill/W_Skill/E_Skill/R_Skill` → `USkillComponent::PlaySkill_*`
- **무기 전환**: `NextWeapon/PrevWeapon`
- **대시**: `Dash` → `USkillComponent::BeginDash`
- **줌**: `ZoomInOut`
- **캐스팅 취소**: `CancelCasting`
- **UI 토글**: `DisplaySkillUI/DisplayWorldmap`

#### 이동/회전

- 우클릭 유지: 커서 지점 히트 갱신 + `AddMovementInput`
- 우클릭 해제: `SimpleMoveToLocation(네비)`
- 회전: `RotateToTarget()` 타이머 → `UpdateRotate()`에서 Yaw만 RInterpTo

<br>

## 4) 기본 공격 파이프라인

- 검: 4콤보 부채꼴 공격  
- 활: 단일 발사, 화살 스폰  
- 지팡이: 3콤보, 관통 공격

<br>

각 무기에는 4개의 스킬(QWER)이 존재하며, 스킬마다 쿨타임이 별도로 설정되어 있습니다.  
다른 무기로 전환하면 해당 무기의 스킬을 즉시 사용할 수 있도록 독립 관리됩니다.

<br>

### 입력 진입점

- `ACharacterBase::OnAttackStart()`
- `AStoryBook::CanReadBook()`이면 중단
- 타겟 트레이스 실패/스킬 실행 중(`SkillState==Progress`)이면 중단
- 캐스팅 확정: `GetCastingFlag()==true`이면 큐에서 `TFunction` 디큐 실행 후 리턴
- 아니면 회전/정지 후 `AttackComponent->BeginAttack()`

### 멀티 디스패치

- `UCharacterDefaultAttackComponent::BeginAttack()`
- `CurrentWeaponType`에 따라
- `BeginSwordDefaultAttack()`
- `BeginBowDefaultAttack()`
- `BeginStaffDefaultAttack()`

#### 검/지팡이 콤보

- 타임 윈도우: `*ComboData->EffectiveFrameCount / FrameRate`
- 윈도우 내 입력 시 `Montage_JumpToSection(NextSection)`
- 종료 시 `MovementMode=Walking`, `bCanChangeWeapon=true`

#### 활 기본 공격

- `SetBow()`로 활 인스턴스 주입(인터페이스)
- `StartAnimation()`에서 화살 스폰(+당기기 애니), `EndAnimation()`에서 초기화·발사

<br>

## 5) 스킬 시스템

스킬은 즉발형과 캐스팅형으로 나뉩니다.  
즉발형은 입력과 동시에 즉시 발동되며, 캐스팅형은 일정 시간의 준비 과정을 거친 후 실행됩니다.

<br>

캐스팅형 스킬은 **이벤트 큐 패턴**을 기반으로 구현되었습니다.  
스킬 사용 요청은 큐에 저장되고, 유저가 공격 명령을 내리면 큐에 있는 스킬이 실행됩니다.  
이 구조는 스킬 사용 중 다른 액션이 개입되지 않도록 안정적으로 관리합니다.

### 공통 흐름

- `ACharacterBase::Q/W/E/R_Skill()` → 회전/정지 → `USkillComponent::PlaySkill_*()`
- `USkillComponent::Begin_()` 내부 규칙
- 가드: `bCanUseSkill_*` & `CurrentSkillState != Progress`
- 쿨다운 시작: `StartCooldown(...)` → HUD에 최대/시작/업데이트 위임
- `CurrentSkillState = Progress`, `bCanChangeWeapon=false`
- 몽타주 재생, 끝나면 상태 복구

### 캐스팅 스킬 (이벤트 큐)

- 대상 지정이 필요한 스킬: 활 W, 스태프 Q/W
- 첫 호출에서 `bCasting=false`면:
- `bCasting=true` 설정
- 자기 자신을 람다로 `SkillQueue.Enqueue([this]{ Begin*(); })`
- (틱에서) 커서 위치 Cursor 지속 갱신/프리뷰
- 확정(좌클릭): `OnAttackStart()`에서 `Dequeue()` → `Begin*()`가 **캐스팅 분기(bCasting==true)**로 실행
- 취소: `ACharacterBase::CancelCasting()` — 플래그 해제, 큐 Pop(), 몽타주/이동/전환 상태 복구

> **포인트:** 입력(확정/취소)과 실행을 큐로 분리해 레이스/인터럽트에 강함.

### 모션 워핑

- 스킬별 타깃 좌표를 직접 지정 후 실행
- 예: 검 R → "SwordR", 활 Q/R → "BowQ" / "BowR", 지팡이 W → "StaffW"
- 종료 시 `RemoveAllWarpTargets()`로 정리

### 패링/실드

- **패링(검 E)**
- 시작 시 `ParryingSign.ExecuteIfBound()`(토글 개념)
- 성공 시: 적 스턴, 방어 이펙트, 추가 공격 몽타주 재생 중 Capsule 프로파일 "Dodge" 적용 → 종료 시 "Player"로 원복
- **실드(스태프 E)**
- `ShieldThreshold`는 스킬 레벨 반영
- 실드 이펙트 활성/종료 파티클 처리
- 피해 누적은 캐릭터 `TakeDamage()`에서 임계 초과분만 HP에 관통

<br>

## 6) 무기 전환 & 장착 파이프라인

플레이어는 검, 활, 지팡이 3종의 무기를 사용할 수 있습니다.  
Z/X 또는 마우스 휠을 통해 무기를 실시간 전환할 수 있으며, 무기마다 전용 기본 공격 및 스킬 구성이 다릅니다.

<br>

- 입력: `NextWeapon/PrevWeapon`
- 가드: `AttackComponent->CanChangeWeapon()` && `SkillComponent->CanChangeWeapon()`
- 인덱스 래핑 후 `ChangeWeapon()`
- 타입 동기화: `SkillComponent/AttackComponent->SetWeaponType`
- 델리게이트 배열로 무기별 `Equip*()` 호출
- 애님 인스턴스 WeaponIndex 반영 + `SignedChangeWeapon.Broadcast()`

#### 장착 구현

- 이전 무기 `Destroy()` → 새 무기 `SpawnActor` → 소켓 부착
- 무기별 이동 속도/사운드 적용
- 활은 `IBowInterface`로 공격 컴포넌트에 활 객체 주입

<br>

## 7) 피해/상태 처리

- `ACharacterBase::TakeDamage()`
- 패링 On: `ParryingSuccess()` 호출, 0 데미지
- Common: 그대로 HP 차감
- Shield: 누적(`SetShieldAmount`) → 임계(`GetShieldThreshould`) 비교
- 임계 이하: 누적만 증가 (반환값은 입력 데미지 그대로 반환)
- 임계 초과: 초과분만 HP 적용, 실드량 리셋, 상태 Common 복귀

<br>

## 8) HUD/스킬 UI 연동

- 캐릭터 → `SetupHUDWidget(UHUDWidget*)`에서 **IPlayerSkillUIInterface**로 스킬 UI 연결
- 스킬 시작 시 `USkillComponent::StartCooldown(...)`이 HUD에 3가지 호출
- `Widget->SetMaxCooldown(Duration, Weapon, Slot)`
- `Widget->StartCooldown(Weapon, Slot)`
- `Widget->UpdateCooldownBar(Duration, TimerHandle, bCanUseFlag, Slot, Weapon, AccumTimer)`
- 현재 구현 기준: 슬롯별 ProgressBar만 표시/숨김/퍼센트 갱신(심플)

<br>

## 9) 코드 포인트

### 캐스팅 확정 우선 처리

```
// ACharacterBase::OnAttackStart()
if (SkillComponent->GetCastingFlag()) {
    TFunction SkillAction;
    if (SkillComponent->SkillQueue.Dequeue(SkillAction))
    {
        RotateToTarget();
        SkillAction(); // 확정 실행
        return;
    }
}
```

### 쿨다운 시작 → UI 위임

```
void USkillComponent::StartCooldown(float Duration, FTimerHandle& Handle,
    bool& bCanUse, ESkillType Slot, int32 Weapon, float& Timer)
{
    bCanUse = false;
    Widget->SetMaxCooldown(Duration, CurrentWeaponType, Slot);
    Widget->StartCooldown(CurrentWeaponType, Slot);
    Widget->UpdateCooldownBar(Duration, Handle, bCanUse, Slot, Weapon, Timer);
}
```

<br>

## 10) API 요약

### ACharacterBase

- `NextWeapon()/PrevWeapon()`, `ChangeWeapon()`
- `OnAttackStart()`, `Q/W/E/R_Skill()`, `CancelCasting()`
- `TakeDamage(...)`, `GetWeaponType()`, `GetPlayerState()`
- `DisplaySkillUI()/DisplayWorldmap()`

### UCharacterDefaultAttackComponent

- `BeginAttack()`, `SetWeaponType(int32)`
- (검) `SwordDefaultAttackHitCheck()`
- (활) `SetBow(ABow*)`, `StartAnimation()`, `EndAnimation()`
- (지팡이) `StaffDefaultAttack()`
- `bool CanChangeWeapon()`

### USkillComponent

- `PlaySkill_Q/W/E/R()`, `BeginDash()`
- `bool GetCastingFlag()/SetCastingFlag(bool)`
- `bool CanChangeWeapon()/SetCanChangeWeapon(bool)`
- `ESkillState& GetSkillState()`
- `SetWeaponType(int32)`, `SetupSkillUIWidget(UHUDWidget*)`
- `UsePlayerSkillPoint(...)`, `GetSkillUpgradeLevel(...)`, `PlusSkillPoint()`

<br>

## 11) 흐름 요약 다이어그램

### A) 공격/캐스팅 확정

```markdown
[좌클릭]
  ├─ StoryBook? → return
  ├─ Trace 실패 or SkillState==Progress? → return
  ├─ CastingFlag==true?
  │     └─ SkillQueue.Dequeue() → RotateToTarget → 실행 → return
  └─ RotateToTarget → OnClickStart(Stop) → AttackComponent.BeginAttack()
```

### B) 캐스팅 스킬(예: Staff_Q / Bow_W)

```markdown
Q/W 입력
  ├─ bCasting==false → bCasting=true → SkillQueue.Enqueue(this:Begin*)
  └─ (확정: 좌클릭) OnAttackStart → Dequeue → Begin* (캐스팅 분기)
       ├─ 쿨다운 시작, 상태 Progress, 무기전환 금지
       └─ 몽타주/이펙트/워핑 → End*에서 상태 복구
 (취소) CancelCasting → bCasting=false, Queue.Pop(), 몽타주 정지 등 복구
```

<br>



## 관련 링크

- 플레이 영상: https://drive.google.com/file/d/1zY7l_9YJuAV5TMM1DHRHP9y_MlMIn4IE/view?usp=drive_link

