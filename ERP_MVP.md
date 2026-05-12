# PurchaseOrder / Payments ERP·회계 도메인 온보딩 노트

작성일: 2026-05-12  
대상 작업 원문: `.context/attachments/pasted_text_2026-05-12_21-14-23.txt`

이 문서는 첨부 기획을 구현하기 전에 개발자가 알아야 할 현재 `admin-backend` 구현과 ERP·회계 도메인 배경을 한곳에 모은 것이다. 법률·세무 판단은 최종적으로 재무/세무 담당자와 확인해야 한다.

## 1. 첨부 기획의 핵심

첨부 기획은 크게 두 단계다.

1. **Tier 1 - 매입 사이드 정상화**
   - O1: 승인된 정확한 단가로만 PO가 생성된다.
   - O2: 입고된 수량과 해당 단가대로만 매입채무와 지급이 진행된다.
   - O3: 재고 변동이 원자적이고 추적 가능한 transaction으로 기록된다.
2. **Tier 2 - 재무·회계 사이드 개발**
   - O4/O5/O7/O8: 매출, 비용, 현금, 회계 분류를 시스템에 내재화한다.
   - O6: 위 데이터를 합성해 SKU/전사 영업이익을 산출한다.

현재 코드에는 PO, 입고계획, 결제문서, 결제조건, 공급사 정산, 세금계산서/은행 거래 조회의 일부가 존재하지만, ERP식 GR/IR, AP 확정, 3-way matching, 회계전표/원장, 은행대사까지는 아직 하나의 통제 흐름으로 연결되어 있지 않다.

## 2. 현재 프로젝트 구현 지형

### 2.1 모듈 경계

- `src/purchase/`
  - v1 구매/발주 모듈이다.
  - `src/purchase/CLAUDE.md` 기준으로 **완전히 deprecated** 상태다. 신규 비즈니스 로직/엔드포인트는 추가하지 않고, 엔티티·enum·호환 DTO 정도만 수정한다.
  - 그래도 핵심 엔티티(`PurchaseOrder`, `ReceivingPlan`, `Lot`, `LotInventoryEvent`)와 payment-sync 컴포넌트는 여기 있다.
- `src/purchase-v2/`
  - 신규 조회/운영 API의 주 경로다.
  - 목록 조회는 QueryBuilder + no hydration, mutation은 repository/populate 기반으로 다룬다.
  - 신규 구매/입고 기능은 이 컨벤션에 맞춰 구현해야 한다.
- `src/payments/`
  - 공급업체 결제문서, 결제 마일스톤, 결제조건, 지급 승인/완료, 공급사 정산을 담당한다.
- `src/finance/`
  - Popbill 기반 홈택스 매입 세금계산서 조회, 은행 거래 조회, 계좌 확인, 조정문서 일부를 제공한다.
  - 현재 PO/Payment와 자동 matching되어 있지는 않다.

### 2.2 핵심 엔티티

#### Quotation

파일: `src/sku/entities/quotation.entity.ts`

현재 단가 마스터에 가장 가까운 엔티티다.

- 대상: `sku`, `material`, `supplier` 중 하나와 연결된다.
- 가격: `unitPrice`, `unitPriceIncludeTax`, `taxRate`, `currency`.
- 적용일: `applyDate`.
- 표시 금액: `unitPriceWithTax` getter가 `unitPriceIncludeTax`가 있으면 그 값을, 없으면 `unitPrice * (1 + taxRate)`를 반환한다.

기획의 R1.1과 비교하면 현재 빠진 개념이 많다.

- `validTo` 없음.
- 승인 상태 없음.
- `quotationType` 없음. `STANDARD / FREE_GOODS` 구분이 없다.
- `pricePolicy` 없음. `FIXED / VARIANCE / RANGE`와 자동승인 범위가 없다.
- 단가 변경 이력은 record 추가로 어느 정도 표현되지만, 만료·승인·정책 위반이 시스템 레벨에서 강제되지 않는다.

#### Sku / Material

파일: `src/sku/entities/sku.entity.ts`, `src/sku/entities/material.entity.ts`

- `quotations` 컬렉션을 갖는다.
- `getQuotationByDate(date)`는 `applyDate <= date` 중 가장 최신 견적을 고른다.
- `currentPurchaseCost*` getter는 현재 날짜 기준 Quotation을 사용한다.

주의할 점:

- 현재 구현은 `applyDate` 기반이다. 기획의 `validFrom <= posting <= validTo`와 다르다.
- `Material.currentPurchaseCost*`는 현재 quotation 정렬만 하고 `applyDate <= now` 필터가 없다. SKU의 `currentQuotation`보다 느슨하다.

#### PurchaseOrder

파일: `src/purchase/entity/purchase-order.entity.ts`

주요 필드:

- 대상: `sku`, `material`, `supplier`.
- 단가: `quotation`.
- 수량: `requestedQuantity`, `confirmedQuantity`.
- 결제조건: `paymentTerm`.
- 일정: `requestedAt`, `requestedArriveAt`, `confirmedAt`, `receivedAt`, `closedAt`.
- 연결: `receivingPlans`, `payments`.

계산 getter:

- `expectedTotalAmount`: `receivingPlan.(quantity || expectedQuantity) * quotation.unitPriceWithTax` 합계.
- `actualPaymentAmount`: RP에 연결된 `payments.amountWithTax` 합계.
- `totalAmountDifference`: expected - actual.

PO 타입:

- `SKU`
- `MATERIAL`
- `SUPPLIER`

PO 상태는 별도 컬럼이 아니라 날짜/입고계획 상태에서 파생된다. `PurchaseOrderStatus`는 `CREATED`, `REQUESTED`, `RECIEPT_RESERVED`, `RECEIPT_COMPLETED`, `CLOSED`를 가진다.

#### ReceivingPlan

파일: `src/purchase/entity/receiving-plan.entity.ts`

ReceivingPlan은 PO를 실제 입고 단위로 나누는 운영 객체다.

주요 필드:

- `purchaseOrder`
- `payments`
- 요청/예정/확정/출고/입고 날짜
- 요청/확정/입고 수량
- `wmsReceivingPlanId`, `wmsReceivingPlanMetadata`
- `lots`

상태는 날짜로 계산된다.

- `CREATED`
- `REQUESTED`
- `REQUEST_CONFIRMED`
- `SHIPPED`
- `INBOUND_RESERVED`
- `INBOUND_COMPLETED`

중요 도메인 메서드:

- `request()`: PO가 요청된 뒤에만 RP 요청 가능. expected date와 expected shipped date를 계산한다.
- `confirm()`: 공급사 확정.
- `reservePlan()`: 입고 예약. `reservedArriveAt`, `quantity`, `shippedAt`, `manufacturedAt` 설정.
- `receive()`: 입고 완료. 현재는 레거시 방식이라는 주석이 있고, `reservedArriveAt`이 필요하다.
- `addLot()`: +5% 허용 범위를 넘으면 거부한다.

#### Lot / LotInventoryEvent

파일: `src/purchase/entity/lot.entity.ts`, `src/purchase/entity/lot-inventory-event.entity.ts`

`Lot`은 실입고 묶음이다.

- `receivingPlan`
- `receivedAt`
- `receivedQuantity`
- `expiredAt`
- `lotInventoryEvents`

`LotInventoryEvent`는 재고 동기화용 outbox에 가깝다.

- `purchaseOrder`, `receivingPlan`, `lot`, `sku`
- `delta`
- `wmsContents`
- `expiredAt`
- `publishedAt`
- `sourceType` getter:
  - `RESERVED`: lot 없음.
  - `UNTRACKED`: lot만 있음, PO/RP 없음.
  - `PO_ONLY`: lot + PO, RP 없음.
  - `COMPLETED`: lot + RP + PO.

기획 O3의 "재고 transaction ledger"와 유사한 시작점이지만, 현재 목적은 commerce-web-api 재고 이벤트 발행이다. 회계 원가 layer, GL 전표, reason code, blocked stock까지는 별도 모델로 존재하지 않는다.

#### BillOfMaterials / LinkMaterials

파일: `src/sku/entities/bill-of-materials.entity.ts`, `src/sku/entities/link-materials.entity.ts`

현재 BOM은 SKU와 Material의 구성 관계를 표현한다.

- `BillOfMaterials`
  - `sku`: 완제품 또는 상위 SKU.
  - `name`.
  - `applyDate`.
  - `description`.
  - `links`.
- `LinkMaterials`
  - `bom`.
  - `start`: 상위 material. 없으면 SKU 바로 아래 단계.
  - `end`: 하위 material.
  - `componentQuantity`.
  - `unit`.

`Sku.hasSubcontracts`는 `billOfMaterials.links`가 있으면 true로 계산된다. 구매 PO 조회 테스트도 BOM에 material link가 있는 SKU를 "사급 있음"으로 노출하는지 확인한다.

또한 `src/demand-forecast/demand-forecast.service.ts`의 `getSubMaterialsForSkus()`는 BOM tree를 BFS로 순회해 SKU별 부자재를 모으고, material별 현재 quotation까지 붙인다. 주석상 PO 생성 시 부자재 자동 발주에 사용된다.

중요한 해석:

- 현재 BOM은 회계 원가 roll-up 모델이라기보다 사급/부자재 소요와 자동 발주 보조 모델에 가깝다.
- 하지만 SKU 영업이익, COGS, 생산/임가공/사급 원가를 정확히 다루려면 BOM은 핵심 master data가 된다.
- 현재 `BillOfMaterials.applyDate`는 있지만 유효 종료일, 승인 상태, BOM version, scrap/yield, 대체 자재, routing/공정비, overhead까지는 모델링되어 있지 않다.

#### Payment

파일: `src/payments/entity/payment.entity.ts`

현재 `Payment`는 "확정 AP invoice"라기보다 운영 결제문서다.

주요 필드:

- 연결: `purchaseOrder`, `receivingPlan`, `supplier`, `quotation`.
- 금액: `amountWithoutTax`, `tax`, `amountWithTax` getter.
- 통화/환율: `currency`, `exchangeRate`.
- 계좌: `bankAccount`, `bankName`, `bankAccountHolder`.
- 일정: `expectedPaymentDueDate`, `paymentDueDate`.
- 마일스톤: `paymentMilestone`.
- 요청/승인/지급: `requestedAt`, `firstApprovedAt`, `finalApprovedAt`, `paidAt`.
- 증빙: `supportingDocument`, `proofOfPayment`.
- 완료/반려: `closedAt`, `rejectedAt`.

상태는 timestamp getter로 계산된다.

- 요청됨: `requestedAt`.
- 1차 승인: `firstApprovedAt`.
- 최종 승인: `finalApprovedAt`.
- 승인 완료: 1차+최종 승인 모두 존재.
- 지급 완료: `paidAt`.
- 종결: `closedAt`.

`PaymentStatus` enum은 있지만 실제 상태 전이는 timestamp 조합 중심이다.

#### PaymentMilestone / PaymentTerm

파일:

- `src/payments/entity/payment-milestone.entity.ts`
- `src/payments/payment-term/enums/payment-term.enum.ts`
- `src/payments/payment-term/parsers/payment-term-parser.ts`
- `src/payments/payment-term/value-objects/payment-schedule.vo.ts`
- `src/payments/payment-term/calculators/payment-due-date-calculator.ts`

`PaymentTerm`은 PO에 저장되는 결제 조건 enum이다. 예:

- 발주일 기준 70% / 출고일 기준 30%
- 발주일 기준 50% / 입고일 기준 다음달 말일까지 50%
- 입고일 기준 다음달 15일까지 100%
- 입고일 기준 15일 분할 특수 케이스
- D-1 / D+2 같은 선결제 케이스

`PaymentTermParser`는 `PaymentTerm + totalAmountWithoutTax + taxRate`를 `PaymentSchedule[]`로 변환한다.

`PaymentSchedule`은 다음을 가진다.

- `milestone`: `PURCHASE_ORDER`, `DISPATCH`, `RECEIPT`, `RECEIPT_SPLIT_BY_15TH`
- `monthsAfterMilestone`
- `alignToMonthEnd`
- `alignToMonthStart`
- `daysAfterMilestone`
- `amountWithoutTax`
- `tax`
- `proportion`

`PaymentDueDateCalculator`는 milestone 기준일을 찾고, 전략에 따라 due date를 계산한다.

- `PURCHASE_ORDER`: PO 요청일.
- `DISPATCH`: 출고일.
- `RECEIPT`: 입고일.
- `RECEIPT_SPLIT_BY_15TH`: 입고일 1~15일이면 당월 말일, 16일 이후면 다음달 15일.

예상 지급일과 실제 지급일이 분리된다.

- 예상 지급일: 예상 출고/입고일 기반.
- 실제 지급일: 실제 출고/입고일 기반.

### 2.3 PO 생성과 단가 적용의 현재 동작

파일:

- `src/purchase/purchase-order.service.ts`
- `src/purchase/dto/purchase-order.mutation.dto.ts`

`CreatePurchaseOrderDto`는 `quotationId`를 필수로 받는다. `createBulk()`는 대상 SKU/Material/Supplier의 현재 적용 가능한 quotation을 `preloadQuotations()`로 조회하지만, 실제 PO에는 DTO의 `quotationId`가 들어간다.

중요한 현재 상태:

- API 레벨에서 단가 수기 우회가 완전히 차단되어 있지 않다.
- `UpdatePurchaseOrderDto`도 `quotationId`를 받을 수 있다.
- 기획의 "PO 생성 시 시스템이 승인된 단가를 자동 결정하고 단가 수정 필드는 없다"와 다르다.
- 결제 승인/지급된 payment는 payment sync에서 제외되지만, PO의 quotation 변경 자체를 회계 통제처럼 금지하는 모델은 아니다.

현재 적용 quotation 찾기 로직:

- `findAppliableQuotation()`은 `applyDate <= now`인 견적 중 가장 최신을 고른다.
- `validTo`, 승인상태, 견적 타입, 가격 정책 범위는 보지 않는다.

### 2.4 PurchaseOrder Payment Sync

파일: `src/purchase/payment-sync/**`

구성:

- `PurchaseOrderPaymentSyncOrchestrator`
- `PurchaseOrderPaymentScheduleAggregator`
- `PurchaseOrderPaymentFactory`
- `PurchaseOrderPaymentUpdater`
- `PurchaseOrderPaymentSyncPolicy`
- `PurchaseOrderPaymentAmountCalculator`
- `PurchaseOrderPaymentFixedAmountCalculator`
- `PaymentTimelineMapper`
- `SupplierResolver`

흐름:

1. PO/RP와 Quotation을 읽는다.
2. `PaymentTermParser`로 schedule을 만든다.
3. `PaymentMilestoneService.upsertMany()`로 milestone 레코드를 확보한다.
4. `PaymentFactory`가 `Payment`를 만든다.
5. `PaymentUpdater`가 금액, 계좌, due date를 계산해 채운다.

SKU/MATERIAL PO:

- RP 단위로 payment가 생성된다.
- 수량은 `receivingPlan.quantity ?? receivingPlan.expectedQuantity ?? 0`.
- 금액은 `quantity * quotation.unitPrice`, 세금은 `amountWithoutTax * quotation.taxRate`.

SUPPLIER PO:

- PO 직접 결제문서로 생성된다.
- 수량은 `confirmedQuantity ?? requestedQuantity`.

Sync 제외 정책:

`PurchaseOrderPaymentSyncPolicy.isSyncable()`은 다음 조건을 만족해야 sync 대상이다.

- 금액이 있다.
- 요청/승인/지급되지 않았다.
- 수동 조정 milestone이 아니다.

따라서 요청·승인·지급된 payment는 이후 PO/RP 변경으로 자동 재계산되지 않는다. 이 정책은 운영상 안전하지만, 회계적으로는 변경분을 adjustment/credit note/debit note로 표현할 별도 모델이 필요하다.

### 2.5 Purchase V2 API

파일:

- `src/purchase-v2/controllers/purchase-order.v2.controller.ts`
- `src/purchase-v2/controllers/receiving-plan.v2.controller.ts`
- `src/purchase-v2/controllers/lot.v2.controller.ts`
- `src/purchase-v2/services/receiving-plan-payment-sync.service.ts`

주요 엔드포인트:

- `GET /v2/purchase-orders`: PO 목록 조회.
- `GET /v2/purchase-orders/payments`: PO/RP 연결 결제 목록 조회.
- `PATCH /v2/purchase-orders/warehouse`: PO 입고지 일괄 변경.
- `GET /v2/receiving-plans`: RP 목록 조회.
- `GET /v2/receiving-plans/:id`: RP 상세 + inventory events.
- `GET /v2/receiving-plans/:id/payment-schedules`: RP 기준 결제 스케줄 미리보기.
- `POST /v2/receiving-plans/:id/payments`: schedule 기반 결제문서 추가.
- `PUT /v2/receiving-plans/:rpId/payments/:paymentId`: 기존 결제문서 schedule 교체.
- `POST /v2/lots/events/*`: WMS 입고 이벤트 수집/발행/수정.
- `POST /v2/lots/reservations`: 입고 예약 생성.

`ReceivingPlanPaymentSyncService`는 이미 지급/요청/승인된 payment 수정을 막는다.

### 2.6 Payment 운영 흐름

파일:

- `src/payments/payment.controller.ts`
- `src/payments/payment.service.ts`

주요 엔드포인트:

- `GET /payments`: 결제 목록.
- `POST /payments`: 결제문서 생성.
- `PATCH /payments/:id`: 결제문서 수정.
- `POST /payments/:id/request`: 결제 요청.
- `POST /payments/:id/approve`: 1차/2차 승인.
- `POST /payments/:id/cancel-approval`: 승인 취소.
- `POST /payments/:id/cancel`: 요청 취소.
- `POST /payments/:id/reject`: 반려.
- `PATCH /payments/:id/payment-complete`: 지급 완료.
- `PATCH /payments/:id/recalculate-due-date`: milestone 기반 due date 재계산.
- `GET /payments/supplier/:id/expected-amount`: 공급사 다음 정산 예상금액.
- `GET /payments/:id/order-shipment-items`: 공급사 정산 품목 엑셀.
- `GET /payments/:id/payment-adjustment`: 결제 조정 엑셀.

`request()` 단계에서 검증하는 것:

- 이미 요청/지급된 payment가 아니어야 한다.
- 계좌 정보가 있어야 한다.
- 요청자, 1차 승인자, 최종 승인자가 있어야 한다.
- milestone 기준일이 충족되어야 한다. 단, `daysAfterMilestone < 0`인 선결제는 기준일 검증을 건너뛴다.
- 요청 시점의 Quotation을 `payment.quotation`에 snapshot처럼 연결한다.

`paymentComplete()` 단계:

- 요청되고 승인된 payment여야 한다.
- 환율 검증을 한다. KRW는 환율 1이어야 하고, 외화는 환율이 필요하다.
- `paidAt`을 현재 시각으로 채운다.

주의:

- 현재 Payment는 "AP 확정 전표"가 아니라 운영 결제 요청/승인/지급 문서에 가깝다.
- 지급 완료 시 현금/예금 계정 감소, AP 소멸 전표 같은 회계 transaction은 없다.

### 2.7 공급사 위탁 정산과 PaymentAdjustment

파일:

- `src/payments/payment.service.ts`
- `src/payments/payment-adjustment-service.ts`
- `src/payments/entity/payment-adjustment.entity.ts`
- `src/payments/vendor-settlement-admin.controller.ts`

위탁 정산 흐름은 PO 기반 매입과 별개의 판매/풀필먼트 정산 성격이 강하다.

`createVendorSettlement()`:

- supplier의 위탁 SKU shipment items를 기간 기준으로 모은다.
- 주문일 기준 SKU quotation이 없으면 Slack 알림 후 정산 중단.
- 누락된 PaymentAdjustment를 생성한다.
- item amount + shipping fee - adjustment amount로 Payment를 생성한다.
- 생성한 Payment를 OrderShipmentItem/PaymentAdjustment에 연결한다.

이 흐름은 첨부 기획 O4/O5/O6와도 연결될 수 있지만, 현재 Tier 1 PO/RP 매입채무 모델과는 다른 경계다.

### 2.8 Finance 구현

파일:

- `src/finance/tax-invoice.service.ts`
- `src/finance/bank-transfer.service.ts`
- `src/finance/finance.service.ts`
- `src/finance/entity/adjustment-document.entity.ts`

현재 기능:

- Popbill `HTTaxinvoiceService`로 매입 세금계산서 작업 요청/상태 조회/검색 결과 조회.
- Popbill `EasyFinBankService`로 은행 거래 작업 요청/상태 조회/검색 결과 조회.
- Popbill `AccountCheckService`로 계좌 실명 확인.
- `AdjustmentDocument`는 `CREDIT_NOTE`, `DEBIT_NOTE`와 supplier, issuedAt, clearedAt, amount, description을 저장한다.

중요한 갭:

- TaxInvoice 결과는 PO/RP/Payment에 자동 연결되지 않는다.
- 은행 거래 결과도 Payment 지급과 자동 대사되지 않는다.
- 세금계산서 역발행 생성/승인 플로우는 현재 보이지 않는다.
- 조정문서는 존재하지만 AP/GRIR/PaymentAdjustment와 회계적으로 연결된 ledger 모델은 아니다.

## 3. 기획 목표 대비 구현 갭

### O1. 정확한 단가로만 발주

현재 가능한 것:

- Quotation record를 여러 개 둘 수 있다.
- `applyDate` 기준 현재 견적을 찾는 helper가 있다.
- PO가 Quotation과 연결된다.

부족한 것:

- `validFrom/validTo` 양방향 유효기간.
- max validity 자동 만료.
- quotation approval workflow.
- quotationType, pricePolicy, 허용 variance.
- PO 생성 시 승인된 단가 자동 결정 강제.
- PO 수정/API/UI의 단가 변경 우회 차단.
- 단가 승인 R&R과 시스템 권한 매핑.

구현 방향:

- `Quotation`을 그대로 확장할지, `PurchasePrice` 같은 별도 price master를 둘지 먼저 결정해야 한다.
- 단가 선택은 API에서 `quotationId`를 받는 구조가 아니라, 대상/거래처/기준일/quotationType으로 서버가 결정해야 한다.
- PO에는 price master 참조와 동시에 가격 snapshot을 두는 것이 안전하다. 참조만 있으면 과거 PO 재현이 어렵다.
- 단가 변경은 "기존 PO 수정"이 아니라 "취소 + 새 단가 승인 + 새 PO" 또는 credit/debit adjustment로 처리해야 한다.

### O2. 입고된 만큼, 그 단가대로 지급

현재 가능한 것:

- RP 수량과 PO quotation으로 Payment schedule을 산출한다.
- PaymentTerm으로 분할 지급 스케줄을 표현한다.
- 요청/승인/지급된 payment는 자동 sync에서 제외한다.
- Popbill 매입 세금계산서 조회 API가 있다.

부족한 것:

- "매입확정" 독립 모델.
- 입고와 GR/IR 대기 채무를 같은 transaction으로 만드는 회계 transaction.
- 세금계산서 엔티티와 PO/RP/Payment line matching.
- 3-way matching engine.
- GR/IR 대기 -> AP 확정 상태 전이.
- AP 확정분만 지급 대상으로 삼는 강제 정책.
- 지급 실행과 현금/예금 감소의 원자적 ledger.

구현 방향:

- `Payment`를 AP 확정 문서로 격상할지, 별도 `Payable` / `VendorInvoice` / `PaymentExecution`을 둘지 정해야 한다.
- 세금계산서는 supplier/date/amount aggregate가 아니라 line-level matching을 고려해야 한다.
- PO, GR/RP, TaxInvoice, AP, PaymentExecution, BankTransaction의 ID chain을 설계해야 한다.

### O3. 재고 변동의 원자 기록

현재 가능한 것:

- `LotInventoryEvent`가 입고/예약 event를 기록하고 commerce-web-api로 발행한다.
- sourceType으로 RESERVED/UNTRACKED/PO_ONLY/COMPLETED를 구분한다.
- WMS 수집 경로가 KCP/JSS/KLS로 분리되어 있다.
- 수량 불일치 감지와 override 발행 경로가 있다.
- BOM을 통해 SKU의 사급/부자재 구성을 조회할 수 있고, demand-forecast 쪽에서 부자재 자동 발주용 tree 순회를 이미 구현했다.

부족한 것:

- 회계용 inventory transaction ledger.
- 재고 상태(available/blocked/quality hold/returned) 분리.
- 원가 layer와 수량 layer의 연결.
- 원인코드(reason code) 강제.
- 반품 재고 blocked 처리와 검수 transaction.
- WMS 대사 결과를 회계/재고 ledger에 반영하는 구조.
- BOM 기준 component issue, finished goods receipt, WIP/임가공 상태 전이.
- BOM version snapshot과 실제 투입량 variance 기록.

구현 방향:

- 기존 `LotInventoryEvent`는 commerce stock outbox로 유지하고, 회계/재고 원장용 transaction은 별도 append-only 모델로 분리하는 것이 안전하다.
- 입고 완료 시 "재고 증가 + GR/IR 대기"를 같은 application transaction 또는 outbox-saga로 묶어야 한다.
- SKU가 BOM 기반 생산/임가공 품목이면 단순히 완제품 lot만 증가시키면 안 된다. 구성품 차감, 완제품 증가, scrap/yield 차이, 임가공비 또는 전환원가를 별도 transaction으로 남겨야 한다.

### O4~O8. 재무·회계 사이드

현재 일부 기반:

- `Payment.costCategory`.
- `AdjustmentDocument`.
- Popbill 세금계산서/은행거래 조회.
- 공급사 정산 Payment/PaymentAdjustment.

부족한 것:

- 매출 확정 원장.
- 비용 발생/상신/승인 통합 모델.
- 계정과목/회계분류 master.
- transaction별 회계 분류 강제.
- 은행거래와 Payment/Cash ledger 대사.
- SKU별 COGS, 판관비 배부, 영업이익 엔진.
- BOM 기반 component cost roll-up과 제조/임가공 원가 산정.

구현 방향:

- `CostCategory` enum만으로는 회계 계정과목을 대체할 수 없다.
- chart of accounts, cost center, SKU allocation key, supplier/payment term, tax category, settlement source를 별도 master로 다뤄야 한다.
- O6은 가장 마지막 산출물이다. O1~O5/O7/O8의 transaction 품질이 선행되지 않으면 정확한 SKU 영업이익은 계산할 수 없다.
- 특히 BOM 품목은 SKU 단가만으로 COGS를 잡으면 안 된다. component purchase price, 실제 투입량, yield/scrap, 임가공비, overhead를 어떤 기준으로 SKU/lot에 roll-up할지 먼저 정해야 한다.

## 4. ERP·회계 도메인 지식

### 4.1 표준 구매-회계 흐름

ERP에서 구매와 회계는 보통 다음 순서로 이어진다.

1. **Purchase Order (PO)**
   - 구매 의사와 조건을 확정한다.
   - 보통 FI 회계전표는 발생하지 않는다.
   - 품목, 수량, 단가, 공급사, payment terms, 납기, 세금, 통화를 확정한다.
2. **Goods Receipt (GR)**
   - 재화/서비스 수령을 기록한다.
   - 재고 또는 비용을 차변으로 기록하고, GR/IR clearing 계정을 대변으로 기록한다.
   - 이 단계는 "우리가 받았으니 잠재 채무가 생겼다"에 가깝다.
3. **Invoice Receipt (IR)**
   - 공급사 세금계산서/송장을 수취한다.
   - GR/IR을 차변으로 제거하고, vendor/AP 계정을 대변으로 확정한다.
   - 가격/수량 차이가 있으면 matching exception이 된다.
4. **Payment**
   - 확정 AP를 지급한다.
   - AP를 차변으로 제거하고 현금/예금을 대변으로 감소시킨다.
5. **Bank Reconciliation**
   - 시스템의 지급 기록과 은행 거래 내역을 대사한다.

SAP Learning은 PO 단계에서는 FI posting이 없고, GR 단계에서 material stock/consumption과 GR/IR open item이 생기며, IR 단계에서 GR/IR과 vendor account가 처리된다고 설명한다. 또한 GR/IR clearing은 goods receipt와 invoice receipt 차이를 exception handling하는 프로세스다.  
출처: [SAP Learning - Automating the Clearing Process](https://learning.sap.com/learning-journeys/implementing-record-to-report-in-sap-s-4hana/automating-the-clearing-process)

### 4.2 GR/IR

GR/IR은 Goods Receipt / Invoice Receipt clearing이다.

왜 필요한가:

- 입고와 세금계산서 수취 시점이 다르다.
- 입고는 됐지만 invoice가 없으면 AP를 확정하기 이르다.
- invoice는 왔지만 입고가 없으면 수량/품질 확인 전 지급 위험이 있다.

일반 전표 예시:

```text
GR:
Dr Inventory 또는 Expense
Cr GR/IR Clearing

IR:
Dr GR/IR Clearing
Cr Accounts Payable - Supplier

Payment:
Dr Accounts Payable - Supplier
Cr Cash/Bank
```

개발 관점:

- "입고 완료"와 "Payment 생성"만으로 GR/IR이 구현된 것은 아니다.
- GR/IR에는 회계 계정, 금액, 통화, 세금, supplier, PO/RP line, clearing 상태가 필요하다.
- 금액 차이는 Payment 금액 재계산이 아니라 matching exception 또는 variance posting으로 표현해야 한다.

### 4.3 2-way / 3-way matching

Microsoft Dynamics 문서는 AP invoice matching을 vendor invoice, purchase order, product receipt 정보를 맞추는 과정으로 설명한다. 3-way matching은 invoice와 PO의 가격, invoice와 product receipt의 수량을 함께 확인한다.  
출처: [Microsoft Learn - Accounts payable invoice matching overview](https://learn.microsoft.com/en-us/dynamics365/finance/accounts-payable/accounts-payable-invoice-matching)

기본 비교:

- 2-way: PO price vs invoice price.
- 3-way: PO price vs invoice price + receipt quantity vs invoice quantity.
- Charges matching: 운송비/부대비용 등 charge 금액 비교.

현재 코드에 매핑하면:

- PO price: `PurchaseOrder.quotation`.
- Receipt quantity: `ReceivingPlan.quantity`, `Lot.receivedQuantity`, `LotInventoryEvent.delta`.
- Invoice amount: Popbill `TaxInvoice.supplyCostTotal`, `taxTotal`, `totalAmount`.
- Payment amount: `Payment.amountWithoutTax`, `Payment.tax`.

현재 없는 것:

- invoice line entity.
- PO/RP line과 invoice line의 matching result.
- tolerance rule.
- mismatch approval workflow.
- AP confirmation gate.

### 4.4 단가 마스터와 구매계약

ERP에서는 공급사-품목 조합 가격을 기간/계약/조건 단위로 관리한다.

Microsoft Dynamics의 purchase agreement는 일정 수량/금액 구매 commitment에 대한 가격/할인 조건을 담고, validity period가 있으며, agreement를 effective로 활성화해야 효력이 생긴다. 가격이 fixed인 agreement에서 PO line 가격이 바뀌면 commitment link가 깨질 수 있다.  
출처: [Microsoft Learn - Purchase agreements](https://learn.microsoft.com/en-us/dynamics365/supply-chain/procurement/purchase-agreements)

개발자가 이해해야 할 설계 포인트:

- "단가"는 단순 숫자가 아니라 계약 조건이다.
- 같은 SKU라도 supplier, 통화, 세금 포함 여부, 단위, MOQ, 유효기간, 프로모션/무상증정 여부에 따라 다른 가격이다.
- PO에는 "현재 가격을 참조"하는 것과 "그 시점의 가격을 snapshot"하는 것이 모두 필요하다.
- 과거 PO 재현과 감사에는 snapshot이 필수다.
- 가격 변경은 기존 record overwrite가 아니라 새 record 추가가 원칙이다.

기획의 `pricePolicy` 해석:

- `FIXED`: 승인된 단가와 정확히 일치해야 한다.
- `VARIANCE`: 허용 편차 내 자동 승인 가능.
- `RANGE`: min/max 범위 안이면 자동 승인 가능.

### 4.5 Payment Terms

Oracle Payables 문서는 payment terms가 invoice installments를 자동 생성하고, 각 line/installment가 due date와 discount date/amount를 가진다고 설명한다. due date는 fixed date, days, calendar, day-of-month/cutoff/months-ahead 같은 방식으로 계산될 수 있다.  
출처: [Oracle Financials - Payment Terms](https://docs.oracle.com/en/cloud/saas/financials/25c/faipp/payment-terms.html)

현재 코드의 `PaymentTerm`은 ERP payment terms와 비슷하지만 다음 차이가 있다.

- enum + switch 기반이라 데이터로 운영자가 쉽게 추가/수정하기 어렵다.
- supplier master default term과 PO override term의 hierarchy가 명확하지 않다.
- invoice/AP installment와 연결되어 있지 않고 Payment schedule 생성용이다.

향후 설계 시 필요한 것:

- supplier default payment term.
- PO-level override.
- tax invoice/AP invoice가 PO를 참조하면 PO payment term 기본 적용.
- 수동 변경 시 기존 installment/manual adjustment 재계산 정책.
- holiday/weekend 보정 정책.

### 4.6 세금계산서: 정발행, 역발행, 매입자발행

국세청은 전자(세금)계산서 발급 완료 시점과 기한을 안내한다. 월합계 세금계산서 등은 공급시기가 속하는 달의 다음달 10일까지 발급 가능하고, 발급일 다음날까지 국세청에 전송해야 한다.  
출처: [국세청 - 발급시기 및 발급·전송기한](https://www.nts.go.kr/nts/cm/cntnts/cntntsView.do?cntntsId=7789&mi=2463)

정발행/역발행 용어:

- 정발행: 공급자(매출자)가 거래 내용을 작성해 공급받는자에게 발행.
- 상용 서비스에서 말하는 역발행: 공급받는자(매입자)가 초안을 작성 요청하고, 공급자가 확인/승인 후 최종 발행.
- 최종 법적 발행 주체는 여전히 공급자라는 점이 중요하다.

역발행 설명 출처: [Bolta - 세금계산서 정발행과 역발행의 차이](https://bolta.io/insight/tax-invoice-issue-types)

주의:

- "역발행"과 "매입자발행세금계산서"는 구분해야 한다.
- 매입자발행세금계산서는 공급자가 발급하지 않는 경우 관할 세무서 확인을 거쳐 매입자가 발행하는 제도 성격이다.
- 기획의 R2.3 "역발행 default"가 상용 API reverse issue를 뜻하는지, 법정 매입자발행 제도를 뜻하는지 반드시 확인해야 한다.

현재 코드:

- Popbill HTTaxinvoiceService로 매입 세금계산서 조회는 가능하다.
- PO/RP/Payment와 matching하는 로직은 없다.
- 역발행 생성/승인 요청 API는 현재 확인되지 않았다.

### 4.7 재고 원가와 COGS

IFRS IAS 2는 재고를 원가와 순실현가능가치 중 낮은 금액으로 측정한다고 설명한다. 재고 원가는 구매원가, 전환원가, 현재 위치와 상태에 이르게 하는 기타 원가를 포함하며, 판매 시 장부금액은 관련 수익이 인식되는 기간의 비용으로 인식된다. FIFO 또는 weighted average 같은 cost formula가 사용된다.  
출처: [IFRS - IAS 2 Inventories](https://www.ifrs.org/issued-standards/list-of-standards/ias-2-inventories/)

개발 관점:

- SKU 영업이익을 내려면 판매 시점의 COGS가 필요하다.
- COGS는 단순히 "현재 quotation"이 아니라 판매된 재고 layer의 원가다.
- 원가 layer 기준이 FIFO인지 weighted average인지, lot-specific인지 정해야 한다.
- 매입 부대비용(물류, 관세, 인증, 임가공 등)을 SKU/PO/lot에 배부해야 진짜 gross margin이 나온다.
- recoverable VAT는 일반적으로 재고 원가가 아니라 세금채권/부채로 분리해야 한다. 현재 코드가 `amountWithoutTax`와 `tax`를 분리하는 방향은 이 관점과 맞다.

### 4.8 BOM, 자재소요, 원가 roll-up

Microsoft Dynamics 문서는 BOM을 제품 생산에 필요한 구성품 정의로 설명하고, BOM calculation이 구성품 원가를 상위 품목 원가로 roll-up하는 계산이라고 설명한다. BOM calculation group은 purchase price trade agreements 같은 원가 출처와 warning rule을 정의할 수 있다. Oracle Cost Rollup도 BOM과 routing을 기반으로 완제품 표준원가를 계산한다.  
출처: [Microsoft Learn - Bills of materials and formulas](https://learn.microsoft.com/en-us/dynamics365/supply-chain/production-control/bill-of-material-bom), [Microsoft Learn - BOM calculations](https://learn.microsoft.com/en-us/dynamics365/supply-chain/cost-management/bom-calculations), [Oracle - Using the Cost Rollup Process to Calculate Product Costs](https://docs.oracle.com/cd/G24469_01/fscm92pbr53/eng/fscm/scem/UsingtheCostRollupProcesstoCalculateProductCosts-c9fbea.html)

첨부 기획과의 관계:

- O1/O2의 최소 범위가 "완제품 또는 material을 정확한 가격으로 발주하고, 입고분만 지급"이라면 BOM은 직접 선행조건이 아니다.
- 하지만 현재 프로젝트처럼 SKU에 사급/부자재 BOM이 있고, 부자재 자동 발주를 고려한다면 O1의 price master는 완제품뿐 아니라 component material에도 적용되어야 한다.
- O3에서는 BOM 품목 입고 시 component issue, finished goods receipt, scrap/yield variance, blocked/returned component 처리까지 transaction으로 남겨야 한다.
- O6 SKU 영업이익에서는 BOM이 사실상 필수다. 완제품 SKU의 COGS는 "판매 SKU의 현재 quotation"만으로 계산할 수 없고, 판매된 lot이 어떤 BOM version과 어떤 component 원가로 만들어졌는지 따라 달라진다.

개발자가 구분해야 할 것:

- **Engineering BOM**: 제품 설계상 필요한 구성.
- **Manufacturing BOM / Formula**: 생산 또는 임가공에 실제 투입되는 구성, yield/scrap 포함.
- **Costed BOM**: 가격 master, routing, overhead를 붙여 원가 roll-up에 쓰는 구성.
- **As-planned BOM**: PO/생산 계획 시점의 예상 구성.
- **As-built BOM**: 실제 입고/생산 완료 시점의 실투입 구성.

현재 코드와의 gap:

- `BillOfMaterials.applyDate`만 있고 active version 확정 규칙이 약하다.
- `LinkMaterials.componentQuantity`는 있지만 scrap/yield, 대체 자재, optional component, phantom assembly 개념은 없다.
- component purchase price는 `Quotation`으로 조회하지만, BOM version과 price version을 함께 snapshot하지 않는다.
- BOM 기반 component issue / WIP / finished goods receipt transaction이 없다.
- 사급 material의 소유권과 원가 귀속이 불명확하다. 사급은 "우리가 산 component를 공급사에 제공해 완제품을 받는지", "공급사가 component까지 부담하는지"에 따라 회계처리가 달라진다.

### 4.9 현금/은행 대사

Microsoft Dynamics Cash and bank management 문서는 법인 은행계좌와 금융수단을 관리하고, bank statement reconciliation과 AP payments/settlements를 같은 현금관리 영역에서 다룬다.  
출처: [Microsoft Learn - Cash and bank management](https://learn.microsoft.com/ro-ro/dynamics365/finance/cash-bank-management/cash-bank-management)

현재 코드:

- Popbill 은행 거래 조회가 있다.
- Payment 지급 완료(`paidAt`)와 은행 거래가 자동으로 연결되지 않는다.

향후 필요:

- Payment execution ID.
- 은행 거래 unique key.
- 금액/날짜/계좌/적요 기반 matching rule.
- one-to-one, one-to-many, many-to-one reconciliation.
- 대사 실패/부분 지급/수수료/환차손익 처리.

## 5. 개발 시 설계 원칙

### 5.1 가격은 mutable field가 아니라 승인된 master data다

하지 말아야 할 것:

- PO API에서 `unitPrice`나 임의 `quotationId`를 그대로 신뢰.
- 기존 quotation record overwrite.
- 지급/승인된 payment를 조용히 재계산.

해야 할 것:

- 서버에서 `target + supplier + postingDate + quotationType + status`로 active price를 결정.
- price master에 승인 상태와 유효기간을 둔다.
- PO line에는 price master ID와 price snapshot을 모두 저장한다.
- 변경분은 취소/재발행 또는 adjustment document로 남긴다.

### 5.2 PO, GR, IR, AP, Payment를 같은 말로 쓰면 안 된다

현재 코드의 `Payment`는 AP 확정 전표가 아니다. 기획 구현에서는 용어를 분리해야 한다.

- PO: 구매 의사/조건.
- GR: 입고 사실과 재고/GRIR 발생.
- IR / TaxInvoice: 공급사 청구/세금계산서.
- AP: matching을 통과한 확정 채무.
- Payment: AP를 실제 지급하는 행위.
- BankTransaction: 은행에서 관측된 현금 이동.

### 5.3 원자성은 "한 service에서 flush" 이상의 의미다

기획 R2.2의 "입고-채무 atomic"은 다음 중 하나를 의미해야 한다.

- 같은 DB transaction 안에서 inventory transaction과 GR/IR transaction을 함께 생성.
- 외부 WMS/commerce publish가 끼는 경우 transactional outbox로 eventual consistency를 보장.

현재 `LotInventoryEvent` publish는 외부 시스템 발행 흐름이므로 회계 원자성과 분리해서 봐야 한다.

### 5.4 Matching exception은 정상 도메인이다

3-way matching 실패는 에러 로그가 아니라 업무 상태다.

필요한 상태 예:

- `MATCH_PENDING`
- `MATCH_PASSED`
- `PRICE_VARIANCE`
- `QUANTITY_VARIANCE`
- `TAX_VARIANCE`
- `CHARGE_VARIANCE`
- `MANUAL_APPROVED`
- `REJECTED`

허용 tolerance와 승인권자는 master/policy로 관리해야 한다.

### 5.5 append-only ledger를 우선 고려한다

재고/회계/현금은 추적성과 감사가 핵심이다.

권장:

- 기존 행 overwrite보다 reversal/adjustment transaction.
- original transaction reference.
- source document reference.
- actor/system origin.
- idempotency key.
- posting date와 createdAt 분리.
- KST business date와 UTC timestamp 분리.

### 5.6 BOM도 version snapshot 대상이다

BOM은 현재값만 참조하면 안 된다.

위험한 구현:

- 판매/입고/정산 시점마다 `SKU.billOfMaterials`의 최신값을 다시 읽어 원가를 계산.
- 과거 PO나 lot의 원가를 BOM 수정 후 재계산.
- component material quotation의 현재값으로 과거 완제품 COGS를 계산.

필요한 방향:

- PO/생산/입고 시점의 BOM version과 component 수량을 snapshot한다.
- component price master version도 함께 snapshot한다.
- 실제 투입량이 계획 BOM과 다르면 usage variance/scrap variance로 남긴다.
- BOM 변경은 승인/유효일/만료일이 있는 master data 변경으로 취급한다.
- "완제품 직접 매입 SKU"와 "BOM 기반 임가공/조립 SKU"를 원가 산식에서 분기한다.

## 6. 구현 후보 모델

기획을 현재 코드 위에 얹을 때 검토할 수 있는 최소 모델이다. 실제 명명은 팀 컨벤션에 맞춰야 한다.

### PurchasePrice 또는 Quotation 확장

필수 필드 후보:

- target type: SKU/MATERIAL/SUPPLIER.
- target ID.
- supplier ID.
- unit, currency.
- amountWithoutTax, taxRate, taxIncluded flag.
- validFrom, validTo.
- quotationType: STANDARD/FREE_GOODS.
- pricePolicy: FIXED/VARIANCE/RANGE.
- allowedVarianceAmount/Rate 또는 min/max.
- status: DRAFT/POLICY_CHECK/PENDING_APPROVAL/ACTIVE/REJECTED/EXPIRED.
- approvedBy, approvedAt.
- evidence files.

### PurchaseOrderLine snapshot

현재 PO는 line entity가 없다. SKU/MATERIAL/SUPPLIER 중 하나의 대상만 가진다. 향후 ERP형 PO라면 line 분리가 필요할 수 있다.

필수 snapshot:

- price master ID.
- unit price without tax.
- tax rate.
- currency/exchange rate.
- quantity.
- supplier.
- posting/business date.

### BomVersionSnapshot / ProductionConsumptionTransaction

BOM 기반 SKU를 O3/O6까지 연결하려면 PO/RP/Lot과 별도의 생산/임가공 소요 transaction이 필요할 수 있다.

`BomVersionSnapshot` 필수 후보:

- source BOM ID.
- SKU ID.
- effective business date.
- approved/active status.
- component lines: material ID, planned quantity, unit.
- component quotation/price master snapshot.
- snapshot reason: PO creation, production plan, receipt completion, cost roll-up.

`ProductionConsumptionTransaction` 필수 후보:

- source PO/RP/Lot 또는 production/order reference.
- finished SKU/Lot reference.
- component material reference.
- planned quantity.
- actual consumed quantity.
- scrap/rejected quantity.
- component unit cost snapshot.
- ownership: company-owned/subcontractor-owned/customer-owned.
- posting date.

### ReceiptTransaction

필수:

- PO/RP/Lot reference.
- received quantity.
- accepted/rejected/blocked quantity.
- inventory status.
- unit cost snapshot.
- amount without tax/tax if applicable.
- WMS source and idempotency key.
- posting date.

### GrIrEntry

필수:

- receipt transaction reference.
- supplier.
- amount.
- currency.
- debit account / credit account.
- open/cleared amount.
- clearedBy invoice/AP reference.

### TaxInvoice / VendorInvoice

필수:

- ntsconfirmNum.
- supplier business registration number.
- issueDate/writeDate/sendDate.
- supplyCostTotal/taxTotal/totalAmount.
- line items.
- invoice direction: regular/reverse/buyer-issued.
- source: Popbill/manual/API.
- status.

### MatchResult

필수:

- PO line.
- receipt transaction(s).
- invoice line.
- expected amount/quantity.
- actual amount/quantity.
- variance.
- tolerance.
- status.
- approval trail.

### AccountsPayable

필수:

- supplier.
- matched invoice.
- confirmed amount.
- due date/installments.
- status: DRAFT/MATCHED/APPROVED_FOR_PAYMENT/PAID/CANCELLED.
- remaining amount.

### PaymentExecution

필수:

- AP installment.
- bank account.
- requested/approved/executed timestamps.
- transfer file/provider result.
- bank transaction match status.

### AccountingTransaction / JournalEntry

필수:

- source type/source id.
- posting date.
- lines: account, debit, credit, currency, dimensions.
- reversal reference.

## 7. 구현 전 반드시 확인할 질문

1. PO 단가 기준일은 무엇인가?
   - 기획에는 `posting`이라고 되어 있다.
   - 현재 코드는 `now` 기준 `applyDate`를 본다.
   - PO 요청일, 작성일, 공급사 확정일, 입고 예정일 중 무엇인지 확정해야 한다.
2. 단가 승인 주체와 단계는 정확히 누구인가?
   - PM 입력, CL 승인, 구매팀 승인 전환 시나리오를 권한/상태로 모델링해야 한다.
3. `quotationId`를 API에서 제거할 수 있는가?
   - 제거하지 않으면 R1.3/R1.4가 충족되지 않는다.
4. 기존 PO의 quotation 변경을 어떻게 마이그레이션할 것인가?
   - 이미 승인/지급된 Payment와 충돌할 수 있다.
5. `FREE_GOODS`는 PO 금액 0으로 볼 것인가, 별도 세금/재고 원가 정책이 있는가?
6. 세금계산서 역발행은 상용 reverse issue인가, 법정 매입자발행세금계산서인가?
7. Popbill에서 세금계산서 생성/역발행 API를 쓸 것인가, 조회만 쓸 것인가?
8. 3-way matching tolerance는 supplier/item/category/legal entity 중 어디서 정의하는가?
9. 입고 수량 over/under tolerance는 현재 +5%와 동일한가?
10. GR/IR과 AP 회계전표를 실제 장부로 쓸 것인가, 내부 관리 ledger로만 쓸 것인가?
11. SKU 원가 산식은 FIFO, weighted average, lot-specific 중 무엇인가?
12. 물류비/관세/인증비/임가공비를 SKU/PO/lot에 어떻게 배부할 것인가?
13. 환율 기준일은 PO, invoice, payment 중 무엇인가?
14. 은행 대사는 payment 단건 기준인가, 여러 AP 묶음 지급을 허용하는가?
15. 위탁 정산(`OrderShipmentItem` 기반)과 PO 매입 정산을 같은 AP 모델에 합칠 것인가, 별도 subledger로 둘 것인가?
16. BOM 품목은 직접 완제품 매입인가, 사급/임가공/조립 흐름인가?
17. BOM active version은 어떤 기준으로 결정하는가: PO 요청일, 생산계획일, 입고일, 판매일?
18. component scrap/yield/대체자재를 관리해야 하는가?
19. component 원가는 완제품 lot에 실제투입 기준으로 귀속할 것인가, 표준 BOM 기준으로 귀속할 것인가?
20. 사급 material의 소유권은 입고 전후 어느 시점에 누구에게 있는가?

## 8. 개발자가 먼저 읽을 파일

규칙:

- `CLAUDE.md`
- `src/purchase/CLAUDE.md`
- `src/purchase-v2/CLAUDE.md`
- `src/purchase/AGENTS.md`
- `src/purchase-v2/AGENTS.md`
- `src/payments/AGENTS.md`

도메인 엔티티:

- `src/sku/entities/quotation.entity.ts`
- `src/sku/entities/sku.entity.ts`
- `src/sku/entities/material.entity.ts`
- `src/sku/entities/bill-of-materials.entity.ts`
- `src/sku/entities/link-materials.entity.ts`
- `src/purchase/entity/purchase-order.entity.ts`
- `src/purchase/entity/receiving-plan.entity.ts`
- `src/purchase/entity/lot.entity.ts`
- `src/purchase/entity/lot-inventory-event.entity.ts`
- `src/payments/entity/payment.entity.ts`
- `src/payments/entity/payment-milestone.entity.ts`
- `src/payments/entity/payment-adjustment.entity.ts`
- `src/finance/entity/adjustment-document.entity.ts`

주요 흐름:

- `src/purchase/purchase-order.service.ts`
- `src/purchase/receiving-plan.service.ts`
- `src/purchase/payment-sync/README.md`
- `src/purchase/payment-sync/orchestrator/purchase-order-payment-sync.orchestrator.ts`
- `src/purchase/payment-sync/aggregator/purchase-order-payment-schedule-aggregator.ts`
- `src/purchase/payment-sync/updater/purchase-order-payment-updater.ts`
- `src/demand-forecast/demand-forecast.service.ts`
- `src/purchase-v2/services/receiving-plan-payment-sync.service.ts`
- `src/purchase-v2/components/receipt/inventory-event.publisher.ts`
- `src/purchase-v2/components/receipt/receiving-plan-completion.processor.ts`
- `src/payments/payment.service.ts`
- `src/finance/tax-invoice.service.ts`
- `src/finance/bank-transfer.service.ts`

테스트 참고:

- `src/purchase/purchase-order-payment-sync.e2e.spec.ts`
- `src/purchase/purchase-order.e2e.spec.ts`
- `src/purchase-v2/controllers/receiving-plan.v2.controller.e2e.spec.ts`
- `src/purchase-v2/controllers/purchase-order.v2.controller.e2e.spec.ts`
- `src/purchase-v2/controllers/lot.v2.controller.e2e.spec.ts`
- `src/payments/payment.service.spec.ts`
- `src/payments/payment-term/**/*.spec.ts`

## 9. 외부 참고 출처

- [SAP Learning - Automating the Clearing Process](https://learning.sap.com/learning-journeys/implementing-record-to-report-in-sap-s-4hana/automating-the-clearing-process)
- [Microsoft Learn - Accounts payable invoice matching overview](https://learn.microsoft.com/en-us/dynamics365/finance/accounts-payable/accounts-payable-invoice-matching)
- [Microsoft Learn - Purchase agreements](https://learn.microsoft.com/en-us/dynamics365/supply-chain/procurement/purchase-agreements)
- [Microsoft Learn - Bills of materials and formulas](https://learn.microsoft.com/en-us/dynamics365/supply-chain/production-control/bill-of-material-bom)
- [Microsoft Learn - BOM calculations](https://learn.microsoft.com/en-us/dynamics365/supply-chain/cost-management/bom-calculations)
- [Oracle - Using the Cost Rollup Process to Calculate Product Costs](https://docs.oracle.com/cd/G24469_01/fscm92pbr53/eng/fscm/scem/UsingtheCostRollupProcesstoCalculateProductCosts-c9fbea.html)
- [Oracle Financials - Payment Terms](https://docs.oracle.com/en/cloud/saas/financials/25c/faipp/payment-terms.html)
- [국세청 - 전자(세금)계산서 발급시기 및 발급·전송기한](https://www.nts.go.kr/nts/cm/cntnts/cntntsView.do?cntntsId=7789&mi=2463)
- [Bolta - 세금계산서 정발행과 역발행의 차이](https://bolta.io/insight/tax-invoice-issue-types)
- [IFRS - IAS 2 Inventories](https://www.ifrs.org/issued-standards/list-of-standards/ias-2-inventories/)
- [Microsoft Learn - Cash and bank management](https://learn.microsoft.com/ro-ro/dynamics365/finance/cash-bank-management/cash-bank-management)
