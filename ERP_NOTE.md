# Accounting / ERP Abbreviations Dictionary

이 문서는 PurchaseOrder / Payments / Finance 기획과 구현을 읽을 때 필요한 회계 ERP 약어 사전이다.

"모든 약어"는 SAP, Oracle, Microsoft Dynamics, NetSuite, 국내 ERP, 회사 내부 용어까지 포함하면 제품별로 끝없이 늘어난다. 따라서 여기서는 현재 프로젝트의 PO / RP / Payment / Finance / 재고 / COGS 논의에 실질적으로 필요한 약어와 회계 ERP 실무에서 자주 쓰이는 표준 약어를 우선 정리한다.

## 현재 기획에서 특히 중요한 약어

| 약어 | 풀어쓴 말 | 한국어 의미 | 개발/기획 해석 |
|---|---|---|---|
| ERP | Enterprise Resource Planning | 전사적 자원 관리 | 구매, 재고, 회계, 영업, 생산 데이터를 하나의 업무 흐름으로 연결하는 시스템. |
| PO | Purchase Order | 발주서 | 공급사에 상품/자재/용역을 주문하는 문서. 현재 `PurchaseOrder`의 핵심 개념. |
| RP | Receiving Plan | 입고 계획 | PO를 실제 입고 단위로 나눈 계획. 현재 결제 금액 산정의 주요 기준. |
| GR | Goods Receipt | 입고 처리 | 물건이나 서비스가 실제로 들어왔음을 기록하는 단계. |
| IR | Invoice Receipt | 매입 세금계산서/청구서 수취 | 공급사의 청구서를 받는 단계. ERP에서는 GR과 매칭된다. |
| GR/IR | Goods Receipt / Invoice Receipt | 입고/송장 대기 계정 | 입고는 되었지만 청구서가 아직 없거나, 청구서는 있는데 입고가 덜 된 차이를 관리하는 clearing 개념. |
| AP | Accounts Payable | 매입채무 | 공급사에게 지급해야 할 돈. PO/RP/Payment 흐름의 회계상 결과물. |
| AR | Accounts Receivable | 매출채권 | 고객에게 받을 돈. Payments의 공급사 지급과는 반대편. |
| COGS | Cost of Goods Sold | 매출원가 | 판매된 상품에 대응되는 원가. SKU 영업이익 계산의 핵심. |
| BOM | Bill of Materials | 자재 명세서 | SKU를 만들기 위해 필요한 원재료/부자재 구성표. |
| SKU | Stock Keeping Unit | 재고 관리 단위 | 판매/입고/재고를 식별하는 상품 단위. |
| WIP | Work In Process | 재공품 | 생산 중이어서 완제품도 원재료도 아닌 중간 상태 재고. |
| GL | General Ledger | 총계정원장 | 모든 회계 전표가 집계되는 회계 원장. |
| JE | Journal Entry | 회계 전표 | 차변/대변으로 회계 사건을 기록하는 단위. |
| VAT | Value Added Tax | 부가가치세 | 매입/매출 세금. 현재 `amountWithoutTax`, `tax` 분리는 이 관점과 맞다. |
| P&L | Profit and Loss Statement | 손익계산서 | 매출, 원가, 비용, 이익을 보여주는 재무제표. |
| BS | Balance Sheet | 재무상태표 | 자산, 부채, 자본 상태를 보여주는 재무제표. |
| IS | Income Statement | 손익계산서 | P&L과 같은 의미로 쓰이는 경우가 많다. |
| CF | Cash Flow | 현금흐름 | 실제 돈의 유입/유출. 지급 완료와 회계 인식은 다를 수 있다. |
| OPEX | Operating Expense | 운영비/판관비 | 상품 원가가 아니라 운영을 위해 발생한 비용. |
| CAPEX | Capital Expenditure | 자본적 지출 | 비용이 아니라 자산으로 처리되는 지출. |
| MOQ | Minimum Order Quantity | 최소 발주 수량 | supplier/material/sku quotation 또는 master에서 중요한 발주 제약. |
| MRP | Material Requirements Planning | 자재 소요 계획 | 수요와 BOM을 기반으로 필요한 자재와 발주 시점을 계산. |
| WMS | Warehouse Management System | 창고 관리 시스템 | 입고, 출고, 재고 위치/수량을 관리하는 시스템. |
| OMS | Order Management System | 주문 관리 시스템 | 주문 생성, 변경, 취소, 출고 요청을 관리하는 시스템. |
| SCM | Supply Chain Management | 공급망 관리 | 구매, 생산, 물류, 재고 흐름 전체. |

## 구매 / 발주 / 입고

| 약어 | 풀어쓴 말 | 한국어 의미 | 설명 |
|---|---|---|---|
| PR | Purchase Requisition | 구매 요청 | 내부 사용자가 구매 필요를 올리는 요청. 승인 후 PO가 된다. |
| PO | Purchase Order | 발주서 | 공급사에 보내는 공식 주문 문서. |
| PPO | Planned Purchase Order | 계획 발주 | MRP가 만든 예정 발주. 확정 전 상태. |
| RFQ | Request for Quotation | 견적 요청 | 공급사에 가격/조건 견적을 요청하는 문서. |
| RFP | Request for Proposal | 제안 요청 | 단순 가격보다 솔루션/제안까지 요청하는 문서. |
| RFI | Request for Information | 정보 요청 | 공급사 역량/정보 확인 목적의 요청. |
| SO | Sales Order | 판매 주문 | 고객에게 판매하기 위한 주문. PO와 반대편. |
| STO | Stock Transfer Order | 재고 이전 주문 | 창고/법인/플랜트 간 재고 이동 주문. |
| ASN | Advanced Shipping Notice | 사전 출하 통지 | 공급사가 출하 전에 보내는 예정 입고 정보. |
| GR | Goods Receipt | 입고 처리 | 구매한 물품이 실제로 들어온 것을 기록. |
| GI | Goods Issue | 출고 처리 | 재고가 창고에서 빠져나가는 것을 기록. |
| RTV | Return to Vendor | 공급사 반품 | 입고 후 불량/초과 등으로 공급사에 반품. |
| RMA | Return Merchandise Authorization | 반품 승인 | 고객/공급사 반품을 승인하고 추적하는 번호/절차. |
| POD | Proof of Delivery | 배송 완료 증빙 | 실제 배송 완료를 증명하는 문서. |
| BOL | Bill of Lading | 선하증권/운송장 | 운송 계약과 화물 수령을 증명하는 물류 문서. |
| MOQ | Minimum Order Quantity | 최소 발주 수량 | 이 수량 미만으로 발주 불가. |
| EOQ | Economic Order Quantity | 경제적 주문량 | 보관비와 주문비를 고려한 이론적 최적 발주량. |
| LT | Lead Time | 리드타임 | 주문부터 입고/완료까지 걸리는 시간. |
| SLA | Service Level Agreement | 서비스 수준 계약 | 납기, 품질, 응답시간 등 약속된 서비스 기준. |
| OTIF | On Time In Full | 정시 완전 납품 | 약속 날짜에 약속 수량을 모두 납품했는지. |
| ETA | Estimated Time of Arrival | 예상 도착 시간 | 입고/도착 예정 시각. |
| ETD | Estimated Time of Departure | 예상 출발 시간 | 출고/출발 예정 시각. |

## 매입채무 / 지급 / 정산

| 약어 | 풀어쓴 말 | 한국어 의미 | 설명 |
|---|---|---|---|
| AP | Accounts Payable | 매입채무 | 공급사에게 지급해야 할 금액. |
| AR | Accounts Receivable | 매출채권 | 고객에게 받을 금액. |
| IR | Invoice Receipt | 송장/세금계산서 수취 | 공급사 청구 문서를 받는 단계. |
| INV | Invoice | 송장/청구서 | 지급 요청의 증빙 문서. 국내에서는 세금계산서/계산서와 연결된다. |
| CN | Credit Note | 차감 전표 | 공급사가 금액을 줄여주는 문서. 반품/할인/오청구 정정에 사용. |
| DN | Debit Note | 추가 청구 전표 | 추가 금액 청구 또는 조정에 사용. |
| 2WM | Two-Way Match | 2자 매칭 | PO와 Invoice 금액/수량을 비교. |
| 3WM | Three-Way Match | 3자 매칭 | PO, GR, Invoice를 비교. 매입채무 통제의 핵심. |
| 4WM | Four-Way Match | 4자 매칭 | PO, GR, Invoice에 검수/품질 결과까지 매칭. |
| GR/IR | Goods Receipt / Invoice Receipt | 입고/송장 대기 | 입고와 청구의 차이를 clearing하는 계정/프로세스. |
| APP | Approval | 승인 | 지급 요청 또는 PO의 승인 단계. |
| EFT | Electronic Funds Transfer | 전자 자금 이체 | 은행 이체 방식. |
| ACH | Automated Clearing House | 미국식 자동 이체망 | 미국 지급/수금 인프라. 국내에서는 직접 대응하지 않을 수 있다. |
| IBAN | International Bank Account Number | 국제 은행 계좌 번호 | 해외 송금 계좌 식별 번호. |
| SWIFT | Society for Worldwide Interbank Financial Telecommunication | 국제 은행 식별/송금망 | 해외 송금 시 은행 식별에 사용. BIC와 같이 쓰인다. |
| BIC | Bank Identifier Code | 은행 식별 코드 | SWIFT 코드와 같은 맥락. |
| FX | Foreign Exchange | 외환 | 외화 지급/환율 처리. |
| T/T | Telegraphic Transfer | 전신환 송금 | 해외 송금 방식. |
| LC / L/C | Letter of Credit | 신용장 | 무역 거래에서 은행이 지급을 보증하는 문서. |
| COD | Cash on Delivery | 착불/인도시 지급 | 물품 인도 시 지급. |
| CIA | Cash in Advance | 선지급 | 납품 전 지급. |
| EOM | End of Month | 월말 | 결제 조건에서 월말 기준 계산에 사용. |
| NET 30 | Net 30 Days | 30일 후 지급 | 청구일/입고일 등 기준일 이후 30일 지급 조건. |
| DPO | Days Payable Outstanding | 매입채무 회전일수 | 공급사 지급까지 평균 며칠 걸리는지. |

## 회계 원장 / 재무제표

| 약어 | 풀어쓴 말 | 한국어 의미 | 설명 |
|---|---|---|---|
| GL | General Ledger | 총계정원장 | 회계 전표가 최종 집계되는 장부. |
| SL | Subledger | 보조원장 | AP, AR, 재고, 고정자산 등 상세 장부. |
| COA | Chart of Accounts | 계정과목표 | 계정 코드와 계정 체계. |
| JE | Journal Entry | 회계 전표 | 차변/대변 회계 기록. |
| DR | Debit | 차변 | 자산/비용 증가, 부채/자본/수익 감소 방향. |
| CR | Credit | 대변 | 부채/자본/수익 증가, 자산/비용 감소 방향. |
| TB | Trial Balance | 시산표 | 차변/대변 잔액 검증표. |
| BS | Balance Sheet | 재무상태표 | 자산, 부채, 자본. |
| IS | Income Statement | 손익계산서 | 수익, 비용, 이익. |
| P&L | Profit and Loss Statement | 손익계산서 | IS와 같은 의미로 쓰이는 경우가 많다. |
| CF | Cash Flow | 현금흐름 | 영업/투자/재무 현금흐름. |
| CFS | Cash Flow Statement | 현금흐름표 | 현금흐름 재무제표. |
| OCI | Other Comprehensive Income | 기타포괄손익 | 당기손익에 바로 들어가지 않는 포괄손익. |
| BSPL | Balance Sheet / Profit and Loss | 재무상태표/손익 | 재무제표 묶음을 지칭할 때 쓰는 약식 표현. |
| FS | Financial Statements | 재무제표 | BS, IS/P&L, CF 등. |
| FY | Fiscal Year | 회계연도 | 회사가 정한 회계 기준 연도. |
| FP | Fiscal Period | 회계기간 | 월/분기 등 회계 마감 단위. |
| YE | Year End | 연말 | 연말 결산. |
| ME | Month End | 월말 | 월말 결산. |
| TB | Trial Balance | 시산표 | 결산 전 잔액 검증. |
| AA | Account Assignment | 계정 지정 | 비용/자산/부서/프로젝트 등 회계 귀속 지정. |
| CC | Cost Center | 코스트센터 | 비용을 귀속하는 조직 단위. |
| PC | Profit Center | 손익센터 | 수익/비용/이익을 관리하는 책임 단위. |
| IO | Internal Order | 내부 오더 | 특정 프로젝트/캠페인/작업 단위 비용 추적. |

## 원가 / 재고 / 제조

| 약어 | 풀어쓴 말 | 한국어 의미 | 설명 |
|---|---|---|---|
| COGS | Cost of Goods Sold | 매출원가 | 팔린 재고에 대응되는 원가. |
| COGM | Cost of Goods Manufactured | 제조원가 | 특정 기간에 완성된 제품의 제조 원가. |
| BOM | Bill of Materials | 자재 명세서 | 완제품을 만드는 구성품과 수량. |
| BOO | Bill of Operations | 공정 명세서 | 제조 공정/작업 단계 정보. |
| BOP | Bill of Process | 공정/프로세스 명세 | 제조 프로세스 정의. |
| Routing | Routing | 공정 경로 | 어떤 작업장/공정을 어떤 순서로 거치는지. 약어는 아니지만 ERP 원가에서 중요. |
| WIP | Work In Process | 재공품 | 생산 중인 재고. |
| RM | Raw Material | 원재료 | 생산 투입 전 재료. |
| PM | Packaging Material | 포장재 | 포장용 부자재. SAP에서는 Plant Maintenance로도 쓰이므로 문맥 주의. |
| FG | Finished Goods | 완제품 | 판매 가능한 완성 재고. |
| SFG | Semi-Finished Goods | 반제품 | 완제품 전 단계 재고. |
| MRO | Maintenance, Repair, and Operations | 유지보수/운영 소모품 | 생산 원재료가 아닌 운영 소모품. |
| FIFO | First In, First Out | 선입선출 | 먼저 들어온 재고가 먼저 판매된 것으로 원가 계산. |
| LIFO | Last In, First Out | 후입선출 | 나중에 들어온 재고가 먼저 판매된 것으로 계산. IFRS에서는 허용되지 않는다. |
| WAC | Weighted Average Cost | 가중평균원가 | 재고 평균 단가로 원가 계산. |
| MAP | Moving Average Price | 이동평균단가 | 입고 시마다 갱신되는 평균 단가. |
| STD | Standard Cost | 표준원가 | 미리 정한 기준 원가. 실제 원가와 차이는 variance로 관리. |
| NRV | Net Realizable Value | 순실현가능가치 | 예상 판매가에서 판매 관련 비용을 뺀 금액. 재고 평가에 사용. |
| PPV | Purchase Price Variance | 구매가격차이 | 표준/예상 가격과 실제 구매 가격 차이. |
| IPV | Invoice Price Variance | 송장가격차이 | PO/입고 가격과 invoice 가격 차이. |
| Usage Variance | Usage Variance | 사용량 차이 | 표준 BOM 수량과 실제 투입 수량 차이. |
| Yield Variance | Yield Variance | 수율 차이 | 예상 산출량과 실제 산출량 차이. |
| Scrap | Scrap | 폐기/불량 손실 | 생산/검수 중 손실. |
| ATP | Available to Promise | 약속 가능 재고 | 현재/예정 재고로 고객에게 약속 가능한 수량. |
| CTP | Capable to Promise | 생산 가능 약속 수량 | 생산 능력까지 고려한 약속 가능 수량. |
| Safety Stock | Safety Stock | 안전재고 | 수요/공급 변동에 대비한 최소 재고. |
| ROP | Reorder Point | 재주문점 | 재고가 이 수준 아래로 떨어지면 재발주. |
| MRP | Material Requirements Planning | 자재 소요 계획 | BOM과 수요로 자재 필요량 계산. |
| MPS | Master Production Schedule | 기준 생산 계획 | 어떤 완제품을 언제 얼마나 생산할지. |
| CRP | Capacity Requirements Planning | 능력 소요 계획 | 작업장/설비/인력 능력 검토. |

## 세금 / 규정 / 회계기준

| 약어 | 풀어쓴 말 | 한국어 의미 | 설명 |
|---|---|---|---|
| VAT | Value Added Tax | 부가가치세 | 국내 매입/매출 세금 처리의 핵심. |
| GST | Goods and Services Tax | 재화용역세 | 일부 국가의 부가세 성격 세금. |
| WHT | Withholding Tax | 원천징수세 | 지급 시 원천적으로 떼는 세금. |
| CIT | Corporate Income Tax | 법인세 | 법인 소득에 대한 세금. |
| PIT | Personal Income Tax | 개인소득세 | 개인 소득에 대한 세금. |
| NTS | National Tax Service | 국세청 | 한국 국세청. |
| IFRS | International Financial Reporting Standards | 국제회계기준 | 한국 상장사 등에서 사용하는 국제 기준. |
| K-IFRS | Korean International Financial Reporting Standards | 한국채택국제회계기준 | 국내 채택 IFRS. |
| GAAP | Generally Accepted Accounting Principles | 일반적으로 인정된 회계원칙 | 보통 US GAAP 등 국가별 회계기준을 지칭. |
| IAS | International Accounting Standard | 국제회계기준 구버전 명칭 | IAS 2는 재고자산 기준. |
| SOX | Sarbanes-Oxley Act | 사베인스-옥슬리법 | 내부통제/재무보고 통제와 관련된 미국 법. |
| ICFR | Internal Control over Financial Reporting | 재무보고 내부통제 | 결산/승인/권한/감사 추적 통제. |

## 재무 지표

| 약어 | 풀어쓴 말 | 한국어 의미 | 설명 |
|---|---|---|---|
| GM | Gross Margin | 매출총이익률 또는 매출총이익 | 매출에서 COGS를 뺀 이익 또는 그 비율. |
| GP | Gross Profit | 매출총이익 | 매출 - 매출원가. |
| OP | Operating Profit | 영업이익 | 매출총이익에서 판관비 등을 뺀 이익. |
| EBITDA | Earnings Before Interest, Taxes, Depreciation and Amortization | 이자/세금/감가상각 전 이익 | 현금창출력 지표로 자주 사용. |
| EBIT | Earnings Before Interest and Taxes | 이자/세금 전 이익 | 영업성과 비교 지표. |
| NI | Net Income | 당기순이익 | 최종 순이익. |
| DSO | Days Sales Outstanding | 매출채권 회전일수 | 매출 후 현금 회수까지 평균 일수. |
| DPO | Days Payable Outstanding | 매입채무 회전일수 | 매입 후 지급까지 평균 일수. |
| DIO | Days Inventory Outstanding | 재고 보유일수 | 재고가 판매되기까지 평균 일수. |
| CCC | Cash Conversion Cycle | 현금전환주기 | DIO + DSO - DPO. |
| ROI | Return on Investment | 투자수익률 | 투자 대비 수익. |
| ROA | Return on Assets | 총자산이익률 | 자산 대비 이익. |
| ROE | Return on Equity | 자기자본이익률 | 자기자본 대비 이익. |

## ERP / 시스템 / 모듈

| 약어 | 풀어쓴 말 | 한국어 의미 | 설명 |
|---|---|---|---|
| ERP | Enterprise Resource Planning | 전사적 자원 관리 | 회계, 구매, 재고, 생산, 판매를 통합. |
| SCM | Supply Chain Management | 공급망 관리 | 수요, 구매, 생산, 물류, 재고. |
| WMS | Warehouse Management System | 창고 관리 시스템 | 창고 입출고/로케이션/재고 관리. |
| TMS | Transportation Management System | 운송 관리 시스템 | 배송/운송 계획과 운임 관리. |
| OMS | Order Management System | 주문 관리 시스템 | 주문 접수부터 출고 요청까지 관리. |
| MES | Manufacturing Execution System | 제조 실행 시스템 | 생산 현장 작업, 실적, 설비 데이터를 관리. |
| PLM | Product Lifecycle Management | 제품 수명주기 관리 | 제품 개발/설계/BOM 변경 관리. |
| CRM | Customer Relationship Management | 고객 관계 관리 | 고객, 영업, CS 관리. |
| EDI | Electronic Data Interchange | 전자문서교환 | PO, ASN, Invoice 등을 시스템 간 표준 문서로 교환. |
| API | Application Programming Interface | 애플리케이션 인터페이스 | 시스템 간 연동 인터페이스. |
| ETL | Extract, Transform, Load | 추출/변환/적재 | 데이터 웨어하우스 적재 방식. |
| ELT | Extract, Load, Transform | 추출/적재/변환 | 데이터 적재 후 변환 방식. |
| DW | Data Warehouse | 데이터 웨어하우스 | 분석용 통합 데이터 저장소. |
| BI | Business Intelligence | 비즈니스 인텔리전스 | 리포팅/분석 도구. |
| MDM | Master Data Management | 기준정보 관리 | 고객, 공급사, 상품, 계정과목 등 master data 관리. |
| IAM | Identity and Access Management | 인증/권한 관리 | 사용자 권한, 역할, 접근 제어. |
| RBAC | Role-Based Access Control | 역할 기반 접근 제어 | 역할별 권한 관리. |
| SSO | Single Sign-On | 통합 로그인 | 한 번 로그인으로 여러 시스템 접근. |
| SLA | Service Level Agreement | 서비스 수준 계약 | 시스템/업무 응답 수준 약속. |

## SAP에서 자주 보이는 모듈 약어

SAP 약어는 다른 ERP에서도 비슷한 도메인 이름으로 대응된다.

| 약어 | 풀어쓴 말 | 한국어 의미 | 설명 |
|---|---|---|---|
| FI | Financial Accounting | 재무회계 | GL, AP, AR, 결산 등. |
| CO | Controlling | 관리회계 | 코스트센터, 손익센터, 내부오더, 원가 배부. |
| MM | Materials Management | 자재관리 | 구매, 입고, 자재 master, 재고. |
| SD | Sales and Distribution | 영업/유통 | 판매 주문, 출하, 청구. |
| PP | Production Planning | 생산계획 | BOM, routing, 생산오더, MRP. |
| QM | Quality Management | 품질관리 | 검사, 품질 판정, 부적합. |
| PM | Plant Maintenance | 설비보전 | 설비 유지보수. Packaging Material 의미와 충돌할 수 있다. |
| WM | Warehouse Management | 창고관리 | SAP 구형 창고관리 모듈. |
| EWM | Extended Warehouse Management | 확장 창고관리 | SAP 고급 창고관리. |
| TM | Transportation Management | 운송관리 | 운송 계획/실행/운임. |
| HCM | Human Capital Management | 인사관리 | 인사, 급여, 조직. |
| HR | Human Resources | 인사 | HCM과 유사. |
| PS | Project System | 프로젝트 시스템 | 프로젝트 예산/원가/진척 관리. |
| AA | Asset Accounting | 고정자산회계 | 자산 취득, 감가상각, 처분. |
| TR | Treasury | 자금관리 | 현금, 은행, 자금 계획. |
| FSCM | Financial Supply Chain Management | 재무 공급망 관리 | 신용, 회수, 현금/채권 관리 등. |
| BW | Business Warehouse | 비즈니스 웨어하우스 | SAP 분석/데이터 웨어하우스. |
| BPC | Business Planning and Consolidation | 예산/연결 관리 | 계획, 예산, 연결 결산. |

## 국내 커머스/운영 문맥에서 같이 보이는 약어

| 약어 | 풀어쓴 말 | 한국어 의미 | 설명 |
|---|---|---|---|
| PG | Payment Gateway | 결제대행사 | 카드/간편결제 승인/매입/정산 연동사. |
| VAN | Value Added Network | 카드 승인 중계망 | 카드 승인/매입 중계 사업자. |
| PG fee | Payment Gateway Fee | 결제 수수료 | PG 정산에서 차감되는 비용. |
| CS | Customer Service | 고객지원 | 주문/클레임/환불 업무. |
| OMS | Order Management System | 주문 관리 시스템 | 커머스 주문 운영 시스템. |
| WMS | Warehouse Management System | 창고 관리 시스템 | 물류센터 입출고/재고. |
| SKU | Stock Keeping Unit | 재고 관리 단위 | 상품 식별 단위. |
| LOT | Lot | 로트 | 같은 조건으로 생산/입고된 묶음. 엄밀히 약어는 아니지만 실무상 대문자로 자주 표기. |
| B2B | Business to Business | 기업 간 거래 | 공급사/도매/법인 거래. |
| B2C | Business to Consumer | 기업-소비자 거래 | 일반 고객 판매. |
| D2C | Direct to Consumer | 소비자 직접 판매 | 브랜드가 고객에게 직접 판매. |
| SLA | Service Level Agreement | 서비스 수준 계약 | CS/물류 응답 기준. |
| KST | Korea Standard Time | 한국 표준시 | 한국 기준 시간. |
| UTC | Coordinated Universal Time | 협정 세계시 | DB/API 표준 시간으로 자주 사용. |

## 현재 코드/문서 약어 매핑

| 코드/문서 표현 | 의미 | 현재 프로젝트 해석 |
|---|---|---|
| `PO` | Purchase Order | `PurchaseOrder` 엔티티. SKU / MATERIAL / SUPPLIER 중 하나를 대상으로 한다. |
| `RP` | Receiving Plan | `ReceivingPlan` 엔티티. SKU/MATERIAL 입고 단위이며 payment 생성 기준이 된다. |
| `SKU` | Stock Keeping Unit | 완제품 또는 판매/재고 관리 단위. |
| `MAT` | Material | PO 코드 prefix에서 material 발주를 나타낸다. |
| `MATERIAL` | Material | 원료/부자재 발주 대상 및 cost category. |
| `SUP` | Supplier | PO 코드 prefix에서 supplier/용역성 발주를 나타낸다. |
| `BOM` | Bill of Materials | SKU와 Material 구성 관계. 현재는 사급/부자재 소요와 자동 발주 보조 모델에 가깝다. |
| `COGS` | Cost of Goods Sold | SKU 영업이익 계산에 필요한 매출원가. BOM 기반 SKU는 component cost roll-up이 필요하다. |
| `GR/IR` | Goods Receipt / Invoice Receipt | 입고와 청구 차이를 관리하는 ERP clearing 개념. 현재 코드에는 명시 모델이 없다. |
| `AP` | Accounts Payable | 공급사 지급채무. 현재 `Payment`는 AP invoice라기보다 운영 결제문서에 가깝다. |
| `VAT` | Value Added Tax | `amountWithoutTax`와 `tax` 분리의 회계적 배경. |
| `KLS`, `KCP`, `JSS` | Fulfillment / WMS provider code | 프로젝트 내부 물류/외부 시스템 식별자. 회계 ERP 표준 약어는 아니다. |
| `WDV-*` | Internal ticket or issue code | 와이즐리 내부 이슈/티켓 코드로 보인다. 회계 ERP 표준 약어는 아니다. |
| `O1` ~ `O8` | Objective / Outcome numbering | 기획안에서 목표 단계를 지칭하는 번호. 표준 ERP 약어는 아니다. |

## 혼동 주의

| 약어 | 충돌 의미 | 주의점 |
|---|---|---|
| PM | Plant Maintenance / Packaging Material / Project Manager | ERP 제조/설비에서는 Plant Maintenance, 자재에서는 Packaging Material, 조직에서는 Project Manager일 수 있다. |
| CO | Controlling / Company / Change Order | SAP에서는 관리회계. 제조/개발 문맥에서는 change order로도 쓰인다. |
| IR | Invoice Receipt / Interest Rate / Investor Relations | PO/AP 문맥에서는 Invoice Receipt. |
| GR | Goods Receipt / Goods Return | 구매 입고 문맥에서는 Goods Receipt. 반품 문맥에서는 goods return으로 쓰일 수 있다. |
| AA | Asset Accounting / Account Assignment | SAP FI에서는 고정자산회계, 구매 문맥에서는 계정 지정으로도 쓰인다. |
| PC | Profit Center / Personal Computer / Product Code | 회계에서는 손익센터. 상품 문맥에서는 product code일 수 있다. |
| CC | Cost Center / Credit Card / Carbon Copy | 회계에서는 코스트센터. 결제에서는 신용카드일 수 있다. |
| PP | Production Planning / Purchase Price | SAP 모듈은 생산계획, 원가 문맥에서는 구매가격. |
| INV | Invoice / Inventory | AP 문맥에서는 송장, 재고 문맥에서는 inventory 약칭일 수 있다. |

## 기획서에 넣으면 좋은 표현

아래처럼 첫 등장 시 약어를 풀어쓰면 개발자/기획자/회계 담당자 간 오해가 줄어든다.

```text
PO(Purchase Order, 발주서)
RP(Receiving Plan, 입고계획)
GR(Goods Receipt, 입고)
IR(Invoice Receipt, 매입 세금계산서/청구서 수취)
GR/IR(Goods Receipt / Invoice Receipt clearing, 입고-청구 대기 계정)
AP(Accounts Payable, 매입채무)
COGS(Cost of Goods Sold, 매출원가)
BOM(Bill of Materials, 자재명세서)
```

## 설계 판단에 직접 영향을 주는 약어

| 약어 | 왜 중요한가 |
|---|---|
| PO | 발주 승인과 단가 확정의 원천 문서다. |
| RP | 실제 지급 금액을 입고 수량 기준으로 만들려면 RP가 payment 생성 단위가 된다. |
| GR | 입고가 발생해야 재고와 GR/IR가 생긴다. |
| IR | 청구서가 들어와야 AP가 확정된다. |
| GR/IR | 입고와 청구가 다를 수 있음을 시스템 모델로 인정해야 한다. |
| AP | Payment와 회계상 매입채무를 같은 것으로 볼지 분리할지 결정해야 한다. |
| VAT | 세전금액과 세금을 분리하지 않으면 재고원가/세금채권/지급액이 섞인다. |
| COGS | SKU 영업이익을 내려면 판매된 lot의 원가를 알아야 한다. |
| BOM | 사급/임가공/BOM 기반 SKU는 완제품 단가만으로 COGS를 계산하면 안 된다. |
| WIP | 생산 중 재고를 추적하려면 원재료 차감과 완제품 입고 사이 상태가 필요하다. |
| GL | 운영 결제문서와 회계 전표를 언제 연결할지 결정해야 한다. |
| COA | 비용/자산/세금/채무 계정 매핑이 필요하다. |

