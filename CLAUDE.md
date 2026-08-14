# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Video Link Referer Manager** web application (동영상 링크 Referer 매니저) - a single-file HTML application for managing video collections with metadata including product numbers, actor information, ratings, and thumbnails. It stores data in Firebase Realtime Database with localStorage as a fallback.

## Technology Stack

- **Frontend**: Single HTML file with embedded CSS and vanilla JavaScript
- **Backend**: Firebase Realtime Database for data persistence
- **Storage**: Firebase Storage for thumbnail images
- **Auth**: Firebase Authentication
- **Fallback**: Browser localStorage for offline functionality

## File Structure

- `index.html` (~3660 lines) - Complete application with embedded CSS and JavaScript

## How to Work With This Project

### Running Locally

Since this is a single HTML file, you can:
1. **Open directly in browser**: `file://` protocol works but Firebase requires HTTPS or localhost
2. **Serve locally for development**: Use any HTTP server:
   ```bash
   # Using Python
   python -m http.server 8000
   # Using Node.js
   npx http-server
   # Using PHP
   php -S localhost:8000
   ```
3. **Access at**: `http://localhost:8000`

There is no build step, package manager, linter, or test suite in this repo — verify changes by loading the page in a browser and exercising the feature manually.

### Firebase Configuration

- Firebase config is embedded in the HTML (`index.html:1575`)
- Project: `simssangreferer` (Realtime Database region: `asia-southeast1`)
- Firebase is initialized inside a `DOMContentLoaded` listener (near the end of the script, after `setupFirebaseListeners()`)
- A second `DOMContentLoaded` listener further down wires up login/logout UI
- Real-time listeners that sync `videos`, `globalReferer`, `globalBaseUrl`, `secondaryReferer`, and `miscNote` from Firebase are set up in `setupFirebaseListeners()` (`index.html:3325`)

## Architecture & Key Concepts

### Data Layer

The app uses a two-tier data system:

1. **Firebase Realtime Database** (primary)
   - `videos`: Array of video objects
   - `globalReferer`: Single global referer string appended to every video link by default
   - `globalBaseUrl`: Optional global base URL prefix (used for the product-number copy link, not the video link)
   - `secondaryReferer`: Alternate address value, prepended instead of `globalReferer` for videos that opt in via the per-video "보조 Referer 사용" checkbox (see below)
   - `miscNote`: Free-text notes field with zero effect on link composition — pure storage, unrelated to any other logic
   - Real-time listeners keep in-memory `window.currentVideos`, `window.currentGlobalReferer`, `window.currentGlobalBaseUrl`, `window.currentSecondaryReferer`, `window.currentMiscNote` in sync

2. **localStorage** (fallback, also used as a write-through backup even when Firebase is active)
   - Keys: `videos_list` (`STORAGE_KEY`), `global_referer` (`GLOBAL_REFERER_KEY`), `global_base_url` (`GLOBAL_BASE_URL_KEY`), `secondary_referer` (`SECONDARY_REFERER_KEY`), `misc_note` (`MISC_NOTE_KEY`)
   - Automatic fallback if Firebase is not initialized (`firebaseInitialized` flag, `index.html:1587`)

### Video Data Structure

Each video object contains (actual field names, not the `videoLink`/`thumbnailUrl` names used elsewhere in older docs — see `addVideo()`/`updateVideo()` for the source of truth):
```javascript
{
  id: number,                     // Date.now()
  link: string,                   // Composed final link actually opened/copied (see Referer Requirement below)
  originalLink: string,           // Raw link as typed into the form, before referer composition
  productNumber: string,
  actors: string[],               // Parsed from hashtags or comma-separated
  content: string,                // Content description
  rating: number,                 // 1-5 star rating
  thumbnail: string,              // Base64 encoded or external URL
  subtitleUrl: string,            // External subtitle URL
  subtitleEnabled: boolean,       // Whether to show subtitles
  useSecondaryReferer: boolean,   // If true, `link` = secondaryReferer + originalLink (no globalReferer)
  checked: boolean                // Selection state for bulk delete
}
```

### UI Components

The application has three main sections:

1. **Sidebar** (left panel)
   - Actor list with counts (filtered dynamically, sortable by name/count)
   - Rating filter buttons (1-5 stars, All option)
   - "2+ actors" filter toggle and subtitle filter dropdown
   - Collapsible sections

2. **Main Content Area** (center/right)
   - Video list with pagination (24 items per page)
   - Modals for add/edit/detail-view of a video
   - Search functionality by product number

3. **Control Buttons** (top)
   - Add Video
   - Referer Settings
   - Import/Export JSON
   - Download list

### Mobile Back-Button / History Navigation

The app pushes `history.pushState` entries for filter changes, pagination, and modal opens, and handles the Android/mobile hardware back button through a single `popstate` listener, `handleMobileBackButton()` (`index.html:3439`). Its priority order: close an open modal → restore filter state encoded in `history.state` → reset an active filter to "전체" (all) → step back a page → confirm-and-exit the app (signs the user out first). Any new filter, pagination, or modal-opening code must push a matching history state (with a `skipHistory` escape hatch used while a `popstate` event is already being handled) or the back button will behave incorrectly on mobile.

## Key Functions & Their Purposes

### Data Management
- `getVideos()` / `saveVideos(videos)` (`index.html:1710`, `1726`) - Get/save video list with Firebase fallback
- `getGlobalReferer()` / `saveGlobalReferer(referer)` (`index.html:1630`, `1639`) - Manage global referer string
- `getGlobalBaseUrl()` / `saveGlobalBaseUrl(baseUrl)` (`index.html:1650`, `1659`) - Manage global base URL prefix (product-link only)
- `getSecondaryReferer()` / `saveSecondaryReferer(referer)` (`index.html:1670`, `1679`) - Manage the alternate/secondary referer address value
- `getMiscNote()` / `saveMiscNote(note)` (`index.html:1690`, `1699`) - Free-text notes field, unrelated to any link-generation logic
- `checkProductNumberDuplicate()` (`index.html:1759`) - Warn when a product number is already in use
- `addVideo()` (`index.html:1785`) - Add new video with validation
- `updateVideo()` (`index.html:1892`) - Update existing video
- `deleteVideo(id)` (`index.html:2871`) - Remove video

### UI Rendering & Filtering
- `renderList()` (`index.html:2386`) - Render filtered video list with pagination
- `renderSidebar()` (`index.html:2190`) - Render actor/rating sidebar with counts
- `getAllActors()` / `getAllRatings()` (`index.html:2071`, `2117`) - Build sidebar source data
- `filterByActor(actorName)` (`index.html:2315`) - Filter videos by actor
- `filterByRating(rating)` (`index.html:2339`) - Filter videos by rating
- `toggleMultipleActorsFilter()` (`index.html:2296`) - Toggle "2+ actors" filter
- `setSubtitleFilter(value)` (`index.html:2370`) - Filter by subtitle presence/enabled state
- `searchByProductNumber(query)` (`index.html:2363`) - Search by product number
- `showAllVideos(skipHistory)` (`index.html:2249`) - Reset all filters (triggered by header click)
- `goToPage(page, skipHistory)` / `goToPreviousPage(skipHistory)` (`index.html:2578`, `3429`) - Pagination

### Modals & History
- `openModal(id)` / `closeModal()` (`index.html:2639`, `2743`) - Video detail modal
- `openAddVideoModal()` / `openEditModal()` / `closeAddVideoModal()` (`index.html:3243`, `3279`, `3273`) - Add/edit-video modal (shared form; `window.currentEditingVideoId` set → edit, `null` → add)
- `openRefererModal()` / `closeRefererModal()` (`index.html:3184`, `3209`) - Referer settings modal
- `handleMobileBackButton(event)` (`index.html:3439`) - Central `popstate` handler (see above)
- `isAnyModalOpen()` (`index.html:3413`) - Used by the back-button handler to decide whether to close a modal first

### Thumbnails & Import/Export
- `fileToBase64(file)` / `handleThumbnailFile(file)` (`index.html:2021`, `3040`) - Convert/process thumbnail uploads
- `setupDragAndDrop()` (`index.html:3074`) - Drag-and-drop thumbnail upload
- `fetchImageFromUrl(url)` / `isValidImageUrl(url)` (`index.html:3149`, `3142`) - Fetch a thumbnail from a URL
- `importVideoList()` / `downloadList()` (`index.html:2976`, `2900`) - Import/export JSON
- `showMessage(text, type)` (`index.html:2952`) - Toast notification system

### Auth
- `handleLogin()` / `handleLogout()` (`index.html:3599`, `3643`) - Firebase Authentication sign in/out
- Auth persistence is set to `SESSION` and the app force-signs-out on `beforeunload`, so closing the tab always requires a fresh login next time

## Important Implementation Details

### Referer Requirement
- Global referer must be set before adding videos (checked in `addVideo()`, `index.html:1785`) — this guard applies regardless of whether the video will actually use the global referer or the secondary one
- Link composition in both `addVideo()` and `updateVideo()`:
  ```javascript
  const finalLink = useSecondaryReferer
      ? getSecondaryReferer() + videoLink   // per-video opt-in, no global referer
      : videoLink + globalReferer;          // default
  ```
  `useSecondaryReferer` comes from a checkbox (`id="useSecondaryReferer"`, default unchecked) rendered directly under the "동영상 링크" field in the add/edit modal, and is persisted per-video so re-opening the edit modal restores its state.
- `globalBaseUrl` is unrelated to this — it's only used by `copyProductLink()` (`index.html:2785`) to build a product-number link (`baseUrl + productNumber`), separate from the video link itself.
- The composed `link` is what every read path uses — `copyModalLink()`, `openModal()`'s displayed link, and `downloadList()`'s JSON export all read `video.link` directly rather than recomposing it, so changes here only need to happen in `addVideo()`/`updateVideo()`.

### Thumbnail Handling
- Supports direct image upload, drag-drop, and URL fetching
- Images are converted to base64 and stored in the database (Firebase/localStorage)
- Large images can hit Firebase's quota limits — handled around `addVideo()`/`updateVideo()` error paths

### Search & Filter State
- Current filter state lives in module-level `let` variables: `currentFilter`, `currentRatingFilter`, `filterMultipleActors`, `currentSubtitleFilter`, plus `window.currentSearchQuery` and `window.currentVideoPage`
- This state is also what gets serialized into `history.pushState` for back-button support (see above)
- Sidebar updates dynamically based on current filtered data; actor counts reflect the filtered result set

### Pagination
- 24 items per page (hardcoded)
- Page state managed through DOM visibility plus `window.currentVideoPage`
- Search and most filter changes reset to page 1

## Common Development Tasks

### Adding a New Field to Videos
1. Add input element to the modal (search for `<div class="modal" id="addVideoModal">`)
2. Read value in `addVideo()` and `updateVideo()`
3. Include in the video object when saving
4. Display in `renderList()` and the detail modal (`openModal()`)

### Modifying the Sidebar
1. Actor/rating lists are generated from current videos in `renderSidebar()`, sourced via `getAllActors()`/`getAllRatings()`
2. Sidebar filtering is handled by `filterByActor()`, `filterByRating()`, `toggleMultipleActorsFilter()`, `setSubtitleFilter()`

### Adding a New Filter or Modal
- If it changes what's visible in the main list, push a `history.pushState` entry mirroring the existing filter calls, and extend `handleMobileBackButton()` to restore/undo it — otherwise the mobile hardware back button will skip over it.

### Styling Changes
- All CSS is embedded in `<style>` tag near top of HTML
- Uses custom properties for colors (gradients, shadows)
- Responsive design with flex layout; `isMobileDevice()` (`index.html:1576`) gates some mobile-only behavior

## Testing Notes

- The app works offline with localStorage
- Firebase must be available for persistent multi-user sync
- Thumbnail uploads are limited by Firebase Storage quota
- Actor hashtag parsing supports both `#actor1 #actor2` and comma-separated formats
- No automated tests exist; validate by running a local server and testing in-browser, including the mobile back-button flow (Chrome DevTools device toolbar + Android back gesture emulation)

## Development Workflow (자동 적용)

**📌 중요: 모든 작업을 시작할 때 항상 다음 프로세스를 따릅니다.**

### Step 1: Codex CLI 분석 (필수)
새로운 기능을 추가하거나 버그를 수정할 때는 **항상 먼저 codex cli를 사용해서 코드를 분석**합니다:

```bash
# 코드 리뷰 및 분석
mcp__codex-cli__review

# 또는 특정 부분 분석
mcp__codex-cli__codex "prompt: 분석할 내용"
```

**분석 항목:**
- 현재 코드 구조 및 패턴 확인
- 잠재적 버그 및 문제점 식별
- 성능 이슈 및 개선사항 파악
- 모바일 UX 호환성 확인
- 보안 취약점 검토

### Step 2: 변경 계획 수립
codex cli 분석 결과를 바탕으로:
- 변경할 파일과 위치 식별
- 영향 범위 파악
- 테스트 계획 수립

### Step 3: 구현
분석 결과를 바탕으로 코드 수정

### Step 4: 커밋
작업 완료 후 **의미있는 메시지와 함께 직접 커밋**:

```bash
git add index.html CLAUDE.md PROJECT_SUMMARY.md
git commit -m "작업 내용을 설명하는 메시지"
```

**커밋 메시지 예시:**
- `"품번 중복 확인 기능 추가"`
- `"모바일 모달 UI 개선: 상단 정렬 및 배경 스크롤 방지"`
- `"배우 필터링 기능 수정"`

**예시 워크플로우:**
```
새 기능 요청
  ↓
codex cli로 코드 분석 ← (필수)
  ↓
변경 계획 수립
  ↓
코드 수정
  ↓
의미있는 메시지로 수동 커밋 ✅
```

### 주의사항
- codex cli 분석 **없이** 코드를 바로 수정하지 않기
- 분석 결과가 중요한 발견사항을 담고 있으면 먼저 공유하기
- 대규모 변경은 여러 커밋으로 나누기
- **커밋 메시지는 "뭘 했는지" 명확하게 작성하기** (예: "작업 완료" ❌, "모바일 UI 개선" ✅)
