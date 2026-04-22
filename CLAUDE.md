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

- `index.html` (2732 lines) - Complete application with embedded CSS and JavaScript

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

### Firebase Configuration

- Firebase config is embedded in the HTML (around line 1310)
- Current config ID: `1:832680598640:web:28854dad3330deb47be8b5`
- The app initializes Firebase on `DOMContentLoaded` (line 2608)
- Real-time listeners sync data from Firebase to the app (line 2566, 2581)

## Architecture & Key Concepts

### Data Layer

The app uses a two-tier data system:

1. **Firebase Realtime Database** (primary)
   - `videos`: Array of video objects
   - `globalReferer`: Single global referer string
   - Real-time listeners keep in-memory `window.currentVideos` and `window.currentGlobalReferer` in sync

2. **localStorage** (fallback)
   - Used when Firebase is not initialized
   - Keys: `videos_list`, `global_referer`
   - Automatic fallback if Firebase fails

### Video Data Structure

Each video object contains:
```javascript
{
  id: string,
  videoLink: string,
  productNumber: string,
  actors: string[],           // Can be parsed from hashtags or comma-separated
  content: string,            // Content description
  rating: number,             // 1-5 star rating
  thumbnailUrl: string,       // Base64 encoded or external URL
  subtitleUrl: string,        // External subtitle URL
  subtitleEnabled: boolean    // Whether to show subtitles
}
```

### UI Components

The application has three main sections:

1. **Sidebar** (left panel)
   - Actor list with counts (filtered dynamically)
   - Rating filter buttons (1-5 stars, All option)
   - Collapsible sections

2. **Main Content Area** (center/right)
   - Video list with pagination (24 items per page)
   - Modals for add/edit video
   - Search functionality by product number

3. **Control Buttons** (top)
   - Add Video
   - Referer Settings
   - Import/Export JSON
   - Download list

## Key Functions & Their Purposes

### Data Management
- `getVideos()` / `saveVideos(videos)` - Get/save video list with Firebase fallback
- `getGlobalReferer()` / `saveGlobalReferer(referer)` - Manage global referer
- `addVideo()` - Add new video with validation
- `updateVideo()` - Update existing video
- `deleteVideo(id)` - Remove video

### UI Rendering
- `renderList()` - Render filtered video list with pagination
- `renderSidebar()` - Render actor/rating sidebar with counts
- `filterByActor(actorName)` - Filter videos by actor
- `filterByRating(rating)` - Filter videos by rating
- `searchByProductNumber(query)` - Search by product number
- `goToPage(page)` - Pagination

### Utilities
- `fileToBase64(file)` - Convert image to base64 for storage
- `handleThumbnailFile(file)` - Process thumbnail uploads
- `importVideoList()` / `downloadList()` - Import/export JSON
- `showMessage(text, type)` - Toast notification system

## Important Implementation Details

### Referer Requirement
- Global referer must be set before adding videos (checked in `addVideo()` at line 1415)
- Referer is stored globally and used by all videos

### Thumbnail Handling
- Supports direct image upload, drag-drop, and URL fetching
- Images are converted to base64 and stored in the database (Firebase/localStorage)
- Large images can hit Firebase's quota limits - noted in error handling at line 1398

### Search & Filter State
- Current filter state: `currentFilter`, `currentRatingFilter`, `window.currentSearchQuery`
- Sidebar updates dynamically based on current data
- Actor counts reflect filtered results

### Pagination
- 24 items per page (hardcoded)
- Page state managed through DOM visibility
- Search resets to page 1

## Firebase Realtime Listeners

Data synchronization happens through real-time listeners (set up around line 2566-2581):
- Changes in Firebase instantly update the in-memory state
- `renderList()` is called after data updates
- If Firebase is unavailable, app falls back to localStorage

## Common Development Tasks

### Adding a New Field to Videos
1. Add input element to the modal (search for `<div class="modal" id="addVideoModal">`)
2. Read value in `addVideo()` function
3. Include in video object when saving
4. Display in `renderList()` video item

### Modifying the Sidebar
1. Actor/rating lists are generated from current videos in `renderSidebar()`
2. Both use `getAllActors()` and `getAllRatings()` helper functions
3. Sidebar filtering is handled by `filterByActor()` and `filterByRating()`

### Styling Changes
- All CSS is embedded in `<style>` tag near top of HTML
- Uses custom properties for colors (gradients, shadows)
- Responsive design with flex layout

## Testing Notes

- The app works offline with localStorage
- Firebase must be available for persistent multi-user sync
- Thumbnail uploads are limited by Firebase Storage quota
- Actor hashtag parsing supports both `#actor1 #actor2` and comma-separated formats

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

### Step 4: 자동 커밋
작업 완료 후 세션 종료 시 자동으로 커밋됨

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
세션 종료 → 자동 커밋 ✅
```

### 주의사항
- codex cli 분석 **없이** 코드를 바로 수정하지 않기
- 분석 결과가 중요한 발견사항을 담고 있으면 먼저 공유하기
- 대규모 변경은 여러 커밋으로 나누기
