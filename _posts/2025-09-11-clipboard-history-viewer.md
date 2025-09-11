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
header:
  image: /assets/images/clipboard-history-viewer.png

date: 2025-09-11
last_modified_at: 2025-09-11
---

# [Chrome Extension] Clipboard History Viewer 개발기

2025년 9월 11일

## On this page
- [프로젝트 소개](#프로젝트-소개) · [Permalink](#프로젝트-소개)
- [기능 기획](#기능-기획) · [Permalink](#기능-기획)
- [개발 과정](#개발-과정) · [Permalink](#개발-과정)
- [사용 방법](#사용-방법) · [Permalink](#사용-방법)
- [한계와 개선 아이디어](#한계와-개선-아이디어) · [Permalink](#한계와-개선-아이디어)
- [마무리](#마무리) · [Permalink](#마무리)

---

## 1. 프로젝트 소개
[Permalink](#프로젝트-소개)

브라우저에서 텍스트를 복사하다 보면 “방금 전에 복사했던 그 문장”을 다시 찾고 싶을 때가 많다. macOS용 클립보드 매니저도 좋지만, 브라우저 안에서 가볍게 쓸 수 있는 도구가 있으면 더 편리하다고 느꼈다.

Clipboard History Viewer는 웹에서 복사한 텍스트를 자동으로 기록하고, 팝업에서 검색·복사·삭제·핀 고정까지 할 수 있는 Chrome/Brave 확장 프로그램이다. Manifest V3 기반으로 TypeScript, Vite, @crxjs/vite-plugin을 사용해 만들었다.

- GitHub 저장소: https://github.com/esoyeon/clipboard-history-viewer
- 데모 영상: https://www.youtube.com/watch?v=hfsTbGbjUAk

![Demo thumbnail](/assets/images/clipboard-history-viewer.png)

---

## 2. 기능 기획
[Permalink](#기능-기획)

핵심은 단순했다. “복사한 텍스트를 빠르게 찾고, 다시 쓰기 좋게.”

- 📋 자동 기록: 웹페이지에서 복사(Cmd/Ctrl+C) 시 히스토리에 자동 저장
- 🔍 검색: 저장된 텍스트/도메인을 기준으로 실시간 필터링
- 📌 핀 고정: 자주 쓰는 문장은 상단에 고정 (Clear All에서도 보호)
- ⏱️ 타임스탬프/출처: 언제/어디서 복사했는지 표시
- 🗑️ 관리: 개별 삭제, 전체 삭제(핀 항목은 보호), 수동으로 클립보드 읽기
- ⌨️ 단축키:
  - 히스토리 열기: Ctrl+Shift+Y (macOS는 Cmd+Shift+Y)
  - 현재 클립보드 저장: Ctrl+Shift+U (macOS는 Cmd+Shift+U)

---

## 3. 개발 과정
[Permalink](#개발-과정)

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
[Permalink](#사용-방법)

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
[Permalink](#한계와-개선-아이디어)

- 🔒 사용자 제스처 제약: 브라우저 보안상 일부 클립보드 읽기는 제스처가 필요
- 🧾 Plain Text 중심: 이미지/리치 텍스트는 미지원
- 💾 내보내기: CSV/JSON 내보내기 기능 추가 예정
- 🗂️ 그룹핑: 도메인별/태그별로 묶어서 보기

---

## 6. 마무리
[Permalink](#마무리)

작지만 매일 쓰게 되는 유틸리티를 브라우저 확장으로 만들어 보았다. Manifest V3 환경에서 Service Worker, Content Script, Popup 흐름을 온전히 구현했고, 실제 사용성(검색/핀/단축키/중복 제거/Brave 호환성)을 꼼꼼히 보강했다.

- GitHub Repo: https://github.com/esoyeon/clipboard-history-viewer
- Demo YouTube: https://www.youtube.com/watch?v=hfsTbGbjUAk
