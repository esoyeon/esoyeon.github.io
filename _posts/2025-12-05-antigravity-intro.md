---
layout: post
title: "[Review] 구글 Antigravity 사용기: 개발 지식 없이 웹사이트 만들기"
date: 2025-12-05 22:00:00 +0900
categories: [Review, Tools]
tags: [Antigravity, Google, AI, Productivity]
---

개발 경험이 없는 분들도 자신만의 웹사이트나 서비스를 만들고 싶어 하는 니즈는 항상 있어왔습니다. 최근 구글에서 공개한 **'Antigravity(안티그래비티)'**는 이러한 니즈를 겨냥한, 자연어 기반의 코딩 에이전트 도구입니다.

직접 사용해보니 단순히 코드를 짜주는 것을 넘어, 프로젝트 관리와 실행까지 도와주는 점이 인상적이었습니다. 이번 포스팅에서는 Antigravity의 설치부터 실제 간단한 **'디지털 명함'** 제작 과정까지 정리해 보았습니다.

---

## 1. 개요 및 설치

**Antigravity**는 사용자가 자연어(한국어 포함)로 명령을 내리면, AI 에이전트가 이를 해석하여 코드 작성, 파일 생성, 터미널 명령어 실행 등을 수행하는 도구입니다.

설치 과정은 일반적인 데스크톱 애플리케이션과 동일하게 간단압니다.

![Antigravity 설치 화면](/assets/images/posts_img/251205_googleantigravity_intro/GA_install1.png)

설치 파일을 다운로드하고 실행하면 위와 같은 환영 스플래시 화면을 볼 수 있습니다. 별도의 복잡한 환경 변수 설정 과정은 필요하지 않았습니다.

---

## 2. 초기 설정 가이드

첫 실행 시 몇 가지 설정 옵션이 제공되는데, 입문자에게 권장하는 세팅은 다음과 같습니다.

![Antigravity 설정 화면](/assets/images/posts_img/251205_googleantigravity_intro/GA_install2.png)

1.  **Development Style**: `Agent-assisted`
    *   사용자가 주도권을 갖되 AI가 보조하는 방식입니다. AI가 독단적으로 행동하지 않고 사용자의 의도를 확인하며 진행하므로 흐름을 파악하기 좋습니다.

2.  **Terminal Policy**: `Auto`
    *   패키지 설치나 서버 실행 등 터미널 명령어가 필요할 때, AI가 매번 승인을 요청하지 않고 자동으로 수행하도록 합니다. 번거로움을 줄여주어 쾌적한 작업이 가능합니다.

---

## 3. 인터페이스 살펴보기

Antigravity의 UI는 크게 두 가지 뷰로 나뉩니다.

### Manager View (매니저 뷰)
![매니저 뷰](/assets/images/posts_img/251205_googleantigravity_intro/GA_window4.png)
사용자가 AI에게 거시적인 작업을 지시하는 곳입니다. 채팅 인터페이스를 통해 "웹사이트를 만들어줘", "기능을 추가해줘"와 같은 명령을 내립니다.

### Editor View (에디터 뷰)
![에디터 뷰](/assets/images/posts_img/251205_googleantigravity_intro/GA_window5.png)
실제 코드가 작성되는 공간입니다. 좌측 탐색기에서 생성된 파일을 확인할 수 있고, 에디터 화면에서 실시간으로 코드가 작성되는 과정을 볼 수 있습니다.

참고로 작업 목적에 따라 사용할 **AI 모델**을 선택할 수도 있습니다.
![모델 선택](/assets/images/posts_img/251205_googleantigravity_intro/GA_window3.png)

---

## 4. 보안 기능 (.aignore 및 승인)

AI가 로컬 파일에 접근하는 것에 대한 우려를 해소하기 위해 몇 가지 보안 장치가 마련되어 있습니다.

![보안 설정](/assets/images/posts_img/251205_googleantigravity_intro/GA_window9.png)

*   **`.aignore`**: `gitignore`와 유사한 개념으로, 여기에 등록된 파일이나 폴더는 AI가 읽거나 수정할 수 없습니다. API 키나 개인 정보가 담긴 파일은 필수로 등록하는 것이 좋습니다.
*   **Approval (승인)**: 파일 삭제나 중요 설정 변경과 같이 파괴적인 작업은 `Auto` 모드에서도 사용자의 명시적인 승인(`Approve`)을 거쳐야만 실행됩니다.

---

## 5. 실습: 디지털 명함 제작

실제로 Antigravity를 활용해 간단한 **디지털 명함 웹페이지**를 제작해 보았습니다.

**요청 내용:**
> "모바일 친화적인 디지털 명함 페이지를 만들어줘. 다크 모드 배경(#121212)에 네온 블루 포인트 컬러를 사용하고, 프로필 사진과 SNS 링크 버튼 3개를 중앙에 배치해줘."

요청을 입력하자 AI가 작업 계획(Plan)을 수립하고, 필요한 `index.html`과 `style.css` 파일을 생성했습니다.

![작업 시작 전 화면](/assets/images/posts_img/251205_googleantigravity_intro/GA_window2.png)
*AI가 작업을 분석하고 게획을 세우는 모습*

**결과물:**
약 3분 정도의 시간이 소요되었으며, 별도의 수정 없이 요구한 디자인 시안이 구현되었습니다.

![AI가 만든 디지털 명함](/assets/images/posts_img/251205_googleantigravity_intro/result_screenshot.png)

---

## 6. 총평

Antigravity는 코딩 장벽을 획기적으로 낮춰주는 도구임이 분명합니다. 특히 **자연어 처리 능력**이 뛰어나 구체적인 기술 용어를 모르더라도 원하는 결과물을 얻을 수 있다는 점이 큰 장점입니다.

물론 복잡한 비즈니스 로직이 필요한 대규모 프로젝트에는 아직 검증이 필요하겠지만, 개인 포트폴리오 사이트나 간단한 MVP(Minimum Viable Product)를 제작하는 용도로는 충분히 현업에서도 활용 가능할 것으로 보입니다.

관심 있으신 분들은 한번 설치하여 경험해 보시기를 추천합니다.
