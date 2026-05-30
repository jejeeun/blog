---
title: 트랙레코드
icon: fas fa-chart-pie
order: 2
layout: page
permalink: /track-record/
---

<p class="feed-intro">매수가·목표가를 미리 공개한 리포트의 성과 추적. 개인 계좌 기준이며 모든 거래는 사후가 아닌 사전 공시됩니다.</p>

<div style="margin:1.1rem 0;padding:.75rem 1rem;border-left:4px solid #e0a72b;background:rgba(224,167,43,.12);border-radius:6px;font-size:.92rem;line-height:1.5;">
⚠️ <strong>예시 데이터입니다.</strong> 이 페이지의 모든 수치는 화면 구성을 위한 데모이며, 실제 운용 성과가 아닙니다.
</div>

<div class="stat-grid" style="margin-top:1.4rem">
  <div class="stat-card">
    <div class="stat-label">누적 수익률</div>
    <div class="stat-value up">+12.4%</div>
    <div class="stat-foot">2026 YTD · 개인 계좌</div>
  </div>
  <div class="stat-card">
    <div class="stat-label">vs KOSPI</div>
    <div class="stat-value up">+6.8%p</div>
    <div class="stat-foot">초과 수익 (alpha)</div>
  </div>
  <div class="stat-card">
    <div class="stat-label">샤프</div>
    <div class="stat-value">1.42</div>
    <div class="stat-foot">연환산</div>
  </div>
  <div class="stat-card">
    <div class="stat-label">MDD</div>
    <div class="stat-value down">-4.8%</div>
    <div class="stat-foot">최대 낙폭</div>
  </div>
</div>

<h2 class="section-h" style="margin-top:2.4rem">보유 종목</h2>

<div class="tr-table-wrap">
<table class="tr-table">
  <thead>
    <tr>
      <th>종목</th>
      <th>코드</th>
      <th class="num">매수가</th>
      <th class="num">현재가</th>
      <th class="num">목표가</th>
      <th class="num">평가손익</th>
      <th class="num">비중</th>
      <th>리포트</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>메리츠금융지주</td><td>139130</td>
      <td class="num">88,000</td><td class="num">91,200</td><td class="num">105,000</td>
      <td class="num up">+3.6%</td><td class="num">5.0%</td>
      <td><a href="{{ '/posts/139130-meritz-financial/' | relative_url }}">보기</a></td>
    </tr>
    <tr>
      <td>KB금융</td><td>105560</td>
      <td class="num">79,200</td><td class="num">80,500</td><td class="num">98,000</td>
      <td class="num up">+1.6%</td><td class="num">7.5%</td>
      <td><a href="{{ '/posts/105560-kb-financial/' | relative_url }}">보기</a></td>
    </tr>
    <tr>
      <td>삼성전자</td><td>005930</td>
      <td class="num">76,500</td><td class="num">78,000</td><td class="num">92,000</td>
      <td class="num up">+2.0%</td><td class="num">6.0%</td>
      <td><a href="{{ '/posts/005930-samsung-electronics/' | relative_url }}">보기</a></td>
    </tr>
    <tr class="tr-total">
      <td colspan="6">현금 비중</td>
      <td class="num">81.5%</td>
      <td></td>
    </tr>
  </tbody>
</table>
</div>

<p class="stat-note" style="margin-top:1rem">※ 데모 데이터입니다. 실제 운영 단계에서는 <code>_data/track_record.yml</code>과 GitHub Actions로 자동 갱신됩니다.</p>

<h2 class="section-h" style="margin-top:2.4rem">월별 수익률</h2>

<div class="tr-table-wrap">
<table class="tr-table">
  <thead>
    <tr><th>월</th><th class="num">포트</th><th class="num">KOSPI</th><th class="num">초과</th></tr>
  </thead>
  <tbody>
    <tr><td>2026.01</td><td class="num up">+2.1%</td><td class="num">-0.3%</td><td class="num up">+2.4%p</td></tr>
    <tr><td>2026.02</td><td class="num up">+3.4%</td><td class="num up">+1.2%</td><td class="num up">+2.2%p</td></tr>
    <tr><td>2026.03</td><td class="num down">-1.5%</td><td class="num down">-2.1%</td><td class="num up">+0.6%p</td></tr>
    <tr><td>2026.04</td><td class="num up">+4.8%</td><td class="num up">+3.1%</td><td class="num up">+1.7%p</td></tr>
    <tr><td>2026.05</td><td class="num up">+3.2%</td><td class="num up">+3.3%</td><td class="num down">-0.1%p</td></tr>
  </tbody>
</table>
</div>
