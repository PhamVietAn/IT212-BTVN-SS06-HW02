# BÀI 2: Thực hành Xây dựng Thuật toán Khuyến mãi Phức tạp (Chain-of-thought - CoT)

## 1. Giải thích tầm quan trọng của thứ tự ưu tiên tính toán trong nghiệp vụ kế toán/tài chính

Trong các hệ thống phần mềm quản lý tài chính, kế toán và thương mại điện tử, thứ tự ưu tiên thực hiện các phép tính (Calculation Order of Precedence) đóng vai trò quyết định tính chính xác, hợp pháp và minh bạch của hệ thống. Chỉ một thay đổi nhỏ trong thứ tự áp dụng các khoản chiết khấu, thuế, hay phí cũng có thể dẫn đến những sai lệch tài chính nghiêm trọng.

Cụ thể, tầm quan trọng của thứ tự ưu tiên này thể hiện ở các mặt sau:

1.  **Tránh thất thoát tài chính cho doanh nghiệp:** 
    *   Ví dụ, nếu áp dụng Mã giảm giá (Coupon - ví dụ giảm 10% tối đa 100k) hoặc Chiết khấu thành viên (Loyalty - ví dụ giảm 5%) trên *Tổng giá trị gốc* trước khi áp dụng *Giảm giá sản phẩm*, doanh nghiệp sẽ bị thiệt hại nặng nề vì giảm giá chồng chéo lên phần tiền đã được giảm trực tiếp của sản phẩm.
    *   Thứ tự chuẩn mực luôn là: **Giá gốc -> Giảm giá trực tiếp của sản phẩm (Product Discount) -> Áp dụng Coupon trên tổng đơn đã giảm -> Áp dụng Chiết khấu thành viên (Loyalty Discount)**.

2.  **Đảm bảo tính hợp pháp về Thuế (Tax Compliance):**
    *   Thuế giá trị gia tăng (VAT) hoặc các loại thuế tiêu dùng khác phải được tính toán dựa trên số tiền thực tế mà khách hàng chi trả cho hàng hóa/dịch vụ (tức là sau khi đã trừ đi tất cả các loại khuyến mãi, chiết khấu được chấp nhận bởi cơ quan thuế).
    *   Nếu tính thuế VAT trước khi trừ coupon hoặc chiết khấu thành viên, hệ thống sẽ tính dư thuế VAT của khách hàng, vi phạm pháp luật về thuế và gây phiền hà khi xuất hóa đơn đỏ (VAT Invoice). Ngược lại, nếu tính thuế sau phí vận chuyển mà phí vận chuyển đó không chịu thuế VAT (hoặc có thuế suất khác), hóa đơn sẽ bị tính sai lệch.

3.  **Hợp lý hóa điều kiện miễn phí vận chuyển (Free Shipping Eligibility):**
    *   Điều kiện miễn phí vận chuyển thường được xác định dựa trên giá trị đơn hàng thực tế sau khi áp dụng chiết khấu nhưng trước thuế (`Total after discounts, before tax`).
    *   Nếu tính phí vận chuyển dựa trên giá trị gốc (chưa giảm giá), khách hàng có thể được miễn phí vận chuyển mặc dù số tiền họ thực tế chi trả là rất nhỏ. Điều này trực tiếp ăn mòn biên lợi nhuận của doanh nghiệp.

4.  **Tính chính xác tuyệt đối với kiểu dữ liệu tiền tệ:**
    *   Trong tài chính, việc làm tròn số (Rounding) ở mỗi bước trung gian phải tuân thủ nghiêm ngặt quy tắc làm tròn kế toán (thường là Half-Even Rounding - `RoundingMode.HALF_EVEN`). Sử dụng sai kiểu dữ liệu (như `double` hoặc `float` vốn bị sai số dấu phẩy động) kết hợp với sai thứ tự tính toán sẽ tạo ra các sai số lũy tiến vô cùng nguy hiểm.

---

## 2. Nội dung Prompt CoT tối ưu thiết kế

Dưới đây là Prompt sử dụng kỹ thuật Chain-of-thought (CoT) kết hợp khung 5 thành phần gửi cho AI:

```markdown
[ROLE]
Bạn là một Senior Backend Developer kiêm Chuyên gia tài chính hệ thống (Financial Software Engineer) chuyên thiết kế các module tính toán hóa đơn phức tạp trong TMĐT.

[GOAL]
Hãy thiết kế thuật toán tính tổng tiền thanh toán cuối cùng của đơn hàng trên SpeedyCart dựa trên các quy tắc khuyến mãi và tính tiền phức tạp, thực hiện dry-run kiểm thử bằng tay, và viết class Java `OrderCalculator` tối ưu.

[CONTEXT]
Khi khách hàng tiến hành thanh toán đơn hàng trên SpeedyCart, hệ thống phải áp dụng các quy tắc khuyến mãi và tính tổng số tiền cuối cùng. Các quy tắc tính tiền bao gồm:
1. Tính tổng tiền gốc của các mặt hàng trong giỏ hàng.
2. Áp dụng Giảm giá sản phẩm (Product Discount - nếu sản phẩm đó đang được giảm giá trực tiếp).
3. Áp dụng Mã giảm giá (Coupon Code - giảm 10% tổng đơn nhưng tối đa 100,000 VND, chỉ áp dụng cho đơn hàng có giá trị sau khi đã trừ đi giảm giá sản phẩm đạt tối thiểu 500,000 VND).
4. Áp dụng Chiết khấu thành viên (Loyalty Discount - hạng Gold giảm 5%, hạng Platinum giảm 10% tính trên tổng tiền đã trừ coupon).
5. Cộng Thuế VAT (10% tính trên tổng tiền sau khi đã trừ tất cả các chiết khấu trên).
6. Cộng Phí vận chuyển (Shipping Fee - tính mặc định là 30,000 VND cho khoảng cách dưới 10km, mỗi km tiếp theo cộng 5,000 VND. Miễn phí vận chuyển nếu tổng tiền đơn hàng sau chiết khấu và trước thuế lớn hơn 1,000,000 VND).

[CONSTRAINTS]
Để thực hiện nhiệm vụ này một cách chính xác tuyệt đối, bạn hãy suy nghĩ và thực hiện theo từng bước sau (Chain-of-thought):
Bước 1: Phân tích logic nghiệp vụ và xác định thứ tự thực hiện đúng của 6 quy tắc tính toán trên. Hãy giải thích tại sao thứ tự này là hợp lý dưới góc độ tài chính/kế toán.
Bước 2: Liệt kê công thức tính toán chi tiết và các điều kiện biên cho từng bước.
Bước 3: Thực hiện chạy thử bằng tay (Dry-run) bằng văn bản với ca kiểm thử cụ thể sau:
   - Giỏ hàng có:
     + 2 sản phẩm A (giá gốc 400,000 VND/sản phẩm, đang được giảm giá 10% trực tiếp trên mỗi sản phẩm).
     + 1 sản phẩm B (giá gốc 300,000 VND, không được giảm giá trực tiếp).
   - Khách hàng áp dụng mã coupon giảm trực tiếp 50,000 VND (yêu cầu giá trị đơn hàng sau giảm giá sản phẩm tối thiểu 500,000 VND).
   - Khách hàng có hạng thành viên là Gold (được giảm 5% trên tổng tiền đã trừ coupon).
   - Địa chỉ nhận hàng cách kho 12km.
   Hãy viết rõ từng phép tính toán trung gian và kết quả số tiền cuối cùng.
Bước 4: Sinh class Java `OrderCalculator` chứa phương thức `public OrderCalculationResult calculateTotal(Order order, double distance, String loyaltyRank, Coupon coupon)`. Bạn bắt buộc phải sử dụng kiểu dữ liệu `BigDecimal` cho tất cả các giá trị tiền tệ, phần trăm, phí vận chuyển để đảm bảo độ chính xác dấu phẩy động. Sử dụng quy tắc làm tròn `RoundingMode.HALF_UP` hoặc `RoundingMode.HALF_EVEN` với độ chính xác 2 chữ số thập phân (hoặc làm tròn về phần nguyên cho tiền VND).

[FORMAT]
Hãy chia rõ câu trả lời của bạn thành các phần:
- Phần 1: Phân tích logic và thứ tự ưu tiên tính toán (Bước 1 & Bước 2).
- Phần 2: Dry-run kiểm thử bằng tay với số liệu cụ thể (Bước 3).
- Phần 3: Mã nguồn Java class `OrderCalculator` hoàn chỉnh (Bước 4) kèm cấu trúc dữ liệu Input/Output rõ ràng để chạy được.
```

---

## 3. Minh chứng chạy thực tế với AI

Dưới đây là toàn bộ phản hồi chi tiết từ AI đối với nội dung Prompt thiết kế ở trên:

```markdown
Chào bạn, dưới đây là phân tích logic nghiệp vụ, kết quả dry-run chi tiết và mã nguồn Java hoàn chỉnh sử dụng `BigDecimal` cho bài toán tính toán đơn hàng SpeedyCart.

---

### PHẦN 1: PHÂN TÍCH LOGIC VÀ THỨ TỰ ƯU TIÊN TÍNH TOÁN

Để đảm bảo tính chính xác về mặt tài chính và tuân thủ các quy định về thuế pháp lý, thứ tự thực hiện các bước tính toán đơn hàng bắt buộc phải tuân theo trình tự sau:

#### Thứ tự thực hiện đúng:
1.  **Bước 1: Tính tổng tiền gốc (Subtotal)** của các mặt hàng trong giỏ hàng.
2.  **Bước 2: Áp dụng Giảm giá trực tiếp của sản phẩm (Product Discount)** để có được "Tổng tiền sau giảm giá sản phẩm" (đây là giá bán thực tế của hàng hóa).
3.  **Bước 3: Áp dụng Mã giảm giá (Coupon Code)** trên tổng tiền đã trừ giảm giá sản phẩm (nếu đủ điều kiện tối thiểu 500,000 VND).
4.  **Bước 4: Áp dụng Chiết khấu thành viên (Loyalty Discount)** dựa trên hạng thành viên (tính trên số tiền còn lại sau bước 3).
5.  **Bước 5: Tính Phí vận chuyển (Shipping Fee)** dựa trên khoảng cách địa lý và xét điều kiện miễn phí vận chuyển (kiểm tra xem tổng tiền sau chiết khấu ở Bước 4 đã lớn hơn 1,000,000 VND chưa).
6.  **Bước 6: Tính Thuế VAT (10%)** dựa trên tổng tiền sau tất cả chiết khấu (kết quả của Bước 4). *Lưu ý: Phí vận chuyển thông thường không chịu thuế VAT chung của hàng hóa hoặc được tính riêng, ở đây đề bài yêu cầu VAT tính trên tổng tiền sau khi trừ chiết khấu nên VAT sẽ tính trên giá trị sau Bước 4.*
7.  **Bước 7: Tính Tổng số tiền cuối cùng (Grand Total)** = (Tổng tiền sau chiết khấu) + VAT + Phí vận chuyển.

#### Giải thích lý do tài chính/kế toán:
*   **Product Discount trước Coupon:** Giảm giá sản phẩm là chương trình khuyến mãi trực tiếp trên từng mặt hàng. Giá trị đơn hàng thực tế sau khi giảm giá sản phẩm mới là căn cứ hợp lệ để kiểm tra xem đơn hàng có đủ điều kiện áp dụng Coupon (ngưỡng 500k) hay không.
*   **Loyalty Discount sau Coupon:** Loyalty discount là đặc quyền của khách hàng thân thiết, được áp dụng trên số tiền mà khách hàng thực sự phải trả sau các chương trình khuyến mãi đại trà (như giảm giá sản phẩm và coupon).
*   **VAT sau chiết khấu:** Thuế VAT chỉ được tính trên số tiền thực tế giao dịch bán hàng (sau khi đã trừ hết các khoản chiết khấu thương mại và giảm giá). Nếu tính VAT trước chiết khấu, khách hàng sẽ phải nộp thừa thuế VAT.
*   **Shipping Fee tách biệt khỏi VAT:** Phí vận chuyển là dịch vụ cộng thêm, không được tính gộp vào giá trị hàng hóa để chịu chiết khấu thương mại (Coupon hay Loyalty không được giảm trừ trên phí ship). Phí ship được cộng vào cuối cùng.

---

### PHẦN 2: DRY-RUN KIỂM THỬ BẰNG TAY (DỰA TRÊN CASE CỤ THỂ)

#### Dữ liệu đầu vào:
*   Sản phẩm A: 2 cái × Giá gốc 400,000 VND (Giảm giá sản phẩm 10% trực tiếp).
*   Sản phẩm B: 1 cái × Giá gốc 300,000 VND (Không giảm giá).
*   Coupon: Giảm trực tiếp 50,000 VND (áp dụng cho đơn hàng sau giảm giá sản phẩm >= 500,000 VND).
*   Hạng thành viên: Gold (Giảm 5%).
*   Khoảng cách: 12 km.

#### Các bước tính toán chi tiết:

1.  **Bước 1: Tính tổng tiền gốc (Subtotal)**
    *   Tiền gốc sản phẩm A = $2 \times 400,000 = 800,000$ VND.
    *   Tiền gốc sản phẩm B = $1 \times 300,000 = 300,000$ VND.
    *   Tổng tiền gốc = $800,000 + 300,000 = 1,100,000$ VND.

2.  **Bước 2: Áp dụng Giảm giá sản phẩm (Product Discount)**
    *   Mức giảm giá sản phẩm A = $10\% \times 400,000 = 40,000$ VND/sản phẩm.
    *   Tổng giảm giá sản phẩm A = $40,000 \times 2 = 80,000$ VND.
    *   Tổng giảm giá sản phẩm B = $0$ VND.
    *   Tổng tiền sau giảm giá sản phẩm = $1,100,000 - 80,000 = 1,020,000$ VND.

3.  **Bước 3: Áp dụng Mã giảm giá (Coupon Code)**
    *   Điều kiện áp dụng: Tổng tiền sau giảm giá sản phẩm ($1,020,000$ VND) phải $\ge 500,000$ VND.
    *   Đơn hàng thỏa mãn điều kiện ($1,020,000 \ge 500,000$).
    *   Áp dụng giảm trừ coupon: $1,020,000 - 50,000 = 970,000$ VND.

4.  **Bước 4: Áp dụng Chiết khấu thành viên (Loyalty Discount)**
    *   Hạng Gold được giảm 5% trên tổng tiền sau coupon.
    *   Giá trị giảm giá thành viên = $970,000 \times 5\% = 48,500$ VND.
    *   Tổng tiền sau tất cả chiết khấu = $970,000 - 48,500 = 921,500$ VND.

5.  **Bước 5: Tính Phí vận chuyển (Shipping Fee)**
    *   Điều kiện miễn phí ship: Tổng tiền sau chiết khấu và trước thuế ($921,500$ VND) phải $> 1,000,000$ VND.
    *   Vì $921,500 \le 1,000,000$, đơn hàng **không được miễn phí vận chuyển**.
    *   Công thức tính phí ship cho khoảng cách 12km:
        *   10km đầu tiên: $30,000$ VND.
        *   2km tiếp theo: $2 \times 5,000 = 10,000$ VND.
        *   Tổng phí ship = $30,000 + 10,000 = 40,000$ VND.

6.  **Bước 6: Tính Thuế VAT (10%)**
    *   VAT được tính trên tổng tiền sau chiết khấu: $921,500 \times 10\% = 92,150$ VND.

7.  **Bước 7: Tính Tổng số tiền thanh toán cuối cùng (Grand Total)**
    *   Grand Total = Tổng tiền sau chiết khấu + Thuế VAT + Phí vận chuyển.
    *   Grand Total = $921,500 + 92,150 + 40,000 = 1,053,650$ VND.

*   **Kết quả cuối cùng:** Khách hàng cần thanh toán **1,053,650 VND**.

---

### PHẦN 3: MÃ NGUỒN JAVA CLASS ORDERCALCULATOR

Dưới đây là mã nguồn Spring Boot/Java hoàn chỉnh sử dụng kiểu dữ liệu `BigDecimal` để đảm bảo độ chính xác tài chính cao nhất.

#### 1. Các Model đầu vào (Input models):

```java
package com.speedycart.model;

import java.math.BigDecimal;
import java.util.List;

public class Order {
    private List<OrderItem> items;

    public Order(List<OrderItem> items) {
        this.items = items;
    }

    public List<OrderItem> getItems() {
        return items;
    }
}
```

```java
package com.speedycart.model;

import java.math.BigDecimal;

public class OrderItem {
    private String productId;
    private BigDecimal unitPrice;
    private int quantity;
    private double discountRate; // ví dụ: 0.1 đại diện cho 10%

    public OrderItem(String productId, BigDecimal unitPrice, int quantity, double discountRate) {
        this.productId = productId;
        this.unitPrice = unitPrice;
        this.quantity = quantity;
        this.discountRate = discountRate;
    }

    public String getProductId() { return productId; }
    public BigDecimal getUnitPrice() { return unitPrice; }
    public int getQuantity() { return quantity; }
    public double getDiscountRate() { return discountRate; }
}
```

```java
package com.speedycart.model;

import java.math.BigDecimal;

public class Coupon {
    private String code;
    private BigDecimal discountAmount; // Ví dụ: 50,000 VND
    private BigDecimal minOrderValue;   // Ví dụ: 500,000 VND

    public Coupon(String code, BigDecimal discountAmount, BigDecimal minOrderValue) {
        this.code = code;
        this.discountAmount = discountAmount;
        this.minOrderValue = minOrderValue;
    }

    public String getCode() { return code; }
    public BigDecimal getDiscountAmount() { return discountAmount; }
    public BigDecimal getMinOrderValue() { return minOrderValue; }
}
```

#### 2. Model kết quả đầu ra (Output model):

```java
package com.speedycart.model;

import java.math.BigDecimal;

public class OrderCalculationResult {
    private BigDecimal subtotal;
    private BigDecimal productDiscountTotal;
    private BigDecimal couponDiscount;
    private BigDecimal loyaltyDiscount;
    private BigDecimal discountedTotal;
    private BigDecimal shippingFee;
    private BigDecimal vatAmount;
    private BigDecimal grandTotal;

    // Getters và Setters
    public BigDecimal getSubtotal() { return subtotal; }
    public void setSubtotal(BigDecimal subtotal) { this.subtotal = subtotal; }

    public BigDecimal getProductDiscountTotal() { return productDiscountTotal; }
    public void setProductDiscountTotal(BigDecimal productDiscountTotal) { this.productDiscountTotal = productDiscountTotal; }

    public BigDecimal getCouponDiscount() { return couponDiscount; }
    public void setCouponDiscount(BigDecimal couponDiscount) { this.couponDiscount = couponDiscount; }

    public BigDecimal getLoyaltyDiscount() { return loyaltyDiscount; }
    public void setLoyaltyDiscount(BigDecimal loyaltyDiscount) { this.loyaltyDiscount = loyaltyDiscount; }

    public BigDecimal getDiscountedTotal() { return discountedTotal; }
    public void setDiscountedTotal(BigDecimal discountedTotal) { this.discountedTotal = discountedTotal; }

    public BigDecimal getShippingFee() { return shippingFee; }
    public void setShippingFee(BigDecimal shippingFee) { this.shippingFee = shippingFee; }

    public BigDecimal getVatAmount() { return vatAmount; }
    public void setVatAmount(BigDecimal vatAmount) { this.vatAmount = vatAmount; }

    public BigDecimal getGrandTotal() { return grandTotal; }
    public void setGrandTotal(BigDecimal grandTotal) { this.grandTotal = grandTotal; }
}
```

#### 3. Class xử lý logic tính toán `OrderCalculator.java`:

```java
package com.speedycart.service;

import com.speedycart.model.*;
import java.math.BigDecimal;
import java.math.RoundingMode;

public class OrderCalculator {

    private static final BigDecimal VAT_RATE = new BigDecimal("0.10"); // Thuế VAT 10%
    private static final BigDecimal FREE_SHIP_THRESHOLD = new BigDecimal("1000000"); // Ngưỡng miễn ship 1,000,000 VND
    private static final BigDecimal BASE_SHIPPING_FEE = new BigDecimal("30000"); // Phí ship cơ bản 30,000 VND
    private static final double BASE_SHIPPING_DISTANCE = 10.0; // Khoảng cách cơ bản 10km
    private static final BigDecimal EXTRA_SHIPPING_FEE_PER_KM = new BigDecimal("5000"); // 5,000 VND/km tiếp theo

    public OrderCalculationResult calculateTotal(Order order, double distance, String loyaltyRank, Coupon coupon) {
        OrderCalculationResult result = new OrderCalculationResult();

        // 1. Tính tổng tiền gốc (Subtotal) và Tổng tiền giảm giá sản phẩm (Product Discount)
        BigDecimal subtotal = BigDecimal.ZERO;
        BigDecimal productDiscountTotal = BigDecimal.ZERO;

        for (OrderItem item : order.getItems()) {
            BigDecimal qty = new BigDecimal(item.getQuantity());
            BigDecimal itemSubtotal = item.getUnitPrice().multiply(qty);
            subtotal = subtotal.add(itemSubtotal);

            if (item.getDiscountRate() > 0) {
                BigDecimal discountRate = BigDecimal.valueOf(item.getDiscountRate());
                BigDecimal itemDiscount = item.getUnitPrice().multiply(discountRate).multiply(qty);
                productDiscountTotal = productDiscountTotal.add(itemDiscount);
            }
        }
        
        // Làm tròn VND không phần thập phân
        subtotal = subtotal.setScale(0, RoundingMode.HALF_UP);
        productDiscountTotal = productDiscountTotal.setScale(0, RoundingMode.HALF_UP);
        
        result.setSubtotal(subtotal);
        result.setProductDiscountTotal(productDiscountTotal);

        // Tổng tiền sau khi trừ giảm giá trực tiếp trên sản phẩm
        BigDecimal afterProductDiscount = subtotal.subtract(productDiscountTotal);

        // 2. Áp dụng Mã giảm giá (Coupon Code)
        BigDecimal couponDiscount = BigDecimal.ZERO;
        if (coupon != null) {
            // Kiểm tra điều kiện đơn hàng tối thiểu
            if (afterProductDiscount.compareTo(coupon.getMinOrderValue()) >= 0) {
                couponDiscount = coupon.getDiscountAmount();
                // Đảm bảo không giảm quá tổng giá trị đơn hàng hiện tại
                if (couponDiscount.compareTo(afterProductDiscount) > 0) {
                    couponDiscount = afterProductDiscount;
                }
            }
        }
        couponDiscount = couponDiscount.setScale(0, RoundingMode.HALF_UP);
        result.setCouponDiscount(couponDiscount);

        BigDecimal afterCoupon = afterProductDiscount.subtract(couponDiscount);

        // 3. Áp dụng Chiết khấu thành viên (Loyalty Discount)
        BigDecimal loyaltyDiscountRate = BigDecimal.ZERO;
        if ("GOLD".equalsIgnoreCase(loyaltyRank)) {
            loyaltyDiscountRate = new BigDecimal("0.05"); // Giảm 5%
        } else if ("PLATINUM".equalsIgnoreCase(loyaltyRank)) {
            loyaltyDiscountRate = new BigDecimal("0.10"); // Giảm 10%
        }

        BigDecimal loyaltyDiscount = afterCoupon.multiply(loyaltyDiscountRate).setScale(0, RoundingMode.HALF_UP);
        result.setLoyaltyDiscount(loyaltyDiscount);

        // Tổng tiền sau khi trừ tất cả các chiết khấu (để so sánh điều kiện freeship và tính VAT)
        BigDecimal discountedTotal = afterCoupon.subtract(loyaltyDiscount);
        result.setDiscountedTotal(discountedTotal);

        // 4. Tính Phí vận chuyển (Shipping Fee)
        BigDecimal shippingFee = BigDecimal.ZERO;
        // Miễn phí vận chuyển nếu tổng tiền đơn hàng sau chiết khấu và trước thuế lớn hơn 1,000,000 VND
        if (discountedTotal.compareTo(FREE_SHIP_THRESHOLD) <= 0) {
            if (distance <= BASE_SHIPPING_DISTANCE) {
                shippingFee = BASE_SHIPPING_FEE;
            } else {
                double extraDistance = Math.ceil(distance - BASE_SHIPPING_DISTANCE); // làm tròn lên số km
                BigDecimal extraFee = BigDecimal.valueOf(extraDistance).multiply(EXTRA_SHIPPING_FEE_PER_KM);
                shippingFee = BASE_SHIPPING_FEE.add(extraFee);
            }
        }
        shippingFee = shippingFee.setScale(0, RoundingMode.HALF_UP);
        result.setShippingFee(shippingFee);

        // 5. Tính Thuế VAT (10% tính trên tổng tiền sau chiết khấu)
        BigDecimal vatAmount = discountedTotal.multiply(VAT_RATE).setScale(0, RoundingMode.HALF_UP);
        result.setVatAmount(vatAmount);

        // 6. Tính Tổng số tiền cuối cùng (Grand Total)
        BigDecimal grandTotal = discountedTotal.add(vatAmount).add(shippingFee).setScale(0, RoundingMode.HALF_UP);
        result.setGrandTotal(grandTotal);

        return result;
    }
}
```
```
