# 동영상 링크 Referer 매니저 개발 로그

## 개요

**프로젝트명**: 동영상 링크 Referer 매니저 (Video Link Referer Manager)  
**기술 스택**: HTML5 + Vanilla JavaScript + Firebase (Authentication, Realtime Database, Storage)  
**파일 구조**: 단일 HTML 파일 (`index.html`, 2732줄)  
**개발 기간**: 2026-04-21 ~ 2026-04-22

---

## 핵심 기능 요약

### 1. Firebase 기반 인증 시스템
- **구현 위치**: `index.html:1034`, `2611`, `2621`, `2683`, `2727`
- **기능**: 로그인/로그아웃, 로그인 화면/앱 화면 자동 전환
- **특징**: Firebase Authentication 호환성

### 2. 영상 컨텐츠 관리 (CRUD)
- **구현 위치**: `index.html:1415`, `1517`, `1720`, `1803`, `2142`
- **기능**: 
  - 영상 추가/수정/삭제
  - 배우별 필터링, 평점 필터링, 품번 검색
  - 실시간 목록 렌더링 및 페이지네이션 (24개/페이지)

### 3. 썸네일 및 미디어 관리
- **구현 위치**: `index.html:1609`, `2345`, `2182`, `2247`
- **기능**:
  - 썸네일 업로드 및 드래그앤드롭
  - Base64 인코딩으로 데이터베이스 저장
  - JSON 가져오기/내보내기

### 4. 전역 Referer 설정 및 자동 링크 생성
- **구현 위치**: `index.html:1160`, `1320`, `1346`, `1355`, `1471`, `2581`
- **기능**:
  - 전역 Referer 문자열 설정 모달
  - 영상 링크에 자동으로 Referer 부착 (`finalLink = videoLink + globalReferer`)
  - Firebase 실시간 동기화 및 localStorage 백업

### 5. Firebase 실시간 동기화
- **구현 위치**: `index.html:2566`, `2581`
- **기능**: 데이터 변경 시 즉시 반영, 오프라인 대응 (localStorage 폴백)

---

## 최근 커밋 히스토리 및 변경사항

### Commit 1: `e647ad2` - "referer" (2026-04-21 23:37 KST)

#### 목적
모든 영상 링크에 공통 **Referer 문자열**을 자동으로 부착하고, 이를 사용자 간 동기화하는 기능 추가

#### 구현 내용

**UI 변경사항**
- Referer 설정 모달 추가 (`index.html:1160`)
  ```
  - 제목: "전역 Referer 설정"
  - 입력 필드: 사용자가 Referer 문자열 입력
  - 저장/취소 버튼
  ```

**데이터 관리**
- localStorage 키: `GLOBAL_REFERER_KEY` 사용
- Firebase 경로: `globalReferer` 노드에 저장
- 저장/조회 함수:
  - `saveGlobalReferer(referer)` (line 1346)
  - `getGlobalReferer()` (line 1355)

**링크 생성 로직**
- 영상 추가 시: `finalLink = videoLink + globalReferer` (line 1417)
- 영상 수정 시: 동일 방식 적용 (line 1519)
- 영상 렌더링 시: 전체 링크 표시 (line 1471)

**Import/Export 연동**
- JSON 내보내기에 `globalReferer` 포함 (line 2265)
- JSON 가져오기 시 `globalReferer` 복원 (line 2182, 2198)

**Firebase 실시간 리스너**
- 백엔드 `globalReferer` 변경 감지 (line 2581)
- 변경 시 UI 즉시 업데이트

---

### Commit 2: `33d212e` - "자동 로그아웃" (2026-04-21 23:46 KST)

#### 목적
브라우저 종료 시 로그인 세션이 유지되지 않도록 **세션 기반 지속성** 적용

#### 구현 내용

**Firebase Auth 설정 변경**
- 위치: `index.html:2615`
- 변경 사항:
  ```javascript
  // Before: 기본 지속성 (indefinite)
  // After: 브라우저 세션 단위 지속성
  firebase.auth().setPersistence(firebase.auth.Auth.Persistence.SESSION);
  ```

**의미**
- 탭/브라우저 창을 닫으면 세션 소멸
- 새로 열 때는 다시 로그인 필요
- 로컬 스토리지에 토큰 저장 불가

---

### Commit 3: `cc00952` - "자동 로그 아웃" (2026-04-21 23:52 KST)

#### 목적
브라우저 창 종료 시 **명시적 로그아웃** 실행으로 보안 강화

#### 구현 내용

**이벤트 리스너 추가**
- 위치: `index.html:2736`
- 구현:
  ```javascript
  window.addEventListener('beforeunload', () => {
    if (firebase.auth().currentUser) {
      firebase.auth().signOut();
    }
  });
  ```

**의미**
- 창을 닫을 때 자동으로 `signOut()` 호출
- Commit 2의 세션 지속성과 함께 이중 보안 구성
- 명시적 로그아웃으로 Firebase 세션 완전 정리

---

### Commit 4: "Referer 설정 - 보조 Referer 주소값 + 동영상별 체크박스 추가" (2026-08-15)

#### 목적
사이트/링크별로 "글로벌 Referer를 붙일지, 아니면 별도 주소값을 앞에 붙일지"를 동영상 단위로 선택할 수 있게 함.
(처음에는 "제외 키워드"로 링크 안의 특정 단어를 자동 감지하는 방식을 시도했으나, 요구사항을 다시 들은 뒤 아래의 체크박스 방식으로 전면 교체함 — 제외 키워드 관련 코드/필드는 모두 삭제됨.)

#### 구현 내용

**① 보조 Referer 주소값 (`secondaryReferer`)**
- 위치: Referer 설정 모달의 "보조 Referer 주소값" 입력란 (`<div class="modal" id="refererModal">` 안, 글로벌 Referer 텍스트 바로 아래)
- 이 필드 자체는 기존과 동일한 저장/동기화 함수(`getSecondaryReferer()`/`saveSecondaryReferer()`)를 그대로 사용. localStorage 키 `secondary_referer`, Firebase 경로 `secondaryReferer`로 저장되고 `setupFirebaseListeners()`에서 실시간 동기화됨.

**② 동영상별 "보조 Referer 사용" 체크박스**
- 위치: 동영상 추가/수정 모달의 "동영상 링크" 입력란 바로 아래, `id="useSecondaryReferer"` 체크박스 (기본값: 미체크)
- `addVideo()` / `updateVideo()`의 링크 합성 로직:
  ```javascript
  const finalLink = useSecondaryReferer
      ? getSecondaryReferer() + videoLink   // 체크: 보조 Referer 주소값 + 동영상 링크 (글로벌 Referer 미포함)
      : videoLink + globalReferer;          // 미체크(기본값): 기존처럼 링크 + 글로벌 Referer
  ```
- 즉, 이 체크박스를 켠 동영상만 "보조 Referer 주소값 + 원본 링크" 형태로 합쳐진 링크가 저장되고, 글로벌 Referer는 붙지 않음. 체크박스는 각 동영상 객체에 `useSecondaryReferer: boolean`으로 저장되어 수정 모달을 다시 열었을 때도 상태가 유지됨.
- "링크복사"(`copyModalLink()`)나 상세보기(`openModal()`)는 항상 저장된 `video.link` 값을 그대로 사용하므로, 위 로직으로 합성해 저장해두면 복사 시에도 자동으로 올바른 링크가 복사됨 (기존 아키텍처와 동일한 패턴).

#### 주의사항 / 확인된 사항
- 기존에 저장된 동영상의 `link`/`useSecondaryReferer` 필드는 소급 적용되지 않음 (새로 추가/수정하는 영상부터 적용, 기존 영상은 `useSecondaryReferer`가 `undefined` → 체크 해제 상태로 취급됨)
- JSON 가져오기/내보내기(`importVideoList()`/`downloadList()`)에는 별도 반영하지 않음 (요청 범위 밖, `useSecondaryReferer` 필드는 다른 필드처럼 그대로 딸려서 내보내짐)
- Node 문법 검사(`new Function()`)로 스크립트 블록 전체 문법 오류 없음을 확인함
- 실제 브라우저 동작 테스트(체크박스 토글, 링크 복사 결과 확인 등)는 사용자가 직접 진행 예정

---

### Commit 5: "Referer 설정 - 기타 메모 텍스트박스 추가" (2026-08-15)

#### 목적
링크 생성 로직과는 전혀 무관하게, 그냥 자유롭게 저장해 둘 수 있는 메모용 텍스트박스가 필요하다는 요청에 대응.

#### 구현 내용
- 위치: Referer 설정 모달의 "기타 메모" 입력란 (`<div class="modal" id="refererModal">` 안, "보조 Referer 주소값" 아래)
- 기존 `globalReferer`/`secondaryReferer`와 완전히 동일한 패턴으로 구현됨:
  - localStorage 키: `misc_note` (`MISC_NOTE_KEY`)
  - Firebase 경로: `miscNote`
  - 조회/저장 함수: `getMiscNote()` / `saveMiscNote(note)`
  - `setupFirebaseListeners()`에 실시간 리스너 추가 → 다른 기기/탭에서 값이 바뀌면 즉시 반영
  - `openRefererModal()`에서 모달 열 때 값 로드, `saveReferer()`에서 저장 시 (빈 값이어도) 항상 저장 — 나중에 비워서 초기화 가능
- **중요**: 이 필드는 `addVideo()`/`updateVideo()`의 링크 합성 로직 어디에도 관여하지 않음. 순수하게 "저장되는 텍스트" 그 이상도 이하도 아님.

#### 확인된 사항
- Node 문법 검사(`new Function()`)로 스크립트 블록 전체 문법 오류 없음을 확인함
- 실제 브라우저 동작 테스트는 사용자가 직접 진행 예정

---

## 기술 아키텍처

### 데이터 흐름도

```
┌─────────────────────────────────────────────┐
│         Firebase Realtime Database          │
│  ┌─────────────────────────────────────┐    │
│  │ - videos (배열)                     │    │
│  │ - globalReferer (전역 설정)        │    │
│  └─────────────────────────────────────┘    │
└─────────┬──────────────────────────┬─────────┘
          │ 실시간 리스너             │ CRUD
          ↓                          ↓
    ┌──────────────────────────────────────┐
    │   JavaScript 메모리 (상태 관리)      │
    │ - window.currentVideos              │
    │ - window.currentGlobalReferer       │
    │ - currentFilter, currentRatingFilter │
    └──────────────────────────────────────┘
          ↕ 백업/복구
    ┌──────────────────────────────────────┐
    │      Browser localStorage           │
    │ - videos_list                       │
    │ - global_referer                    │
    └──────────────────────────────────────┘
```

### 주요 함수 맵

| 범주 | 함수명 | 위치 | 설명 |
|------|--------|------|------|
| **데이터 조회/저장** | `getVideos()` | 1320 | 동영상 목록 조회 |
| | `saveVideos(videos)` | - | 동영상 목록 저장 |
| | `getGlobalReferer()` | 1355 | 전역 Referer 조회 |
| | `saveGlobalReferer(referer)` | 1346 | 전역 Referer 저장 |
| **동영상 CRUD** | `addVideo()` | 1415 | 새 동영상 추가 |
| | `updateVideo()` | 1517 | 동영상 수정 |
| | `deleteVideo(id)` | 1720 | 동영상 삭제 |
| **UI 렌더링** | `renderList()` | 1803 | 필터된 목록 렌더링 |
| | `renderSidebar()` | 2142 | 사이드바(배우/평점) 렌더링 |
| **필터/검색** | `filterByActor(name)` | - | 배우별 필터 |
| | `filterByRating(rating)` | - | 평점별 필터 |
| | `searchByProductNumber(query)` | - | 품번 검색 |
| **유틸** | `fileToBase64(file)` | 1609 | 이미지 → Base64 변환 |
| | `handleThumbnailFile(file)` | 2345 | 썸네일 처리 |
| | `importVideoList()` | 2182 | JSON 가져오기 |
| | `downloadList()` | 2247 | JSON 내보내기 |

---

## 보안 개선 사항 (Commit 2, 3)

### Before (Commit 1)
```
[로그인] → Firebase Auth (indefinite) → [창 종료 후 재접속] → 로그인 상태 유지 ❌ 보안 위험
```

### After (Commit 2 + 3)
```
[로그인] → Firebase Auth (SESSION) → [beforeunload 이벤트 signOut]
    ↓
[창 종료] → 세션 소멸 + 명시적 로그아웃 ✅ 이중 보안
    ↓
[재접속] → 새로 로그인 필요 ✅ 안전
```

---

## 영상 데이터 구조

```javascript
{
  id: string,                    // 고유 ID
  videoLink: string,             // 원본 영상 링크
  productNumber: string,         // 상품번호
  actors: string[],              // 배우 목록 (해시태그 또는 쉼표 구분)
  content: string,               // 설명
  rating: number,                // 평점 (1-5)
  thumbnailUrl: string,          // 썸네일 (Base64 또는 URL)
  subtitleUrl: string,           // 자막 URL
  subtitleEnabled: boolean,      // 자막 활성화 여부
  useSecondaryReferer: boolean   // (2026-08-15 추가) true면 링크 = 보조 Referer 주소값 + 원본 링크(글로벌 Referer 미포함)
}
```

### 영상 링크 최종 형태
```
finalLink = videoLink + globalReferer
예) "https://example.com/video" + "?ref=manager&id=user123"
  = "https://example.com/video?ref=manager&id=user123"
```

---

## 사용 시나리오

### 시나리오 1: 신규 사용자 - 첫 동영상 추가

1. 로그인 (`firebase.auth().signInWithEmailAndPassword()`)
2. **Referer 설정** 모달에서 전역 Referer 입력
   - 예) `?ref=mymanager&user=john`
3. "동영상 추가" 버튼 클릭
4. 폼 작성 및 제출
5. 저장 시 자동으로 `videoLink + globalReferer`로 합성
6. Firebase에 저장 및 실시간 동기화

### 시나리오 2: 재접속 시 데이터 복구

1. 로그인
2. Firebase 실시간 리스너 활성화
3. `globalReferer`, `videos` 데이터 자동 다운로드
4. 기존 동영상 목록 복원

### 시나리오 3: 오프라인 동작

1. 인터넷 연결 끊김 → localStorage에서 데이터 로드
2. 오프라인 상태로 조회/필터링 가능
3. 재연결 시 Firebase와 동기화

### 시나리오 4: 브라우저 종료 시

1. 창 닫기 버튼 클릭
2. `beforeunload` 이벤트 발생
3. `firebase.auth().signOut()` 자동 실행
4. 세션 정보 삭제
5. 다음 접속 시 재로그인 필요

---

## 주요 특징 및 이점

| 특징 | 이점 |
|------|------|
| **단일 HTML 파일** | 배포 간편, 서버 의존성 제로 |
| **Firebase 실시간 동기화** | 다중 기기 간 데이터 자동 동기화 |
| **localStorage 백업** | 오프라인 사용 가능, 성능 향상 |
| **Base64 썸네일 저장** | 별도 CDN 필요 없음, 자체 포함 |
| **세션 기반 인증** | 보안 강화, 자동 로그아웃 |
| **Referer 중앙 관리** | 모든 영상 링크에 일관된 메타데이터 부착 |

---

## 개발 마일스톤

| 단계 | 커밋 | 날짜 | 주요 성과 |
|------|------|------|----------|
| **Phase 1** | e647ad2 | 2026-04-21 23:37 | Referer 기능 구현, 영상 CRUD 완성 |
| **Phase 2** | 33d212e | 2026-04-21 23:46 | 세션 기반 인증 적용 |
| **Phase 3** | cc00952 | 2026-04-21 23:52 | 명시적 로그아웃 (보안 이중화) |

---

## 추후 개선 가능 사항

- [ ] TypeScript 마이그레이션 (타입 안정성)
- [ ] Vue.js/React 전환 (컴포넌트 재사용성)
- [ ] 동영상 스트림 기능 (재생 기능 통합)
- [ ] 배우별 통계/분석 대시보드
- [ ] 다국어 지원 (i18n)
- [ ] 다크모드 UI 옵션
- [ ] 고급 검색 (날짜 범위, 다중 필터)
- [ ] 사용자 권한 관리 (Admin/Viewer)

---

## 파일 정보

- **파일명**: `index.html`
- **줄 수**: 2732줄
- **크기**: ~120KB (추정)
- **배포 방식**: 정적 파일 호스팅 또는 로컬 HTTP 서버

---

## 개발팀 정보

- **개발자**: simssang (simssang@xlgames.com)
- **저장소**: E:\fork\SimReferer
- **현재 브랜치**: main
- **마지막 업데이트**: 2026-04-21 23:52

---

**문서 작성일**: 2026-04-22  
**문서 형식**: 나무위키 스타일 마크다운  
*이 문서는 프로젝트의 개발 현황과 아키텍처를 전체적으로 이해하기 위한 참고 자료입니다.*
