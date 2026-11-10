# Review metric cột A — Unified OM db

Review danh sách metric OM request team Dev lấy từ 3 ad platform, phục vụ Semantic Model / Report trong Power BI.

| | |
|---|---|
| Nguồn | [Unified OM db: matcha latte](https://docs.google.com/spreadsheets/d/1RtpQ71SvgqaN4hxuUyCaXoj3gJqtfhUo9kXyXFllRkI/edit?gid=82501209#gid=82501209) — tab `Metrics mapping` |
| Platform | `facebook_ads` · `googleadwords_int` · `tiktokglobal_int` |
| Ngày review | 10/11/2026 |
| Hiện trạng | Cột A có **61 dòng** = 12 dòng bảng mapping (A6:A17) + 49 dòng list matcha (A20:A68) |
| Đề xuất | **~52 metric** sau khi gộp trùng, bỏ metric không dùng và bổ sung metric còn thiếu |

---

## Scope đã chốt

| Quyết định | Hệ quả lên cột A |
|---|---|
| Cột A **chỉ lấy data có sẵn từ platform** | Register, Unique Login, Revenue **không nằm trong cột A**. Hệ quả: cột A chỉ đo được *media delivery*, không đo được hiệu quả campaign. CPI / CPR / CPL thật phải tính ở tầng trên, sau khi join game DB |
| **Pull metric tỷ lệ sẵn từ platform** (không tính bằng DAX) | Giữ 18 metric tỷ lệ trong cột A, kèm ràng buộc kỹ thuật ở mục 2 |
| **Tách organic vs paid** cho Post Boosting | Cột A hiện không có metric organic nào, và 3 connector đang dùng đều không trả organic — mục 5 |
| **Revenue lấy từ game DB** | `ConversionValue` bỏ khỏi cột A |

**Ba loại campaign OM chạy:** User Acquisition (focus Install & Register) · User Engagement (focus Unique Login của churn user) · Post Boosting.

> Vì Install / Register / Login nằm ở game DB, **phải chốt khoá join** (`campaign_id`, hoặc chuẩn naming kiểu `FCM-VN-MAIA-AND-1609-25Jun26-...`). Không có khoá join thì 61 metric này chỉ trả lời được "tiêu bao nhiêu tiền", không trả lời được "được bao nhiêu user".

---

## Tổng kết

| | Số lượng |
|---|---|
| Trùng lặp / cần gộp | **11** |
| Bỏ hẳn | **6** |
| Giữ nguyên | 44 |
| Cần thêm | **8** (3 paid + 5 organic) |
| → Cột A đề xuất | **~52 metric sạch** |

---

## 1. Trùng lặp — 11/61 dòng

Lỗi dễ sửa nhất. Để nguyên thì Dev build 2 cột cho cùng một con số, và report sẽ có 2 số khác nhau cho cùng một câu hỏi.

| Trùng | Với | Xử lý |
|---|---|---|
| `Conversion` (A6) | `Conversion` (A30) | Trùng y hệt — xoá 1 |
| `Reach` (A11) | `Reach` (A51) | Trùng y hệt — xoá 1 |
| `Share` | `Shares` | Trùng y hệt, chỉ khác số nhiều — giữ `Shares` |
| `Interaction` | `Interactions` | Trùng — giữ `Interactions` |
| `Post Engagement` | `Engagement` | Trùng — giữ `Engagement` |
| `Page Visit` | `ProfileVisits` | Trùng — giữ `ProfileVisits` |
| `Link Click` | `ClickDestination` | TikTok gọi click destination chính là link click — giữ 1 |
| `ThruPlay15s` | `ThruPlay` | ThruPlay của FB **chính là** 15-second thruplay — giữ 1 |
| `PageLike` | `Follows` | FB Page Like ≈ TikTok Follow, cùng ý nghĩa business — gộp thành `Follows` |
| `In-app actions` | `Conversion` | Là một loại conversion — chuyển thành `conversion_type` |
| `Landing Page View` | `Conversion` | Cũng là một loại action — chuyển thành `conversion_type` |

**Lỗi naming:** họ video metric đang có 2 convention song song — `VideoView1s` / `VideoView3s` và `VideoWatched2s` / `VideoWatched6s`. Phải thống nhất: `VideoViews2s` / `VideoViews3s` / `VideoViews6s`.

---

## 2. 18/61 metric không được phép SUM

Hệ quả trực tiếp của quyết định pull tỷ lệ sẵn từ platform. Đây là rủi ro kỹ thuật lớn nhất của model.

```
Reach · AdReach · Ctr · Cpc · Cpm · Cpp · ConversionRate · ConversionRateV2
EngagementRate · InteractionRate · ProfileVisitsRate · VideoViewRate
CostPerConversion · CostPerThruplay · AverageCost · AverageCpe · AverageCpv · AverageVideoPlay
```

Số pull từ platform **đúng ở đúng grain mà nó được pull** (ví dụ ad × ngày). Khi user kéo lên campaign hoặc tháng trong Power BI, `SUM` sẽ cộng dồn tỷ lệ và `AVERAGE` sẽ ra sai — average của CTR không phải CTR.

Ràng buộc cần đưa vào spec cho Dev:

1. Mỗi metric tỷ lệ pull về được đánh dấu **`Display-only at source grain`**, không set aggregation mặc định là Sum / Average.
2. Song song, định nghĩa measure DAX recompute cho roll-up: `CTR = DIVIDE(SUM(Click), SUM(Impression))`. Hai cột tách bạch — `Ctr (platform)` để đối chiếu với Ads Manager, `CTR %` để dùng trên report.
3. **Ghi rõ mẫu số của từng tỷ lệ** vào bảng mapping. Hiện chưa suy ra được mẫu số của:
   - `ConversionRateV2` — khác `ConversionRate` ở đâu?
   - `VideoViewRate` — mẫu số là `Impression` hay `VideoPlayActions`?
   - `ProfileVisitsRate`
   - `AverageCost` — Google định nghĩa là cost / interaction, không phải cost / click

Không có mẫu số thì không recompute được, và cũng không ai verify được số.

`Reach` và `AdReach` còn một tầng nữa: không chỉ không cộng được qua ngày, mà reach campaign ≠ tổng reach các ad (trùng người). **Đặc biệt quan trọng với campaign UE** — retarget churn user là tập nhỏ, overlap rất cao.

---

## 3. Mapping mới xong 1/61 dòng

| Tình trạng | Số dòng |
|---|---|
| Có đủ cả 3 platform | **1** (`ThruPlay15s`) |
| Có 1–2 platform | 5 |
| **Trống hoàn toàn** | **55** (6 dòng A12:A17 + toàn bộ 49 dòng matcha) |

49 metric matcha chưa có một mapping nào ở cột C/D/E. Không phải lỗi thiết kế — là **phần việc còn lại chiếm ~90%**, và nó mới là phần quyết định metric nào thực sự khả thi.

**Dự đoán về Google** (`googleadwords_int`) — không có nhóm social engagement:

```
Likes · Comments · Shares · Saves · Reaction · PageLike · Follows
ProfileVisits · Post Engagement · ThruPlay · Cpp · VideoViews2sUnique · Result
```

Tức ~13 metric sẽ trống cột D vĩnh viễn. Điều đó **không sai** — Post Boosting vốn chỉ chạy FB + TikTok. Nhưng model phải xử lý rõ: cột D trống là **"không áp dụng"** chứ không phải **"bằng 0"**. Nếu Dev fill 0, mọi tỷ lệ trung bình cross-platform sẽ bị kéo xuống sai.

---

## 4. "Target" không phải kết quả, và thiếu CPI thật

Cột A có 2 metric là **bid setting, không phải performance metric**:

- `AverageTargetCostPerInstall` — giá CPI mục tiêu đặt trong bid
- `AverageTargetCpa` — giá CPA mục tiêu đặt trong bid

Trong khi đó, với campaign UA focus Install, **cột A đang thiếu `CostPerInstall` thực tế** — tức thiếu đúng KPI chính của loại campaign chạy nhiều nhất.

**Xử lý:** chuyển 2 metric `AverageTarget*` sang bảng campaign settings (dimension), và thêm `CostPerInstall` vào cột A.

---

## 5. Organic cho Post Boosting cần connector thứ 2

Quyết định tách organic vs paid kéo theo một thay đổi lớn hơn là thêm vài dòng vào cột A:

- 3 connector đang dùng (`facebook_ads`, `googleadwords_int`, `tiktokglobal_int`) đều là **Ads API** — chỉ trả phần paid.
- Organic nằm ở **Facebook Page Insights API** và **TikTok Business Account API** — hai nguồn hoàn toàn khác.

Tức cần **thêm 2 data source và 1 grain mới (post-level, không phải ad-level)**. Một post boosting có 1 post ID nhưng có thể có nhiều ad chạy lên nó — model phải nối được `ad → post_id → organic metrics`.

Chính tab `Metrics mapping` (ô A72) xác nhận FB đã tách paid/organic ở tầng metric:

> *"15-second focused views (paid views) — chỉ campaign chạy post boosting mới có phần paid views"*

Cột A hiện chỉ lấy một nửa.

**Nhóm organic cần thêm:** `OrganicReach` ⚠ · `OrganicImpression` · `OrganicEngagement` · `OrganicVideoViews` · `FollowerGrowth`

---

## 6. Danh sách BỎ — 6 metric

| Metric | Lý do |
|---|---|
| `ConversionValue` | Revenue lấy từ game DB — số này không dùng, giữ lại chỉ gây nhầm với revenue thật |
| `Result` | Objective-dependent: Result của campaign Traffic là click, của campaign Install là install. **Không cộng được giữa các campaign khác objective**, không so sánh được giữa platform, Google không có |
| `Views` | Mơ hồ nhất bảng — video view hay page view? Trùng với nhóm `VideoView*` |
| `ConversionRateV2` | Tên kỹ thuật rò rỉ từ connector, không phải tên business. Giữ chỉ khi làm rõ được khác `ConversionRate` ở mẫu số nào |
| `AverageTargetCpa` | Bid setting, không phải kết quả — chuyển sang bảng settings |
| `AverageTargetCostPerInstall` | Bid setting, không phải kết quả — chuyển sang bảng settings |

`Lead` để ngỏ: với 3 loại campaign hiện tại (UA / UE / Post Boosting) thì Lead không dùng tới. Nếu không có campaign lead-gen thì bỏ luôn.

---

## 7. Danh sách THÊM — 8 metric

| Metric | Vì sao |
|---|---|
| **`Frequency`** ⚠ | **Metric quan trọng nhất đang thiếu.** Với campaign UE retarget churn user — tập audience nhỏ, frequency tăng rất nhanh, user bị bội thực quảng cáo. Có `Reach` + `Impression` mà không có `Frequency` là thiếu đúng chỗ đau. FB và TikTok đều trả sẵn |
| **`CostPerInstall`** ⚠ | KPI chính của UA mà cột A đang thiếu (mục 4) |
| **`CostPerLinkClick`** ⚠ | Khác `Cpc` (all clicks, gồm cả click vào ảnh / tên page). FB tách rõ 2 loại — dùng `Cpc` cho campaign traffic sẽ ra số đẹp hơn thực tế |
| `OrganicReach` ⚠ · `OrganicImpression` · `OrganicEngagement` · `OrganicVideoViews` · `FollowerGrowth` | Nhóm organic cho Post Boosting (mục 5) |

---

## 8. Đối chiếu với 3 loại campaign

| Campaign | Platform đo được (cột A) | Lỗ hổng |
|---|---|---|
| **User Acquisition**<br>(Install & Register) | Spend, Impression, Click, ClickDestination, Installations, Conversion, Ctr, Cpc, Cpm, Reach, video metrics | ① Thiếu `CostPerInstall` thật<br>② `Installations` của platform là **số attributed**, sẽ **luôn lệch** với install trong game DB (attribution window + modeled conversions) — phải định nghĩa trước dùng số nào làm chuẩn<br>③ Register không có ở platform — bắt buộc join game DB |
| **User Engagement**<br>(Unique Login churn user) | Spend, Impression, Click, Reach, Ctr, Cpc, Cpm | ① **Thiếu `Frequency`** — metric then chốt của loại campaign này<br>② `Reach` là unique theo platform, không phải unique user trong game — không dùng làm mẫu số cho Unique Login được<br>③ Churn user là segment của game DB, platform chỉ nhận custom audience — cột A không tách được nhóm churn vs non-churn |
| **Post Boosting** | Engagement, Reaction, Likes, Comments, Shares, Saves, Follows, ProfileVisits, ThruPlay, VideoViewsP25–P100 | ① **Thiếu toàn bộ nhóm organic** (mục 5)<br>② Grain post-level chưa có<br>③ Google không áp dụng — cột D trống là đúng |

---

## 9. Cột A đề xuất — 52 metric, 6 nhóm

⚠ = non-additive, không SUM

### Nhóm 1 · Chi phí & phân phối

`Spend` · `Impression` · `Click` · `ClickDestination` · `Reach` ⚠ · `Frequency` ⚠ **[thêm]**

### Nhóm 2 · Chi phí đơn vị — pull từ platform, toàn bộ ⚠

`Cpm` · `Cpc` · `Cpp` · `Ctr` · `CostPerConversion` · `CostPerThruplay` · `AverageCpe` · `AverageCpv` · `AverageCost` · `CostPerInstall` **[thêm]** · `CostPerLinkClick` **[thêm]**

### Nhóm 3 · Chuyển đổi

`Conversion` + dimension `conversion_type` (thay cho In-app actions / Landing Page View / Lead) · `ConversionRate` ⚠ · `Installations` · `AdReach` ⚠

### Nhóm 4 · Tương tác — Post Boosting, FB + TikTok

`Engagement` · `EngagementRate` ⚠ · `Interactions` · `InteractionRate` ⚠ · `Reaction` · `Likes` · `Comments` · `Shares` · `Saves` · `Follows` · `ProfileVisits` ⚠ · `ProfileVisitsRate` ⚠

### Nhóm 5 · Video

`VideoPlayActions` · `VideoViews2s` · `VideoViews3s` · `VideoViews6s` · `VideoViews2sUnique` ⚠ *(FB only)* · `VideoViewsP25` · `VideoViewsP50` · `VideoViewsP75` · `VideoViewsP100` · `VideoViewRate` ⚠ · `AverageVideoPlay` ⚠ · `ThruPlay`

> **Về `ThruPlay`:** 3 platform định nghĩa khác nhau — FB 15s, TikTok 15s, **Google TrueView 30s** (skippable in-stream) hoặc **10s** (in-feed / Shorts), đúng như note ở ô A75 của tab. Gộp chung 1 cột rồi so CPV giữa 3 platform là so sai.
>
> Giữ `ThruPlay` nhưng **dùng nhóm `VideoViewsP25`–`P100` làm metric video so sánh chính** — nhóm này định nghĩa theo % độ dài video nên cả 3 platform giống nhau.

### Nhóm 6 · Organic — Post Boosting, nguồn data mới

`OrganicReach` ⚠ · `OrganicImpression` · `OrganicEngagement` · `OrganicVideoViews` · `FollowerGrowth`

---

## 10. Cột cần bổ sung vào tab `Metrics mapping`

Cột B `Metrics (PBI)` vẫn trống 100% — chưa chốt tên chuẩn thì Dev không có đích để map.

| Cột đề xuất | Nội dung |
|---|---|
| `Tên chuẩn PBI` | Tên metric trong semantic model |
| `Nhóm` | 1–6 theo mục 9 |
| **`Aggregation`** | `Sum` / `Non-additive` / `Recompute` — **cột quan trọng nhất**, ngăn 18 metric non-additive bị cộng sai |
| `Mẫu số` | Bắt buộc với metric tỷ lệ |
| `Grain thấp nhất` | ad / campaign / post / account |
| `Nguồn` | Ads API / Page Insights / TikTok Business |
| `Áp dụng cho campaign type` | UA / UE / Post Boosting |
| `Định nghĩa nghiệp vụ` | Một câu, bằng tiếng Việt |

---

## 11. Cần chốt với Dev song song

Không phải metric nhưng quyết định số đúng hay sai.

| # | Vấn đề | Vì sao |
|---|---|---|
| 1 | **Attribution window** | FB mặc định 7d-click / 1d-view, Google 30d-click + modeled, TikTok 7d-click / 1d-view. Không khoá cùng setting thì `Conversion` và `Installations` của 3 platform **vĩnh viễn không khớp nhau**, và cũng không khớp với game DB |
| 2 | **Restatement** | FB và Google sửa lại số trong vòng 28 ngày sau đó. Pipeline phải re-sync rolling 28 ngày, không chỉ append ngày mới |
| 3 | **Timezone + currency** | Mỗi ad account có timezone và currency riêng. Cần quy tắc convert và tỷ giá theo ngày nào |
| 4 | **Khoá join với game DB** | `campaign_id` hay chuẩn naming? Quyết định này chặn toàn bộ việc nối cột A với Install / Register / Login |

---

## Trạng thái

- [x] Review 61 metric cột A
- [ ] Chốt danh sách cuối với team OM
- [ ] Điền mapping cột C/D/E cho 55 dòng còn trống
- [ ] Bổ sung cột `Aggregation` + `Mẫu số` vào tab
- [ ] Quyết định nguồn data organic cho Post Boosting
