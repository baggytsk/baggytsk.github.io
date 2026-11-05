# 영속성 컨텍스트와 1차 캐시

> 공부한 날: 11/4 · 출처: [rlaope/estudy — 영속성 컨텍스트](https://github.com/rlaope/estudy/blob/master/brains/Back-End/jpa/persistence/persistcontext.md)
> 관련: (예정) `spring/transactional-proxy.md` · `jpa/n-plus-one.md` · `db/isolation-level.md`

## 한눈에

**1차 캐시는 트랜잭션 하나 동안만 사는 `Map<(엔티티 타입, id), 엔티티>`다.** 같은 트랜잭션에서 같은 id로 `find`하면 DB에 가지 않고 이 Map에서 꺼낸다. 그래서 그사이 다른 트랜잭션이 DB를 바꿔도 옛 값이 보일 수 있다.

- 1차 캐시는 설정 없이 늘 켜져 있고, 트랜잭션(정확히는 영속성 컨텍스트)이 끝나면 통째로 사라진다
- `readOnly = true`는 1차 캐시를 끄지 않는다. 끄는 건 스냅샷·변경 감지·flush다
- 최신 값을 보는 문제는 두 층이다: ① JPA 1차 캐시 ② DB 격리 수준

## 꼬리 질문

- [ ] 1차 캐시는 언제 생기고 언제 사라지나? → [동작 흐름](#동작-흐름)
- [ ] `readOnly = true`는 무엇을 바꾸나? 이 버그의 원인인가? → [조건과 예외](#조건과-예외)
- [ ] 같은 트랜잭션에서 최신 값을 보려면? → [조건과 예외](#조건과-예외)
- [ ] REPEATABLE READ였어도 JPQL로 최신 값을 보나? → [조건과 예외](#조건과-예외)
- [ ] 1차 캐시와 Redis 캐시는 뭐가 다른가? → [인접 개념](#인접-개념)
- [ ] 왜 개발 중에 못 찾았고, 고쳤나? → [내 프로젝트에서](#내-프로젝트에서)

---

## 비유

트랜잭션마다 **작업 책상**이 하나 생긴다. DB에서 꺼낸 엔티티를 책상에 올려 두고, 같은 걸 또 찾으면 창고(DB)까지 가지 않고 책상 위 것을 집는다. 책상 위에서 고친 내용은 일이 끝날 때(커밋) 한꺼번에 창고에 반영한다. 일이 끝나면 책상은 치운다.

## 정의

- **영속성 컨텍스트**: 엔티티를 보관·관리하는 곳. `EntityManager`가 관리한다
- **1차 캐시**: 영속성 컨텍스트 안의 Map
  - 키 = **(엔티티 타입, `@Id` 값)** → `Member#1`과 `Team#1`은 섞이지 않는다
  - 값 = 엔티티 객체

엔티티는 영속성 컨텍스트와의 관계에 따라 네 상태를 오간다.

```
New ──persist()──▶ Managed ──flush()──────▶ DB
                    │   ▲   ◀──find()/JPQL──
 detach()/clear()/  │   │ merge()
          close()   ▼   │
                  Detached

Managed ──remove()──▶ Removed ──flush()──▶ DB (DELETE)
```

| 상태 | 뜻 | 들어가는 법 |
|---|---|---|
| 비영속 (New) | 그냥 자바 객체 | `new Member()` |
| 영속 (Managed) | 책상 위, JPA가 관리 | `persist()`, `find()` |
| 준영속 (Detached) | 책상에서 내려감. id는 있지만 관리 안 함 | `detach()`, `clear()`, `close()` |
| 삭제 (Removed) | 삭제 예정 | `remove()` |

- `merge()`는 넘긴 객체를 바꾸지 않는다. **영속 상태인 사본을 돌려주고**, 넘긴 객체는 그대로 준영속이다

## 왜 쓰나

영속성 컨텍스트가 사이에 있어서 생기는 기능이 다섯 개다.

**1차 캐시**: 같은 트랜잭션에서 같은 엔티티를 다시 찾을 때 DB에 안 간다.

**동일성 보장**: 같은 트랜잭션, 같은 id면 같은 객체다.
```java
Member a = em.find(Member.class, 1L);
Member b = em.find(Member.class, 1L);
a == b; // true
```
트랜잭션이 다르면 다른 객체라서 `==`는 false다. 이럴 땐 `@Id` 기준으로 구현한 `equals()`로 비교한다.

**쓰기 지연**: `INSERT`·`UPDATE`·`DELETE`를 바로 보내지 않고 모아 뒀다가 flush 때 보낸다. 중간에 롤백되면 아예 안 보내도 된다.

**변경 감지**: 엔티티 값만 바꾸면 flush 때 처음 상태(스냅샷)와 비교해서 `UPDATE`를 만든다. `save()`를 다시 부를 필요가 없다.
```java
Member m = em.find(Member.class, 1L);
m.setName("새 이름"); // 커밋하면 UPDATE
```

**지연 로딩**: 연관 엔티티(`member.getTeam()`)를 실제로 쓸 때까지 조회를 미룬다. 먼저 가짜 객체(프록시)를 준다. 쓰기 지연과는 다른 것이다 → N+1에서 이어서

## 동작 흐름

**조회** (`em.find`, Spring Data `findById`도 같은 경로)
1. 1차 캐시에서 (타입, id)로 찾는다
2. 있으면 그대로 돌려준다 (DB 안 감)
3. 없으면 DB에서 읽고, 1차 캐시에 올린 뒤 돌려준다

JPQL(`select ...`)은 항상 DB로 간다. 단, 결과가 엔티티이고 이미 1차 캐시에 있으면 **1차 캐시의 옛 객체를 돌려준다.**

**저장·확정·정리**

| | 하는 일 | 되돌리기 |
|---|---|---|
| flush | 모아 둔 변경을 SQL로 DB에 **보낸다** | 롤백 가능. 1차 캐시도 안 비움 |
| 커밋 | DB에서 **확정** | 불가 |
| close / clear | 책상을 치운다 → 엔티티가 준영속 | 저장 안 함 |

flush가 일어나는 때:
- 커밋 직전 (자동)
- JPQL 실행 직전 (자동, 그 쿼리가 다루는 테이블에 쌓인 변경이 있을 때만)
- `em.flush()` 직접 호출

## 조건과 예외

**영속성 컨텍스트 수명**: 기본은 트랜잭션과 같다. 단, **OSIV(`spring.jpa.open-in-view`)가 켜져 있으면 HTTP 요청이 끝날 때까지** 산다. 스프링 기본값은 켜짐이다.

**`readOnly = true`**: Spring이 Hibernate 세션을 읽기 전용 + `FlushMode.MANUAL`로 바꾼다. 스냅샷을 안 만들고, 변경 감지·자동 flush가 없다. JDBC 연결도 읽기 전용으로 설정한다. **1차 캐시는 그대로 쓴다.**

**`IDENTITY` 전략**: id를 DB가 만들어서, `persist()` 하는 순간 `INSERT`가 나간다(쓰기 지연 예외). 그래서 INSERT를 배치로 묶을 수도 없다.

**같은 트랜잭션에서 최신 값 보기**
1. `em.refresh(entity)`로 DB 값을 다시 채운다
2. 트랜잭션을 나눠서 다시 조회한다. 단 OSIV가 꺼져 있어야 하고, 같은 클래스 안에서 자기 메서드를 부르면 프록시를 안 거쳐 트랜잭션이 안 나뉜다 → 프록시 TIL에서 이어서
3. 필요한 값만 JPQL로 조회한다 (`select a.status from CardAttempt a where a.id = :id`). 엔티티가 아니라서 1차 캐시와 상관없다

⚠️ **1차 캐시를 피해도, DB가 새 커밋을 보여 줄지는 격리 수준이 정한다**

| 격리 수준 | 같은 트랜잭션에서 다시 읽으면 | 기본값인 DB |
|---|---|---|
| READ COMMITTED | SQL 한 문장마다 새로 읽음 → 새 커밋이 보임 | PostgreSQL, Oracle |
| REPEATABLE READ | **트랜잭션 안 첫 조회 때** 잡은 스냅샷 유지 → 옛 값 | MySQL InnoDB |

## 인접 개념

| | 1차 캐시 | JPA 2차 캐시 | 스프링 캐시 + Redis |
|---|---|---|---|
| 범위 | 트랜잭션 하나 | 서버 하나 (트랜잭션끼리 공유) | 서버 여러 대가 공유 |
| 담는 것 | 엔티티 객체 | 엔티티를 분해한 값 | 메서드 결과 (DTO 등) |
| 켜기 | 늘 켜짐 | 기본 꺼짐, 설정 필요 | `@EnableCaching` + `@Cacheable` |
| 목적 | 트랜잭션 안에서 객체를 하나로 유지 | DB 조회 줄이기 | DB·외부 호출 줄이기 |

2차 캐시가 있어도 `find`는 1차 캐시를 먼저 본다.

## 장단점

| 장점 | 단점 |
|---|---|
| 같은 엔티티를 여러 번 찾아도 DB 한 번 | 트랜잭션 안에서는 DB 변경이 안 보일 수 있음 |
| 같은 행 = 같은 객체라 값이 안 어긋남 | 트랜잭션이 길면 1차 캐시가 커져 메모리를 먹음 (대량 처리 땐 `clear()`) |
| 변경 감지·쓰기 지연의 바탕 | OSIV가 켜져 있으면 수명이 길어져 더 헷갈림 |

트레이드오프: "트랜잭션 안의 일관성"과 "다른 트랜잭션의 최신 값"은 같이 못 얻는다. 최신 값이 필요한 확인 로직은 트랜잭션 밖에 두거나 값만 조회한다.

## 내 프로젝트에서

**MINE — SSE 구독 직후 재확인이 옛 값을 읽음**
- `SseService.java:25` 클래스 전체가 `@Transactional(readOnly = true)` → `subscribe()`의 64줄·79줄에서 `findById`를 두 번 부름 → 두 번째는 1차 캐시의 PROCESSING을 받아 완료 알림을 놓침
- 재현: `SseRecheckReproTest` 86~105줄. `first == again` true, `again` = PROCESSING, 상태만 뽑은 JPQL = COMPLETED (103줄)
- 이 테스트가 통과하는 건 MINE이 PostgreSQL 기본값 READ COMMITTED라서
- 사실 관계: 프로젝트가 끝난 뒤 **AI 코드 리뷰로 찾았고** 테스트로 재현함. **레포 코드는 안 고침** (11/2 결정)
- 설정: OSIV는 `application-prod.yml:19`·`application-release.yml:20`에서만 꺼짐. 엔티티 19개 전부 `IDENTITY`. `hibernate.jdbc.batch_size` 없음. 2차 캐시 안 씀, Redis는 `RedisConfig.java:36`
- 못 찾은 이유: 통합 테스트가 한 스레드라 다른 커밋이 끼어들 틈이 없었음 [추측]

## 실제 면접에서 받은 질문 (생기면 날짜·회사와 함께)

- (아직 없음)

## 공부 기록

| 날짜 | 질문 | 바뀐 곳 |
|---|---|---|
| 11/4 | 1차 캐시는 언제 DB 대신 답하나 | 처음 작성. [우사기3] 두 번 검토 (OSIV·readOnly·격리 수준·키·merge 수정), 템플릿 형식으로 다시 씀 |

## 남은 것

- [ ] `SseRecheckReproTest` 직접 돌려 보기
- [ ] 면접 문장 쓰기
