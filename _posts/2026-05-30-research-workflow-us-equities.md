---
title: "미국 주식 분석 실무 9단계 — SEC EDGAR에서 의사결정까지"
description: S&P 500 종목을 보는 9개의 질문. 한국 주식과 다른 미국 시장만의 데이터 자산(10-K Risk Factors · 13F · Form 4 · Fed dot plot)을 중심으로.
author: equity-analyst
date: 2026-05-30 11:00:00 +0900
categories: [투자공부, 종목분석법]
tags: [리서치워크플로우, 실무, 미국주식, 펀더멘털분석, 데이터소스]
---

## 들어가며

[종목 분석 실무 9단계]({% post_url 2026-05-30-research-workflow-data-to-insight %})의 미국 주식 버전. 구조는 같고 — 9개 질문에 순서대로 답을 채워가는 일 — 데이터 출처와 해석 기준이 다르다.

미국 시장이 한국 대비 갖는 자산:

| | 한국 | 미국 |
|---|---|---|
| 공시 | DART (텍스트 위주) | SEC EDGAR (Risk Factors가 substantive, 변호사가 강제) |
| 컨센서스 | 한경 컨센서스 | 다층 (Bloomberg/FactSet 전문가, Seeking Alpha 분석가, FinTwit) |
| 펀드 보유 현황 | 공시 부분적 | **13F filings** — 분기마다 hedge fund·뮤추얼펀드 보유 종목 공개 |
| 임원 매매 | 임원 5% 신고 정도 | **Form 4** — CEO·CFO 매수·매도 실시간 |
| 어닝스 콜 | 일부 운용사 access | **모든 종목의 transcript 무료 공개** (Tikr·Seeking Alpha) |

이 다섯 가지가 미국 주식 분석에서 한국과 결정적으로 다른 무기들이다. 각 단계마다 어떻게 활용하는지 짚어둔다.

> 각 단계 5라인: 핵심 질문 / 보는 데이터 / 어디서 어떻게 / 해석 기준 / 보고서 활용
{: .prompt-tip }

---

## 1. 종목 발굴 — "들여다볼 가치가 있나"

미국 상장사만 6,000개+. 정량 스크린으로 100개 이하로 좁히는 게 출발점.

**핵심 질문:** 펀더멘털·밸류에이션상 한 번 더 깊이 볼 가치가 있는가?

**보는 데이터:** Market cap, FCF margin, ROIC, EPS growth, multiples, sector

**어디서·어떻게:**
- **Finviz.com → Screener** — 30+ 필터 (시총·PE·sector·country·exchange·dividend·short)
- **Stockanalysis.com → Stock Screener** — 무료, 5년 financials 포함
- **TIKR.com → Screener** — fundamentals + 컨센서스 estimates 통합
- **Simply Wall St** — 시각화된 fair value snapshot (빠른 1차 스캔)

**해석 기준:**
- 시총 $1B 이상 (mid-cap+) → 유동성 안전 (운용역 관점)
- ROIC > 15% (5y avg) + Net Debt/EBITDA < 2x → quality 후보
- FCF margin > 10% + 5년 revenue growth > 8% → growth quality
- PE < 5년 median + 컨센서스 EPS revisions up → value-momentum
- 미국은 SBC(stock-based comp) 빼고 본 "non-GAAP EPS"의 함정 주의 — diluted GAAP 위주로

**보고서 활용:** Watchlist 구성. 통과 후보 중 1편씩 깊이.

---

## 2. 사업 구조 — "어떻게 돈을 버나"

미국 회사는 **10-K의 Item 1**이 회사가 자기 입으로 적은 가장 권위 있는 사업 정의.

**핵심 질문:** 매출의 어디에서, 누구한테, 어떤 마진으로 들어오는가? 지역·세그먼트 분포는?

**보는 데이터:** Revenue segments by geography·product·customer, customer concentration

**어디서·어떻게:**
- **SEC EDGAR → 종목 검색 → 10-K → Item 1. Business** — 사업 자기 정의 (15~30 페이지)
- **10-K → Item 2. Properties** — 시설·지리 분포
- **10-K → Item 8. Note "Segment Information"** — 세그먼트별 revenue·operating income 5년 시계열
- **Investor Presentation** (회사 IR 페이지) — 시각화 잘 돼있고 narrative 명확
- **TIKR / Koyfin** — segment revenue history 차트로
- **Earnings call transcripts** (Tikr 무료) — 임원이 어떤 사업을 강조하는지

**해석 기준:**
- 매출 70%+를 차지하는 1~2개 사업이 "실질적 회사" (mega-cap도 대개 한두 개 사업 라인이 매출의 대부분)
- **International revenue mix** → 환율·지정학 노출 (S&P 500 평균 ~40% 해외)
- **Customer concentration disclosure** — 10-K가 매출 10%+ 차지하는 고객을 의무 공개 (반드시 체크)
- "Segment Information" 주석에서 세그먼트 mix가 5년간 어떻게 변했는지 — 회사가 자기 선언 없이 보여주는 전략 방향

**보고서 활용:** 「기업 개요」 — 매출 세그먼트 표 + "이 회사는 ◯◯로 돈을 번다" 한 줄 정의.

---

## 3. 재무 정밀 — "자본배분의 질"

미국은 자사주 매입(buyback)·SBC 처리·M&A goodwill 변수가 한국보다 큼. 4개 숫자에 집중.

**핵심 질문:** 자본을 효율적으로 굴리는가, 그 효율이 지속되는가, 자본 environmentally 배분 의도는?

**보는 데이터 (4가지):**

| 지표 | 의미 | 위험 신호 |
|---|---|---|
| **ROIC** (5y avg) | 자본 1원당 수익 | < 8% (WACC 미달) = 가치 파괴 |
| **FCF conversion** (FCF/Net Income) | 회계 이익이 현금화 되는가 | < 80%면 회계 의심 |
| **SBC % of revenue** | 주식 보상이 매출 대비 얼마 | > 5%면 dilution 위험 (tech 주의) |
| **Buyback yield** (buyback - issuance) / cap | 진짜 EPS 자연 상승 폭 | + 면 주주환원, - 면 dilution |

**어디서·어떻게:**
- **SEC EDGAR → 10-K → Item 8. Financial Statements** (감사 받은 재무, 4개월 lag)
- **10-Q**에서 분기별 업데이트 (6주 lag)
- **Stockanalysis.com** — 종목 페이지 Financials 탭, 10년 데이터 깔끔
- **Macrotrends.net** — 20년+ 장기 시계열 차트 (사이클 종목 필수)
- **DEF 14A Proxy Statement** (SEC EDGAR) → 임원 보상·자사주 정책 의도
- **TIKR** → segment 단위까지 detail

**해석 기준:**
- 5년 ROIC > 15% 유지 = 진짜 좋은 비즈니스 (Buffett benchmark)
- **FCF conversion 80%+ + buyback yield positive** = 주주환원 견고
- SBC > 10% of revenue (tech 종목 흔함) → "non-GAAP EPS"의 거품. GAAP 기준으로 재계산 권장
- Goodwill > Tangible Book Value 큰 비율 → M&A 의존 성장 (impairment 리스크)
- 5년 OPM (operating margin) 변화 ±5%p 이상 흔들리면 사이클·경쟁 의심

**보고서 활용:** 「재무 분석」 — ROIC + FCF conversion + buyback yield 한 표.

---

## 4. 밸류에이션 — "지금 가격이 적절한가"

미국 시장은 글로벌 벤치마크 자체. Damodaran의 데이터셋이 가장 직접 적용된다.

**핵심 질문:** 이 회사의 적정 가치는 얼마이고, 지금 그 가치 대비 어디?

**보는 데이터:** P/E, P/S, EV/EBITDA, FCF yield, DCF (sensitivity 포함)

**어디서·어떻게:**
- **Damodaran Data** → `Updated Data` → **Industry Multiples** (PE, PB, EV/EBITDA, EV/Sales by industry, US 전용 sheet)
- **Damodaran Data** → **Risk Premiums for Equity** (ERP), **Cost of Capital by industry** (DCF input)
- **Stockanalysis.com** → 종목 5년 multiple bands (PE TTM, Forward PE, EV/EBITDA)
- **Koyfin** → 동종 비교 차트 (5+ stocks side-by-side)
- **Musings on Markets** (Damodaran 블로그) — DCF 케이스 스터디
- **Yahoo Finance** → Analyst Targets (median, high, low)
- **Tikr / Wisesheets** → 컨센서스 EPS 시계열 + revisions

**해석 기준:**
- Current PE < 5-year average + EPS still growing → 시장 misprice 가능
- Industry PE 대비 50%+ discount → 회사 고유 이슈 vs 시장 misprice 분리 검증
- **DCF는 단일 숫자 X. Bull/Base/Bear 3 시나리오 + WACC ±1% / terminal growth ±0.5% 민감도**
- **FCF Yield (FCF/Market Cap) > 5% + stable business + buyback** = 매수 후보 강함
- US large cap PE > 25x = expensive (역사 forward median ~16-17x)

**보고서 활용:** 「밸류에이션」 — 목표가 + 시나리오 + 민감도 표.

---

## 5. 컨센서스 차이 — "시장이 놓치는 것"

미국은 sell-side 커버리지가 한국 대비 5배+. 그래서 **variant perception 찾기가 더 어렵지만 데이터는 풍부**.

**핵심 질문:** 시장이 이 종목에 대해 무엇을 알고 있고, 무엇을 놓치고 있는가?

**보는 데이터:** Consensus EPS, target distribution, transcript themes, sell-side reports

**어디서·어떻게:**
- **Seeking Alpha** → analyst opinions (bull case + bear case multiple, 한 종목당 10+ 글)
- **TIKR / Wisesheets** → 컨센서스 EPS revisions 시계열 + sell-side rating breakdown
- **Tikr.com / Seeking Alpha** → **Earnings call transcripts** — 분기별 임원 발언 5년+ 시계열
- **FT, WSJ, Bloomberg.com** → 시장 narrative
- **Twitter/X FinTwit** — quality 분석가 계정 follow (산업·매크로별로 큐레이션)
- **Substack** — Net Interest (금융), The Diff (Byrne Hobart), Marc Rubinstein 등

**해석 기준:**
- 컨센서스 target distribution이 좁게 모이면 (예: median ±5% 안에 모두) → 시장 한쪽 쏠림. 반대 베팅 가치
- **Earnings call Q&A 세션에서 분석가의 어려운 질문에 대한 임원 답변 톤** = 자신감의 진짜 측정
- 임원이 반복하는 단어 = 회사가 전달하려는 narrative (시장이 이미 안다)
- 시장이 단기 quarter에만 집중하고 5년+ 구조 변화는 자주 놓침
- **Bear case write-ups (Seeking Alpha, Substack)을 읽고 반박 가능한지 자기 점검**

**보고서 활용:** 「투자 포인트」 — "시장이 X를 놓치고 있다 / 본인은 Y로 본다" 한 줄이 보고서의 차별성.

---

## 6. 수급 · Positioning — "스마트머니는 어디"

미국 시장의 결정적 자산: **13F filings**. 분기마다 $100M+ 운용 fund가 보유 종목 공개. + **Form 4**로 insider 실시간.

**핵심 질문:** 거대 펀드·임원이 어떻게 움직이고 있고, 그게 본인 thesis를 지지하는가?

**보는 데이터:** 13F holdings, Form 4 insider, short interest, ETF flows

**어디서·어떻게:**
- **WhaleWisdom.com / Fintel.io** → **13F filings** (분기마다 대형 헤지펀드·뮤추얼펀드 보유 종목 공개)
- **OpenInsider.com** → **Form 4 insider transactions** (CEO·CFO·이사 매수·매도 실시간, 거래 가격·수량·관계까지)
- **Fintel / S3 Partners** → Short interest data (float 대비 %)
- **ETF.com** → Sector ETF flows (산업 자금 흐름)
- **Finviz Heatmap** — 산업별 시각화

**해석 기준:**
- **Cluster of insider buying** (CEO+CFO+여러 이사 동시 매수) = 가장 강한 내부 신호. 1인 매수는 노이즈
- 단일 임원 매도 = 보통 노이즈 (세금·이혼·diversification 다양한 이유)
- **검증된 가치투자 펀드 다수가 4분기 연속 매수** = quality 신호 (한 곳보다 여러 곳이 동시일 때 강함)
- Short interest > 20% of float + ratio rising = 단기 변동성·squeeze 가능
- Sector ETF 자금 4주+ 유입 = 산업 모멘텀

**보고서 활용:** 「모멘텀·수급」 — Insider buying cluster + 13F 거장 보유 = 가장 강한 보조 근거.

---

## 7. 리스크 — "어디서 깨지나"

미국 10-K의 Item 1A는 **변호사가 강제로 적게 한** Risk Factors 섹션. 한국 사업보고서 대비 substantively 더 깊다.

**핵심 질문:** 이 thesis가 깨지는 조건은 무엇이고, 어떤 신호로 감지하나?

**보는 데이터:** Risk Factors, 8-K (material events), patent/litigation, competitive threats

**어디서·어떻게:**
- **SEC EDGAR → 10-K → Item 1A. Risk Factors** — 가장 substantive (보통 20~50 페이지)
- **전년 10-K Risk Factors와 비교** — 새로 추가된 리스크 = 회사가 새로 인지한 것
- **8-K filings** → "Material" events 실시간 (M&A, exec change, litigation, write-down)
- **DEF 14A Proxy** → 임원 보상·이사회 변화
- **경쟁자의 10-K Risk Factors도 같이 검토** — 같은 산업의 공통 위험
- **Oaktree Memos** (Howard Marks) → 사이클 위치 판단
- **PACER (법원 기록), patent tracker** → 소송 신호

**해석 기준:**
- **새로 추가된 Risk Factor** = 회사가 1년 내 인지한 새 위험 — 반드시 확인
- Risk Factor의 **순서** = 회사가 진심으로 우려하는 우선순위 (앞에 적은 게 더 큰 우려)
- 미국 동종이 적은 Risk Factor → 한국 동종에도 잠재 가능 (특히 지정학·기술·규제 항목)
- **Thesis-breaker는 수치로**: "Revenue growth가 2 quarters 연속 < 5% 이면 thesis 깨짐"
- **8-K가 발행되는 즉시 텍스트 읽기** — 시장 반응 전에 본인이 먼저 의미 해석

**보고서 활용:** 「리스크」 — 최소 3개 thesis-breaker를 수치로 정의.

---

## 8. 매크로 컨텍스트 — "Fed가 어디 가나"

미국은 Fed가 세계 매크로의 진앙. **FOMC dot plot + yield curve + DXY**가 가장 강한 매크로 신호.

**핵심 질문:** 시장 사이클이 어디에 있고, 그게 종목·산업 베타에 어떻게 작용하나?

**보는 데이터:** Fed funds rate, yield curve, dot plot, inflation, USD, market PE

**어디서·어떻게:**
- **FRED (St. Louis Fed)** → 시리즈 코드 직접:
  - **FEDFUNDS** (Fed funds rate)
  - **DGS10 / DGS2** (10y / 2y yield)
  - **T10Y2Y** (yield curve spread — 역사적 recession 선행지표)
  - **CPIAUCSL / CPILFESL** (CPI Headline / Core)
  - **DTWEXBGS** (Broad USD index)
  - **DGORDER** (durable goods, ISM 대용)
- **Federal Reserve** → **FOMC statements + Summary of Economic Projections (dot plot)** 분기 발표
- **Shiller PE Ratio** (multpl.com) — 시장 valuation 역사 위치
- **Bloomberg.com / WSJ / FT** — 매크로 narrative
- **Investing.com** — 캘린더 (FOMC, CPI 발표일)

**해석 기준:**
- **T10Y2Y < 0 (yield curve inversion)** → 역사적으로 6~18개월 내 recession (지난 7번 중 7번 적중, false signal 1번)
- Fed dot plot hawkish surprise → 성장주·long duration 자산 risk-off
- DXY > 105 → 미국 large cap의 해외 매출 压迫 (S&P 500 EPS guide 하향 흔함)
- **Shiller PE > 30** = expensive 역사 95th percentile (현재 30 ~ 35 사이면 신중)
- VIX > 25 = fear regime, 매수 기회 (단, 펀더멘털 confirm 후)

**보고서 활용:** 시황 보고서 본체. 종목 보고서에선 「산업 환경」 배경.

---

## 9. 트래킹 — "8-K 실시간 + 분기 transcript"

미국은 **SEC EDGAR RSS / API**로 보유 종목 8-K가 분 단위로 들어옴. 자동화 가치가 가장 큰 단계.

**핵심 질문:** 내 thesis가 시간에 따라 어떻게 검증되고, 어디서 깨질 신호가 보이나?

**보는 데이터:** 8-K (실시간), quarterly earnings, transcripts, Form 4 ongoing

**어디서·어떻게:**
- **SEC EDGAR → Company Search → 보유 종목 alerts** (RSS feed로 8-K 즉시)
- **SEC EDGAR API** (자동화) — 본인이 만들 boilerplate: 보유 종목 8-K filing → Slack 알림
- **Earnings Whispers / Zacks Earnings Calendar** → 발표 일정
- **Tikr / Seeking Alpha** → Conference call transcript 발표 즉시 (~보통 1시간 안)
- **OpenInsider** → 매수 후 insider 활동 지속 트래킹
- **WhaleWisdom** → 분기마다 거장 펀드 보유 변화

**해석 기준:**
- **Beat vs miss vs guide**: EPS beat 했어도 guide cut → 주가 하락 흔함. 양쪽 다 봐야
- 첫 quarter miss = 노이즈 가능. **2 quarters 연속 miss = thesis 점검 시작**
- **8-K "Departure of Directors or Certain Officers" (Item 5.02)** → CFO·CEO 사임 = 회사 내부 신호 (큰 우려)
- **8-K "Material Definitive Agreement" (Item 1.01)** → M&A·partnership 시그널
- Insider cluster buying that continues = thesis 강화

**보고서 활용:** 분기마다 follow-up 글 (Investment 카테고리). 매수가·목표가 대비 실제 흐름 공개 검증.

---

## 워크플로우 — 9단계 작업 체크리스트

한 종목을 처음부터 끝까지 적용할 때:

| 단계 | 산출물 | 데이터 출처 |
|---|---|---|
| 1. 발굴 | 정량 필터(시총·ROIC·FCF margin) 통과 워치리스트 1줄 | Finviz / Stockanalysis Screener |
| 2. 사업 구조 | 매출 세그먼트·지역 분해 + customer concentration 체크 | 10-K Item 1 + Segment Note |
| 3. 재무 정밀 | ROIC·FCF conversion·SBC·buyback yield 4 숫자 + DEF 14A로 자본정책 | 10-K Item 8 + Stockanalysis + DEF 14A |
| 4. 밸류에이션 | 5년 multiple 밴드 + 산업 평균(Damodaran) + DCF 시나리오 | Damodaran + Stockanalysis + Koyfin |
| 5. 컨센서스 차이 | 시장 narrative 요약 + bear case 정독 + 본인 variant 한 문단 | Seeking Alpha + Earnings transcripts |
| 6. 수급 | 13F 거장 펀드 보유 + Form 4 insider cluster + short interest | WhaleWisdom + OpenInsider + Fintel |
| 7. 리스크 | Risk Factors 신규 항목 식별 + thesis-breaker 3개 수치 정의 | 10-K Item 1A (전년과 비교) + 동종 10-K |
| 8. 매크로 | 사이클 위치 + Fed·USD·yield curve 영향 | FRED + FOMC dot plot |
| 9. 트래킹 | 분기 점검 항목 (실적 beat/miss, guide, 8-K 신호) | SEC EDGAR RSS + Earnings calendar |

→ 종목 보고서 한 편.

각 단계 30분~1시간, 합쳐서 **6~10시간이면 한 편**. 미국 종목은 데이터 양이 많아 한국보다 평균 +2시간.

---

## 매일 루틴 — 미국 시장은 시차 따로

한국 시간 기준:

| 시간 | 작업 | 사이트 |
|---|---|---|
| **아침 (15분)** | 전일 미국 종가, 시간외, 보유 종목 8-K | Yahoo Finance / SEC EDGAR RSS |
| **출근 후 (15분)** | 다가오는 FOMC·CPI 발표, 어닝스 캘린더 | Earnings Whispers + Investing |
| **점심 (15분)** | OpenInsider 매수 cluster · WhaleWisdom 13F 업데이트 (분기) | OpenInsider / WhaleWisdom |
| **저녁 (25분)** | 깊이 메모 (Net Interest · Musings on Markets · Stratechery 등) | 메모 사이트 |

요일별 분담:
- 월·수 — Insider + 13F
- 화·목 — 매크로 (FRED + Fed)
- 금 — 깊이 메모 1편 정독

---

## 한국 vs 미국 빠른 매핑

| 답하려는 질문 | 한국 1순위 | 미국 1순위 |
|---|---|---|
| 들여다볼 가치 있나? | KRX 스크리닝 | **Finviz / Stockanalysis Screener** |
| 어떻게 돈을 버나? | DART 사업보고서 II | **10-K Item 1 + Segment Note** |
| 자본 잘 굴리나? | DART 재무제표 | **10-K Item 8 + DEF 14A** |
| 가격이 적절한가? | KRX 5년 밴드 | **Damodaran Industry Multiples + 5y bands** |
| 시장이 놓치는 것? | 한경 컨센서스 | **Earnings call transcripts + Seeking Alpha bear cases** |
| 수급이 받쳐주나? | KRX 투자자별 | **13F (WhaleWisdom) + Form 4 (OpenInsider)** |
| 어디서 깨지나? | DART 위험 항목 | **10-K Item 1A Risk Factors** |
| 사이클 어디? | FRED + ECOS | **FRED + FOMC dot plot** |
| 시간 따라 검증? | DART 분기 + 주요사항 | **8-K filings (RSS / API) + transcripts** |

---

## 정리

미국 주식은 한국 대비 **데이터의 양과 substantively 깊이가 압도적**이다. 동시에 sell-side 커버리지가 더 많아 variant perception 찾기는 더 어렵다.

본질은 같다 — 9개 질문에 답한다. 도구만 다르다.

다음 — 한국·미국 종목 1편씩 차례로 작성해보면서 두 시장의 작업 차이를 직접 검증한다. 첫 종목은 사업 이해 진입 비용이 낮은 것 (단일 사업 dominant·본인이 사용자로서 친숙한 것)으로 고르는 게 학습 효율 ↑.

🍀
