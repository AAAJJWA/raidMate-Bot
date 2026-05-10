# raidMate-Bot
# 프로젝트 개요

로스트아크 지인용 디스코드 레이드 모집 봇 제작.

목표는:

- 디스코드 내부 UI만 사용하여
- 빠르게 레이드 모집 생성 및 참여 가능하게 만들고
- 로스트아크 API를 활용하여 캐릭터 정보를 자동으로 불러오는 것.

웹 프론트엔드 및 복잡한 DB 기반 서비스 구조는 제외하며,  
개인/지인 규모에서 실제로 사용 가능한 수준의 가벼운 프로젝트를 목표로 한다.

---

# 핵심 방향

## 사용 환경

- Discord 기반
- 모바일 Discord 앱 사용 고려
- 웹 서비스 제작하지 않음
- Discord UI(Button, SelectMenu, Slash Command) 중심

---

# 기술 스택

## Backend

- Java
- JDA(Java Discord API)

## LostArk API

사용 목적:

- 원정대 캐릭터 자동 조회
- 클래스 표시
- 아이템 레벨 표시

자동 숙제 추적 및 클리어 판별 기능은 구현하지 않음.

## 저장 방식

초기:

- JSON 파일 기반 최소 저장

추후 Oracle Cloud Free VPS 이전 고려 가능하도록 구조 설계.

예:

- `CharacterRepository` 인터페이스 분리
- `JsonCharacterRepository` 구현
- 이후 `DbCharacterRepository`로 교체 가능하게 구성

---

# 주요 기능

## 1. 원정대 캐릭터 연동

사용자 명령:

```text
/연동 대표캐릭터명
```

로스트아크 API:

```http
GET /characters/{characterName}/siblings
```

조회 데이터:

- 캐릭터명
- 클래스
- 아이템 레벨

응답 예시:

```text
원정대 캐릭터 목록

- 백돌 (1720 워로드)
- 람쥐 (1710 바드)
- 고라니 (1690 브레이커)

[등록]
[취소]
```

예외 상황 UX:

- 캐릭터 없음/오타: `캐릭터를 찾을 수 없습니다. 대표 캐릭터명을 다시 확인해주세요.`
- API 장애/타임아웃: `로스트아크 API 응답이 지연되고 있습니다. 잠시 후 다시 시도해주세요.`

---

## 2. 레이드 모집 생성

Slash Command 기반 단일 입력 대신,  
Discord 버튼 + SelectMenu 기반 단계형 UI로 진행.

### 모집 생성 흐름

```text
[생성]
   ↓
레이드 선택
   ↓
난이도 선택
   ↓
모집 생성
```

### 1단계 - 생성 패널

채널 상단 고정 메시지 예시:

```text
레이드 모집 생성

[생성]
```

### 2단계 - 레이드 선택

```text
레이드 선택

- 카멘
- 에기르
- 베히모스
- 모르둠
```

`StringSelectMenu` 사용.

### 레이드 목록 관리 방식

로아 API에서 레이드 목록을 직접 제공하지 않으므로 내부 관리.

- Enum 또는 JSON 기반
- 권장 구조: `raidId`, `displayName`, `difficulty[]`, `defaultMaxMember`

예:

```java
class RaidDefinition {
    String raidId;
    String displayName;
    List<Difficulty> difficulties;
    int defaultMaxMember;
}
```

### 3단계 - 난이도 선택

```text
난이도 선택

- 노말
- 하드
```

### 4단계 - 모집 생성 완료

```text
카멘 하드 1-4

현재 인원 1/8

딜러 0
서포터 0

[참여]
[취소]
[마감]
```

---

## 3. 참여 신청

사용자가 `[참여]` 버튼 클릭 시:

### 1단계 - 캐릭터 선택

```text
- 백돌 (워로드 1720)
- 람쥐 (바드 1710)
```

### 2단계 - 역할 선택

```text
[딜러]
[서포터]
```

자동 역할 분류는 사용하지 않음.

이유:

- 바드/홀리나이트/도화가를 딜 세팅으로 운용하는 경우 존재.

---

## 4. 모집 상태 자동 갱신

예:

```text
현재 인원 5/8

딜러 3
서포터 2

1. 백돌 (워로드/딜러)
2. 람쥐 (바드/서포터)
3. 고라니 (브레이커/딜러)
```

Discord Message Edit 방식 사용.

---

## 5. 모집 마감

`[마감]` 클릭 시:

- 버튼 비활성화
- 추가 참여 차단

---

# 권한/동시성/안정성 규칙

## 권한 규칙

- 모집 생성자(방장)만 `[마감]` 가능
- 필요 시 방장만 `[취소]`(모집 삭제 성격) 가능
- 참여자는 본인 참여만 취소 가능

## 중복/경합 처리

- 동일 유저 중복 참여 금지
- 동일 유저 버튼 연타 시 1회만 반영
- 동시 클릭 충돌 방지 위해 `RaidRoom` 단위 동기화(락/직렬 처리) 적용

## 메시지 편집 실패 대응

- 모집 메시지가 삭제되었거나 권한이 없으면:
  - 상태맵 정리
  - 사용자에게 `유효하지 않은 모집글입니다.` 안내

---

# 생성 세션 구조

레이드 생성은 단계형 UI이므로 임시 세션 사용.

```text
UserId → RaidCreationSession
```

예:

```java
class RaidCreationSession {
    RaidType raidType;
    Difficulty difficulty;
    int maxMember;
    Instant createdAt;
}
```

세션 정책:

- TTL 3~5분
- 만료 시 자동 폐기
- 만료 후 버튼 입력 시 `세션이 만료되었습니다. 다시 생성해주세요.` 응답

---

# 데이터 저장 정책

## 저장 O

- Discord 사용자별 캐릭터 목록

```text
DiscordUserId
 └─ Character
      ├─ name
      ├─ class
      ├─ itemLevel
```

초기 저장 방식:

```text
characters.json
```

저장 안정성 규칙:

- 파일 직접 덮어쓰기 대신 `임시 파일 -> 원자적 교체` 방식
- 저장 실패 시 기존 파일 보존
- 앱 시작 시 JSON 파싱 실패하면 백업 파일로 복구 시도(옵션)

## 저장 X

- 숙제 기록
- 출석 기록
- 통계
- 레이드 히스토리

레이드 모집 상태는 메모리 기반 관리:

```text
Map<MessageId, RaidRoom>
```

봇 재시작 시 초기화 허용.

---

# 서버 운영 방향

## 초기 개발 단계

- 개인 PC에서 실행
- `java -jar bot.jar`
- 레이드 시간대 중심 사용

## 추후 확장 고려

Oracle Cloud Free VPS 이전 가능하도록 구조 설계.

목표:

- 봇 24시간 실행
- 항상 사용 가능 운영

현재 단계에서는 도입하지 않음:

- DB
- 웹 서버
- 프론트엔드
- 인증 시스템

---

# 아키텍처 방향

```text
Discord Event
      ↓
Interaction Handler
      ↓
RaidCreationSessionManager
      ↓
RaidService
      ├─ RaidRoom
      ├─ Participant
      ├─ CharacterService
      └─ LostArkApiClient
```

저장소:

```text
CharacterRepository
 ├─ JsonCharacterRepository
 └─ (추후) DbCharacterRepository
```

---

# 프로젝트 목표

복잡한 웹 서비스가 아닌  
**"실제로 지인들과 사용할 수 있는 디스코드 레이드 모집 도구"** 제작.

우선순위:

1. 실제 사용성
2. 모바일 Discord UX
3. 구현 완성 가능성
4. Oracle Cloud Free 확장 가능 구조
5. 유지보수 단순화
