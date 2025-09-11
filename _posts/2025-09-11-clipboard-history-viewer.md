---
title: "[Chrome Extension] Clipboard History Viewer 개발기"
excerpt: "브라우저에서 텍스트 복사를 자동 기록·검색·핀·삭제까지. TypeScript + Vite + MV3로 만든 경량 확장."
categories:
  - Development
tags:
  - chrome-extension
  - manifest-v3
  - typescript
  - vite
  - crxjs
  - clipboard
permalink: /development/clipboard-history-viewer/
toc: true
toc_sticky: true

date: 2025-09-11
last_modified_at: 2025-09-11
---

# [Chrome Extension] Clipboard History Viewer 개발기

2025년 9월 11일

[![Demo thumbnail](/assets/images/posts_img/clipboard-history-viewer.png)](https://www.youtube.com/watch?v=hfsTbGbjUAk)

---

## 1. 프로젝트 소개

브라우저에서 텍스트를 복사하다 보면 “방금 전에 복사했던 그 문장”을 다시 찾고 싶을 때가 많다. macOS용 클립보드 매니저도 좋지만, 브라우저 안에서 가볍게 쓸 수 있는 도구가 있으면 더 편리하다고 느꼈다.

Clipboard History Viewer는 웹에서 복사한 텍스트를 자동으로 기록하고, 팝업에서 검색·복사·삭제·핀 고정까지 할 수 있는 Chrome/Brave 확장 프로그램이다. Manifest V3 기반으로 TypeScript, Vite, @crxjs/vite-plugin을 사용해 만들었다.

- GitHub 저장소: [https://github.com/esoyeon/clipboard-history-viewer](https://github.com/esoyeon/clipboard-history-viewer)
- 데모 영상: [https://www.youtube.com/watch?v=hfsTbGbjUAk](https://www.youtube.com/watch?v=hfsTbGbjUAk)

---

## 2. 기능 기획

핵심은 단순했다. “복사한 텍스트를 빠르게 찾고, 다시 쓰기 좋게.”

- 📋 자동 기록: 웹페이지에서 복사(Cmd/Ctrl+C) 시 히스토리에 자동 저장
- 🔍 검색: 저장된 텍스트/도메인을 기준으로 실시간 필터링
- 📌 핀 고정: 자주 쓰는 문장은 상단에 고정 (Clear All에서도 보호)
- ⏱️ 타임스탬프/출처: 언제/어디서 복사했는지 표시
- 🗑️ 관리: 개별 삭제, 전체 삭제(핀 항목은 보호), 수동으로 클립보드 읽기
- ⌨️ 단축키: 히스토리 열기(Ctrl+Shift+Y / Cmd+Shift+Y), 현재 클립보드 저장(Ctrl+Shift+U / Cmd+Shift+U)

### 설계 의도

- “빠르게 찾기”가 목표라서 텍스트 위주의 단순한 UI/UX를 채택했습니다. 검색은 입력 즉시 필터링되도록 클라이언트 사이드에서 처리합니다.
- 브라우저 보안 정책(사용자 제스처 필요, 특수 페이지 제한)을 고려하여 자동 수집은 웹페이지에서의 복사 이벤트에 집중하고, 외부 앱에서 복사한 내용은 버튼/단축키로 “수동 저장” 흐름을 제공합니다.
- 중복은 “연속 중복(바로 직전과 동일)”과 “전체 중복(리스트 내 중복)”을 분리해 처리합니다. 연속 중복은 노이즈를 줄이고, 전체 중복 제거는 최신 항목만 남기기 위함입니다.
- “핀 고정”은 단순 즐겨찾기가 아니라 “보호”의 의미를 갖도록 했습니다. Clear All 시에도 핀은 남겨 워크플로우를 망치지 않도록 했습니다.

---

## 3. 개발 과정

### 기술 스택
- Manifest V3 기반 Chrome/Brave Extension
- TypeScript + Vite + @crxjs/vite-plugin
- chrome.storage.local로 데이터 영구 저장

### 디렉터리 구조
   
```bash
clipboard-history/
├─ manifest.json
├─ sw.ts          # Service Worker (백그라운드)
├─ content.ts     # copy/cut 이벤트 감지
├─ popup.html     # UI
├─ popup.ts       # 검색/복사/핀/삭제 로직
├─ styles.css
└─ icons/
```

### MV3 + CRX 설정 포인트
@crxjs/vite-plugin은 manifest.json의 경로를 “프로젝트 루트 기준”으로 해석한다. TypeScript 파일을 직접 참조하려면 `src/` 접두어를 붙여야 한다.

```json
{
  "manifest_version": 3,
  "action": { "default_popup": "src/popup.html" },
  "background": { "service_worker": "src/sw.ts", "type": "module" },
  "content_scripts": [{ "matches": ["<all_urls>"], "js": ["src/content.ts"] }],
  "icons": {
    "16": "src/icons/16.png",
    "48": "src/icons/48.png",
    "128": "src/icons/128.png"
  },
  "commands": {
    "open-popup": {
      "suggested_key": { "default": "Ctrl+Shift+Y", "mac": "Command+Shift+Y" },
      "description": "Open clipboard history"
    },
    "save-current-clipboard": {
      "suggested_key": { "default": "Ctrl+Shift+U", "mac": "Command+Shift+U" },
      "description": "Save current clipboard to history"
    }
  }
}
```

### 주요 구현 흐름

1) Content Script — 복사 감지 및 전송

웹페이지에서 발생하는 `copy/cut` 이벤트를 가장 먼저 잡아냅니다. 선택된 텍스트가 있으면 그것을 우선 사용하고, 없으면 `ClipboardEvent`의 `clipboardData`에서 텍스트를 읽습니다. 일부 브라우저(특히 Brave) 환경에서 이벤트 타이밍/권한 차이가 있어, 짧은 지연 후 `navigator.clipboard.readText()`로 폴백을 한 번 더 시도합니다. 최종적으로는 Service Worker로 메시지를 보냅니다.
   
```ts
// src/content.ts
function pushCopy(text: string) {
  if (!text) return;
  chrome.runtime.sendMessage({
    type: "COPIED",
    text,
    url: location.href,
    host: location.host,
    ts: Date.now()
  });
}

document.addEventListener("copy", (e) => {
  const sel = window.getSelection()?.toString();
  if (sel && sel.trim()) return pushCopy(sel);

  const dt = (e as ClipboardEvent).clipboardData;
  const t = dt?.getData("text/plain") || dt?.getData("text");
  if (t && t.trim()) pushCopy(t);
});
```

2) Service Worker — 중복 제거/저장

백그라운드(Service Worker)는 수신한 텍스트를 `chrome.storage.local`에 저장합니다. 저장 정책은 다음과 같습니다.
- 최대 200개로 제한(LRU 느낌으로 상단이 최신)
- “연속 중복”은 바로 무시하여 노이즈 제거
- “전체 중복”은 기존 항목을 제거한 뒤 맨 앞에 새로 추가(최신 본문 유지)
   
```ts
// src/sw.ts
const KEY = "clipboard_history";
const LIMIT = 200;

chrome.runtime.onMessage.addListener((msg, _sender, sendResponse) => {
  (async () => {
    if (msg?.type !== "COPIED") return;

    const { [KEY]: items = [] } = await chrome.storage.local.get(KEY);
    const text = String(msg.text ?? "").trim();
    if (!text) return;

    // 연속/전체 중복 제거
    if (items.length && items[0]?.text === text) return;
    const filtered = (items as any[]).filter(i => i.text !== text);

    filtered.unshift({
      text,
      ts: msg.ts || Date.now(),
      url: msg.url || "",
      host: msg.host || ""
    });
    if (filtered.length > LIMIT) filtered.length = LIMIT;

    await chrome.storage.local.set({ [KEY]: filtered });
    sendResponse({ ok: true });
  })();
  return true; // async
});
```
   
3) Popup — 검색/복사/핀/삭제 + Clear All 시 핀 보호

팝업은 “바로 찾아 쓰기”에 집중했습니다.
- 검색은 소문자 변환 후 포함 여부만 체크하는 단순/빠른 방식
- 복사는 기본 `navigator.clipboard.writeText` 사용, 실패 시 `execCommand('copy')` 폴백
- 핀은 `Set<string>`(본문 기준)으로 관리하고 `chrome.storage.local`에 별도 저장하여 세션 간 유지
- Clear All은 “핀 보호”가 기본값: 핀만 남기고 나머지만 삭제
      
```ts
// src/popup.ts (발췌)
const KEY = "clipboard_history";
let allItems: Item[] = [];
let pins = new Set<string>();

function filterBy(q: string) {
  const query = (q || "").trim().toLowerCase();
  if (!query) return allItems;
  return allItems.filter(i =>
    i.text.toLowerCase().includes(query) ||
    (i.host || "").toLowerCase().includes(query)
  );
}

// Clear All: 핀 항목은 보호
document.getElementById("clear")!.addEventListener("click", async () => {
  const pinnedItems = allItems.filter((item) => pins.has(item.text));
  const unpinnedCount = allItems.length - pinnedItems.length;
  if (unpinnedCount === 0) {
    alert("No unpinned items to clear. All items are pinned.");
    return;
  }
  const message = pinnedItems.length > 0
    ? `Clear ${unpinnedCount} unpinned items? (${pinnedItems.length} pinned items will be kept)`
    : "Are you sure you want to clear all history?";
  if (!confirm(message)) return;

  await chrome.storage.local.set({ [KEY]: pinnedItems });
  // ...
});
```

---

## 4. 사용 방법

### 1) 클론 & 설치 & 빌드  
```bash
git clone https://github.com/esoyeon/clipboard-history-viewer.git
cd clipboard-history-viewer
npm install
npm run build
```

### 2) Chrome/Brave에 로드
- chrome://extensions (또는 brave://extensions)
- “개발자 모드” ON → “Load unpacked” → `dist` 폴더 선택

### 3) 사용
- 📋 웹에서 텍스트를 복사하면 자동 기록
- 🔎 팝업 검색창으로 실시간 필터링
- 📌 자주 쓰는 문장은 Pin
- 🗑️ Clear All 시에도 Pin은 보호
- ⌨️ 단축키
  - 히스토리 열기: Ctrl+Shift+Y (macOS Cmd+Shift+Y)
  - 현재 클립보드 저장: Ctrl+Shift+U (macOS Cmd+Shift+U)

---

## 5. 한계와 개선 아이디어

- 🔒 사용자 제스처 제약: 브라우저 보안상 일부 클립보드 읽기는 제스처가 필요
- 🧾 Plain Text 중심: 이미지/리치 텍스트는 미지원
- 💾 내보내기: CSV/JSON 내보내기 기능 추가 예정
- 🗂️ 그룹핑: 도메인별/태그별로 묶어서 보기

---

## 6. 마무리

작지만 매일 쓰게 되는 유틸리티를 브라우저 확장으로 만들어 보았다. Manifest V3 환경에서 Service Worker, Content Script, Popup 흐름을 온전히 구현했고, 실제 사용성(검색/핀/단축키/중복 제거/Brave 호환성)을 꼼꼼히 보강했다.

- GitHub Repo: [https://github.com/esoyeon/clipboard-history-viewer](https://github.com/esoyeon/clipboard-history-viewer)
- Demo YouTube: [https://www.youtube.com/watch?v=hfsTbGbjUAk](https://www.youtube.com/watch?v=hfsTbGbjUAk)
