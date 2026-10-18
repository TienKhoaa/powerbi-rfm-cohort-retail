# Công thức DAX và Power Query

Tài liệu này tổng hợp mọi công thức dùng trong dashboard `RFM_Dashboard.pbix`.
Nếu máy dùng cài đặt khu vực tiếng Việt, đổi dấu `,` giữa các tham số thành `;` khi dán vào Power BI.

## 1. Power Query (bảng Fact_Sales)

Các bước lọc và thêm cột, theo thứ tự:

```
// Loại dòng thiếu khách hàng
= Table.SelectRows(#"Changed Type1", each ([CustomerID] <> null))

// Loại hóa đơn hủy
= Table.SelectRows(#"Filtered Rows", each not Text.StartsWith([Invoice], "C"))

// Giữ số lượng dương
= Table.SelectRows(#"Filtered Rows1", each [Quantity] > 0)

// Giữ giá dương
= Table.SelectRows(#"Filtered Rows2", each [Price] > 0)

// Doanh thu
Revenue = [Quantity] * [Price]

// Ngày không có giờ
InvoiceDay = DateTime.Date([InvoiceDate])

// Loại ngoại lai
= Table.SelectRows(#"Changed Type3", each [Quantity] < 50000)
```

## 2. Bảng tính toán (Calculated tables)

### Dim_Date

```
Dim_Date =
ADDCOLUMNS(
    CALENDAR(
        DATE(YEAR(MIN(Fact_Sales[InvoiceDay])), 1, 1),
        DATE(YEAR(MAX(Fact_Sales[InvoiceDay])), 12, 31)
    ),
    "Year", YEAR([Date]),
    "MonthNo", MONTH([Date]),
    "Month", FORMAT([Date], "MMM"),
    "YearMonth", FORMAT([Date], "YYYY-MM")
)
```

Cột `Month` được sắp xếp theo `MonthNo`. Bảng được đánh dấu là **Date table**.

### Dim_Customer

```
Dim_Customer =
SUMMARIZE(
    Fact_Sales,
    Fact_Sales[CustomerID],
    "Country", MAX(Fact_Sales[Country])
)
```

## 3. Cột tính toán trong Dim_Customer

```
LastPurchaseDate = CALCULATE(MAX(Fact_Sales[InvoiceDay]))
```

```
FirstPurchaseDate = CALCULATE(MIN(Fact_Sales[InvoiceDay]))
```

```
CohortMonth = FORMAT(Dim_Customer[FirstPurchaseDate], "YYYY-MM")
```

```
Recency =
VAR SnapshotDate =
    CALCULATE(MAX(Fact_Sales[InvoiceDay]), ALL(Fact_Sales)) + 1
RETURN
    DATEDIFF(Dim_Customer[LastPurchaseDate], SnapshotDate, DAY)
```

```
Frequency = CALCULATE(DISTINCTCOUNT(Fact_Sales[Invoice]))
```

```
Monetary = CALCULATE(SUM(Fact_Sales[Revenue]))
```

### Điểm RFM (quintile 1-5)

```
R_Score =
VAR r = Dim_Customer[Recency]
RETURN
SWITCH(
    TRUE(),
    r <= PERCENTILE.INC(Dim_Customer[Recency], 0.2), 5,
    r <= PERCENTILE.INC(Dim_Customer[Recency], 0.4), 4,
    r <= PERCENTILE.INC(Dim_Customer[Recency], 0.6), 3,
    r <= PERCENTILE.INC(Dim_Customer[Recency], 0.8), 2,
    1
)
```

```
F_Score =
VAR f = Dim_Customer[Frequency]
RETURN
SWITCH(
    TRUE(),
    f <= PERCENTILE.INC(Dim_Customer[Frequency], 0.2), 1,
    f <= PERCENTILE.INC(Dim_Customer[Frequency], 0.4), 2,
    f <= PERCENTILE.INC(Dim_Customer[Frequency], 0.6), 3,
    f <= PERCENTILE.INC(Dim_Customer[Frequency], 0.8), 4,
    5
)
```

```
M_Score =
VAR m = Dim_Customer[Monetary]
RETURN
SWITCH(
    TRUE(),
    m <= PERCENTILE.INC(Dim_Customer[Monetary], 0.2), 1,
    m <= PERCENTILE.INC(Dim_Customer[Monetary], 0.4), 2,
    m <= PERCENTILE.INC(Dim_Customer[Monetary], 0.6), 3,
    m <= PERCENTILE.INC(Dim_Customer[Monetary], 0.8), 4,
    5
)
```

### Phân nhóm khách hàng

Thứ tự các điều kiện quan trọng vì `SWITCH(TRUE())` dừng ở điều kiện đúng đầu tiên.

```
Segment =
VAR R = Dim_Customer[R_Score]
VAR F = Dim_Customer[F_Score]
RETURN
SWITCH(
    TRUE(),
    R >= 4 && F >= 4, "Champions",
    R >= 3 && F >= 3, "Loyal",
    R >= 4 && F <= 2, "New Customers",
    R <= 2 && F >= 3, "At Risk",
    R <= 2 && F <= 2, "Lost",
    "Need Attention"
)
```

## 4. Cột tính toán trong Fact_Sales

```
CohortIndex =
DATEDIFF(
    RELATED(Dim_Customer[FirstPurchaseDate]),
    Fact_Sales[InvoiceDay],
    MONTH
)
```

## 5. Measures (bảng _Measures)

```
Total Revenue = SUM(Fact_Sales[Revenue])
```

```
Total Orders = DISTINCTCOUNT(Fact_Sales[Invoice])
```

```
Total Customers = DISTINCTCOUNT(Fact_Sales[CustomerID])
```

```
Avg Order Value = DIVIDE([Total Revenue], [Total Orders])
```

```
Customer % =
DIVIDE(
    [Total Customers],
    CALCULATE([Total Customers], ALL(Dim_Customer[Segment]))
)
```

```
Revenue % =
DIVIDE(
    [Total Revenue],
    CALCULATE([Total Revenue], ALL(Dim_Customer[Segment]))
)
```

```
Cohort Size =
CALCULATE(
    [Total Customers],
    REMOVEFILTERS(Fact_Sales[CohortIndex]),
    Fact_Sales[CohortIndex] = 0
)
```

```
Retention % = DIVIDE([Total Customers], [Cohort Size])
```

Hai measure dạng chữ dùng riêng cho card KPI để hiển thị đủ số (không bị rút gọn thành K):

```
Total Orders (Display) = FORMAT([Total Orders], "#,0")
```

```
Total Customers (Display) = FORMAT([Total Customers], "#,0")
```

## 6. Ghi chú thiết kế

- Dùng **cột tính sẵn** cho RFM vì đây là thuộc tính của khách hàng, cần làm slicer và drill-through theo nhóm. Đổi lại, RFM không đổi theo bộ lọc năm.
- Dùng `DIVIDE` thay cho dấu `/` để tránh lỗi chia cho 0.
- Dùng `ALL(Dim_Customer[Segment])` để lấy tổng toàn bộ làm mẫu số khi tính tỷ trọng từng nhóm.
- Ngày chốt dữ liệu là ngày cuối cùng của dataset cộng 1, không dùng `TODAY()` vì dữ liệu là dữ liệu lịch sử.
