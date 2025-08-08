# NoFace

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

## 주요 시스템

# 캐릭터 시스템 구현 요약 (UE5/C++)

## 입력 시스템 (Enhanced Input)

- **바인딩 위치:** `SetupPlayerInputComponent`
- **입력 종류:** 좌/우클릭(이동·공격), Q/W/E/R(스킬), 무기 전환(Next/Prev), 대시, 줌, 캐스팅 취소, 스킬 UI/월드맵 토글
- **리소스 로딩:** 
- 생성자에서 `ConstructorHelpers::FObjectFinder`로 `IMC_Default` 및 각 `InputAction(IA_*)` 로드
- **런타임 등록:** 
- `BeginPlay`에서 `UEnhancedInputLocalPlayerSubsystem::AddMappingContext`
- **핵심:** 
- IA 로딩 ↔ 런타임 바인딩 분리로 에셋/코드 변경 내구성 확보

---

## 이동/조준 (쿼터뷰 최적화)

### 우클릭 이동 플로우

1. **OnClickStart()** 
- 즉시 `StopMovement()`로 현재 이동 중단
2. **OnClicking()** 
- `GetHitResultUnderCursor` → `CachedLocation` 갱신 
- `(CachedLocation - ActorLoc).GetSafeNormal()`을 `AddMovementInput`으로 프레임 단위 가속
3. **OnRelease()** 
- `SimpleMoveToLocation(GetController(), CachedLocation)`로 네비 경로 이동

### 시선 회전

- **RotateToTarget()** 
- 타이머 시작(0.01s)
- **UpdateRotate()** 
- 커서 히트 지점 기준 Yaw만 `FMath::RInterpTo(속도 50)`로 보간 
- 각도 오차 < 1°면 타이머 종료

**핵심:** 
- 드래그 중 물리적 가속, 해제 후 네비 이동의 이중 모드 
- Pitch 억제로 쿼터뷰 가독성 유지

---

## 기본 전투 & 캐스팅 확정

- **기본 공격:** 
- `OnAttackStart()` → `AttackComponent->BeginAttack()`
- **가드 조건:** 
- `AStoryBook` 상호작용 중이면 리턴(`CanReadBook()`)
- `TraceAttack()==false` 또는 `SkillState==Progress`면 리턴
- **캐스팅 확정 우선:** 
- 캐스팅 중(`SkillComponent->GetCastingFlag()`): 
- `SkillQueue.Dequeue()`로 대기 중 커맨드 실행 후 리턴



cpp
// OnAttackStart() 중 핵심
if (SkillComponent->GetCastingFlag()) {
    TFunction SkillAction;
    if (SkillComponent->SkillQueue.Dequeue(SkillAction))
    {
        RotateToTarget();
        SkillAction();  // 확정 실행
        return;
    }
}




**핵심:** 
- 공격 입력과 캐스팅 확정이 겹쳐도 큐 우선 규칙으로 레이스 없이 일관 처리

---

## 스킬 시스템 (Q/W/E/R + 캐스팅 취소)

- **Q_Skill / W_Skill / E_Skill / R_Skill** 
- 공통: `RotateToTarget()` → `OnClickStart()`(즉시 정지) → `SkillComponent->PlaySkill_*()`
- **취소:** `CancelCasting()`
- `SetCastingFlag(false)`, `SkillQueue.Pop()`
- `AnimInstance->StopAllMontages(0.1f)`
- 이동 모드 `MOVE_Walking` 복귀
- `SkillComponent->SetCanChangeWeapon(true)`

**핵심:** 
- 플래그/큐/애님 3요소 동시 리셋 → 인터럽트에도 깨끗한 복구

---

## 무기 전환 & 장착 (검/활/지팡이)

- **전환 입력:** `NextWeapon()` / `PrevWeapon()`
- `AttackComponent->CanChangeWeapon()` && `SkillComponent->CanChangeWeapon()` 가드
- 인덱스 0~2 래핑 → `ChangeWeapon()`
- **ChangeWeapon():**
- `SkillComponent`/`AttackComponent`에 무기 타입 동기화
- 델리게이트 배열로 무기별 장착 함수 호출
- `AnimWeaponIndex()`로 애님 인스턴스 동기화
- `SignedChangeWeapon.Broadcast(WeaponIndex)`



cpp
void ACharacterBase::ChangeWeapon() {
    SkillComponent->SetWeaponType(WeaponIndex);
    AttackComponent->SetWeaponType(WeaponIndex);
    TakeItemDelegateArray[WeaponIndex].TakeItemDelegate.ExecuteIfBound();
    CurrentWeaponType = static_cast(WeaponIndex);
    SignedChangeWeapon.Broadcast(WeaponIndex);
}




- **장착 구현:** `EquipSword` / `EquipBow` / `EquipStaff`
- 기존 무기 `Destroy()` → 새 무기 `SpawnActor` → 소켓 `AttachToComponent`
- 무기별 `MaxWalkSpeed`/전환 사운드 적용
- 활: `IBowInterface`로 `AttackComponent`에 활 인스턴스 주입

**핵심:** 
- “전환 가능 가드 → 타입 동기화 → 델리게이트 장착 → 애님/브로드캐스트” 표준 체인으로 결합도↓ 확장성↑

---

## 피해/상태 (패링 & 실드)

- **패링:** 
- `bIsParrying` 시 피해 0, `SkillComponent->ParryingSuccess(DamageCauser)` 호출
- **Common:** 
- `UCharacterStatComponent::ApplyDamage(Damage)`
- **Shield:**
- `SetShieldAmount(Damage)` 누적 → 임계(`GetShieldThreshould()`) 비교
- 임계 이하: (현재 로직상) 피해 반환만
- 임계 초과: 초과분만 `Stat->ApplyDamage(...)`로 HP 적용 → 실드 리셋 → 상태 Common 복귀

**핵심:** 
- 실드가 누적 흡수하고 임계 초과분만 HP에 관통되도록 상태 기반 제어

---

## UI · 카메라

- **스킬 UI:** 
- `DisplaySkillUI()` — 있으면 제거, 없으면 `CreateWidget` → `AddToViewport()`
- **월드맵:** 
- `DisplayWorldmap()` — 없으면 생성 후 `HitTestInvisible`로 추가, 있으면 Hidden 후 제거
- **줌:** 
- `ZoomInOut()` — `SpringArm->TargetArmLength`를 200~2500으로 클램프

---

## 핵심 코드 포인트 (레퍼런스)

- **입력 바인딩:** `SetupPlayerInputComponent`
- **이동/조준:** `OnClickStart` / `OnClicking` / `OnRelease`, `RotateToTarget` / `UpdateRotate`
- **공격 시작/확정:** `OnAttackStart` (캐스팅 플래그 + `SkillQueue.Dequeue`)
- **스킬 실행/취소:** `Q/W/E/R_Skill`, `CancelCasting`
- **무기 전환/장착:** `Next/PrevWeapon` → `ChangeWeapon` → `EquipSword/Bow/Staff`, `AnimWeaponIndex`
- **피해 처리:** `TakeDamage`
- **UI:** `DisplaySkillUI`, `DisplayWorldmap`

---

## 핵심 구현 아이디어 (요약)

- **입력·상태·실행 분리:** 
- 캐스팅을 **플래그 + 큐(커맨드)**로 분리해 입력 경쟁 상황에도 안정적
- **무기 전환 파이프라인 표준화:** 
- 가드 → 타입 동기화 → 델리게이트 장착 → 애님/브로드캐스트로 유지보수성↑
- **쿼터뷰 UX 최적화:** 
- 드래그 가속 + 네비 이동, Yaw 보간만 적용
- **상태 기반 피해 처리:** 
- 패링/실드/일반 분리로 예외 케이스(인터럽트/상태 전환) 흡수

---

**참고:** 
- 각 항목별 상세 구현은 C++ 코드 및 UE5 Blueprint와 연동 필요 
- 확장/변경 시 각 시스템의 분리와 표준화된 파이프라인 유지 권장

---

### 스킬 시스템

스킬은 즉발형과 캐스팅형으로 나뉩니다.  
즉발형은 입력과 동시에 즉시 발동되며, 캐스팅형은 일정 시간의 준비 과정을 거친 후 실행됩니다.

<br>

캐스팅형 스킬은 **이벤트 큐 패턴**을 기반으로 구현되었습니다.  
스킬 사용 요청은 큐에 저장되고, 유저가 공격 명령을 내리면 큐에 있는 스킬이 실행됩니다.  
이 구조는 스킬 사용 중 다른 액션이 개입되지 않도록 안정적으로 관리합니다.

<br>

또한, **기본 공격과 스킬 공격은 각각 별도의 컴포넌트로 분리**되어 있습니다.  
- 기본 공격: `CharacterDefaultAttackComponent`  
- 스킬 공격: `SkillComponent`  

<br>

이를 통해 캐릭터 클래스와 전투 로직 간 결합도를 낮췄으며,  
모듈화된 설계로 유지보수와 확장이 용이합니다.

<br>

### 무기 시스템

플레이어는 검, 활, 지팡이 3종의 무기를 사용할 수 있습니다.  
Z/X 또는 마우스 휠을 통해 무기를 실시간 전환할 수 있으며, 무기마다 전용 기본 공격 및 스킬 구성이 다릅니다.

<br>

무기 교체는 인덱스를 기반으로 순환하며, `AttackComponent`, `SkillComponent`, `HUD`와 연동하여 현재 무기에 맞는 기능이 실시간 적용됩니다.

<br>

### 전투 시스템

- 검: 4콤보 부채꼴 공격  
- 활: 단일 발사, 화살 스폰  
- 지팡이: 3콤보, 관통 공격

<br>

각 무기에는 4개의 스킬(QWER)이 존재하며, 스킬마다 쿨타임이 별도로 설정되어 있습니다.  
다른 무기로 전환하면 해당 무기의 스킬을 즉시 사용할 수 있도록 독립 관리됩니다.

<br>

### 스킬 쿨타임 UI

무기별 스킬에 총 12개의 쿨타임 바가 존재합니다.  
HUD는 무기 전환 시 현재 무기에 맞는 쿨타임바만 노출하도록 구성되어 있으며, `FTimerHandle`과 `ProgressBar`를 사용해 실시간 UI 반영이 이루어집니다.

<br>

### AI 몬스터

총 6종의 몬스터가 구현되어 있으며, 모두 EnemyBase 클래스를 상속받아 제작되었습니다.  
AI Perception을 통해 Sight/Damage 기반 감지를 수행하고, 각 몬스터는 BT 기반의 개별 전투 패턴을 가집니다.

<br>

### 스테이지 시스템

- READY: 플레이어 감지 대기  
- FIGHT: 몬스터 스폰 및 문 닫힘  
- NEXT: 모든 몬스터 처치 후 문 개방

스테이지는 비트 플래그를 기반으로 문 방향과 상태를 제어합니다.

<br>

## 회고

### 배운 점

1. **Git LFS 이슈**: 블루프린트는 바이너리로 인식 되어 병합할 때 문제가 많은 점 
2. **캐스팅 스킬 구조**: 함수 객체 + 큐 활용으로 실행 흐름 구현
3. **상태 패턴 적용**: 복잡한 전투 구조에 효과적임을 체감

<br>

### 아쉬운 점

1. **불필요한 리소스 문제**: 사용하지 않은 애셋 정리가 부족했음
2. **무기 생성 방식**: 실질 기능 없는 무기 Actor 생성/제거 반복 → `StaticMeshComponent` 교체 or 오브젝트 풀링 적용 고려 필요

<br>

## 관련 링크

- 플레이 영상: https://drive.google.com/file/d/1zY7l_9YJuAV5TMM1DHRHP9y_MlMIn4IE/view?usp=drive_link
- 자세한 구현 사항: https://darkened-beryl-7c6.notion.site/No-Face-1328687479db8006870fd8a5b8a8eb3b
