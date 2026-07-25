---
title: Projects
icon: fas fa-code
order: 2
---

<div class="portfolio-project-grid">
  <article class="portfolio-project-card portfolio-project-card--featured">
    <div class="portfolio-card-heading">
      <div><p class="portfolio-eyebrow">BACKEND · TRAVEL PLATFORM</p><h2>TripCrew</h2></div>
      <span class="portfolio-status">Backend Developer</span>
    </div>
    <p>관광지 탐색과 여행 일정 관리를 제공하는 여행 플랫폼</p>
    <div class="portfolio-metric-row" aria-label="성능 개선 결과">
      <span><strong>52ms</strong> 전체 조회 <small>506ms에서 개선</small></span>
      <span><strong>4ms</strong> 키워드 검색 <small>523ms에서 개선</small></span>
      <span><strong>34ms</strong> 카페 검색 <small>940ms에서 개선</small></span>
    </div>
    <h3>주요 담당</h3>
    <ul>
      <li>Spring Security와 JWT 기반 인증</li>
      <li>관광지 검색 및 필터 API와 MyBatis SQL 작성</li>
      <li>검색 및 목록 조회 성능 개선</li>
      <li>Redis Sorted Set 기반 랭킹</li>
      <li>외부 API 장애 대응 설계</li>
      <li>Flyway 데이터베이스 초기화와 Docker Compose 개발 환경 구성</li>
    </ul>
    <p class="portfolio-tech-list">Java · Spring Boot · Spring Security · MyBatis · MySQL · Redis · Flyway · Docker</p>
    <div class="portfolio-actions">
      <a class="portfolio-button" href="/posts/tripcrew-performance/">성능 개선 상세</a>
      <!-- TODO: 프로젝트 기술 글과 GitHub 저장소의 실제 URL을 확인한 뒤 버튼을 추가하세요. -->
    </div>
  </article>

  <article class="portfolio-project-card">
    <div class="portfolio-card-heading">
      <div><p class="portfolio-eyebrow">REALTIME GAME</p><h2>WebRTC 요트다이스</h2></div>
      <span class="portfolio-status portfolio-status--progress">개발 진행 중</span>
    </div>
    <p>휴대폰 모션 센서와 실시간 통신을 활용하는 멀티플레이 요트다이스 게임</p>
    <h3>담당 및 진행 중인 설계</h3>
    <ul>
      <li>서버 기준 라운드 상태와 시작·제출·종료 이벤트 관리</li>
      <li>deadline 기반 타이머 동기화와 중복 제출 방지</li>
      <li>재접속 시 게임 상태 복구</li>
      <li>WebSocket 상태 브로드캐스트와 동시성 제어 설계</li>
    </ul>
    <p>WebRTC는 미디어·센서 통신에 활용하고, 게임 상태와 시그널링에는 WebSocket을 함께 사용합니다.</p>
    <p class="portfolio-tech-list">Java · Spring Boot · WebSocket · WebRTC · Redis</p>
    <div class="portfolio-actions"><a class="portfolio-button" href="/posts/game-timer-deadline/">타이머 설계 글</a></div>
  </article>

  <article class="portfolio-project-card">
    <p class="portfolio-eyebrow">AI VOICE KIOSK</p>
    <h2>말랑오더</h2>
    <p>음성으로 탐색, 주문, 결제를 지원하는 AI 음성 주문 키오스크</p>
    <h3>담당</h3>
    <ul>
      <li>JWT 인증, 주문 CRUD, 백엔드 API</li>
      <li>React와 TypeScript 기반 사용자 화면</li>
      <li>Figma UI 설계 및 화면 개선</li>
    </ul>
    <p class="portfolio-tech-list">Java · Spring Boot · MyBatis · React · TypeScript · Figma</p>
  </article>

  <article class="portfolio-project-card">
    <p class="portfolio-eyebrow">CAREER MANAGEMENT</p>
    <h2>11JOB</h2>
    <p>구직 활동과 포트폴리오 자료를 관리하는 서비스</p>
    <h3>담당</h3>
    <ul>
      <li>사용자 인증과 설정</li>
      <li>포트폴리오 및 프로젝트 관리 API</li>
      <li>AWS S3 파일 업로드, URL 반환 및 예외 처리</li>
    </ul>
    <p class="portfolio-tech-list">Java · Spring Boot · MyBatis · AWS S3</p>
  </article>
</div>
