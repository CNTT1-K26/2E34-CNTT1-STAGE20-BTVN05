# KIẾN TRÚC BẢO MẬT ĐĂNG NHẬP VÀ XÁC THỰC VÍ TIỀN ĐIỆN TỬ (CRYPTO WALLET)

## 1. Bối cảnh & Yêu cầu Bảo mật

Ví tiền điện tử (Crypto Wallet) chứa các tài sản có giá trị cao và giao dịch đảo ngược bất khả thi (irreversible). Do đó, luồng đăng nhập yêu cầu cấp độ bảo mật cao nhất, bảo vệ người dùng trước các nguy cơ bị lộ mật khẩu (Password Breach), tấn công Mạng (Man-in-the-Middle) và chiếm đoạt phiên làm việc (Session Hijacking).

## 2. Kiến trúc Bảo mật 3 Lớp (Triple-Layer Defense Architecture)

Luồng xác thực kết hợp 3 lớp độc lập:
1. **Lớp 1 (Something You Know):** Mật khẩu chính (Master Password / PIN code) + Chuỗi muối (Salt).
2. **Lớp 2 (Something You Have):** Thuật toán TOTP (Time-based One-Time Password - RFC 6238) sinh mã 6 chữ số thay đổi mỗi 30 giây.
3. **Lớp 3 (Something You Are / Device Identity):** Xác thực Sinh trắc học (Biometrics: Fingerprint / Face ID) kết hợp Chữ ký khóa phần cứng thiết bị (Device Hardware-bound Key - Secure Enclave/TEE).

### Sơ đồ Luồng Dữ liệu Xác thực (Data Flow Diagram)

```
                       [Người dùng / Thiết bị Ví]
                                   │
  1. Nhập Master Password          │
  ┌────────────────────────────────┼────────────────────────────────┐
  │                                ▼                                │
  │                   [Lớp 1: Knowledge Check]                      │
  │                   Xác thực Mật khẩu & Hash                      │
  └────────────────────────────────┬────────────────────────────────┘
                                   │
                                   ▼
                       Mật khẩu ĐÚNG hay SAI?
                         ├─── SAI ──► [Từ chối Truy cập & Báo lỗi]
                         │
                         └─── ĐÚNG
                                   │
  2. Mở ứng dụng TOTP              │
  ┌────────────────────────────────┼────────────────────────────────┐
  │                                ▼                                │
  │                    [Lớp 2: TOTP Check]                          │
  │               Nhập Mã OTP 6 Số (RFC 6238)                       │
  └────────────────────────────────┬────────────────────────────────┘
                                   │
                                   ▼
                        Mã TOTP ĐÚNG hay SAI?
                         ├─── SAI ──► [Khóa tạm thời & Cảnh báo]
                         │
                         └─── ĐÚNG
                                   │
  3. Quét Vân tay / FaceID         │
  ┌────────────────────────────────┼────────────────────────────────┐
  │                                ▼                                │
  │                [Lớp 3: Hardware & Biometrics]                   │
  │            Xác thực Secure Enclave trên Thiết bị                │
  └────────────────────────────────┬────────────────────────────────┘
                                   │
                                   ▼
                         Sinh trắc học HỢP LỆ?
                         ├─── SAI ──► [Hủy phiên giao dịch]
                         │
                         └─── ĐÚNG
                                   │
                                   ▼
                     [CẤP ACCESS TOKEN & MỞ VÍ]
```

### Cơ chế Ngăn chặn Hacker (Kịch bản Hacker Có Mật khẩu):

Khi hacker bằng cách nào đó sở hữu Master Password (do lộ dữ liệu, dính bàn phím ma/keylogger...):
1. **Bị chặn ở Lớp 2 (TOTP):** Mã TOTP được tạo cục bộ trên thiết bị vật lý của nạn nhân. Hacker không có thiết bị di động của người dùng sẽ không thể lấy được mã 6 chữ số đang biến đổi theo thời gian thực.
2. **Bị chặn ở Lớp 3 (Secure Enclave / Biometrics):** Ngay cả khi hacker cài kịch bản tự động gõ cả Password và TOTP (qua tấn công Phishing thời gian thực), hệ thống vẫn đòi hỏi Chữ ký số giải mã từ chip bảo mật phần cứng (Hardware Security Module) chỉ kích hoạt khi người chủ ví quét Vân tay/FaceID thực tế trên thiết bị gốc.

---

## 3. Giải quyết Bẫy Ngoại lệ (Edge Case): Giao dịch trên Máy bay (Offline Mode)

### Câu hỏi: *Làm sao người dùng trên máy bay (không có Wifi / sóng di động) vẫn lấy được mã OTP để xác thực giao dịch?*

### Giải đáp & Cơ chế Kỹ thuật:

Thực tế, **thuật toán TOTP (Time-based One-Time Password - RFC 6238) hoàn toàn KHÔNG CẦN Internet hay sóng di động để tạo mã OTP.**

```
┌─────────────────────────────────────────────────────────────┐
│                 CƠ CHẾ HOẠT ĐỘNG OFFLINE CỦA TOTP           │
│                                                             │
│  1. Secret Key (Khóa bí mật) ──┐                            │
│                                ├──► [HMAC-SHA1] ──► Mã OTP   │
│  2. Unix Timestamp (Thời gian) ┘                    (6 Số)   │
│     (Lấy từ Đồng hồ Nội vi của Máy)                         │
└─────────────────────────────────────────────────────────────┘
```

#### Nguyên lý Kỹ thuật:
1. **Chia sẻ Secret Key từ trước:** Trong quá trình cài đặt 2FA ban đầu, máy chủ và ứng dụng Authenticator (Google Authenticator, Bitwarden, YubiKey) đã cùng lưu trữ một chuỗi khóa bí mật chung (`Secret Key / Seed`).
2. **Tính toán dựa trên Đồng hồ Nội vi (Hardware Clock):**
   * Thuật toán TOTP tính toán giá trị đếm thời gian: 
     $$T = \lfloor \frac{\text{Current Unix Time} - T_0}{X} \rfloor$$
     *(trong đó $X = 30$ giây).*
   * Mã OTP 6 chữ số được tạo ra bằng hàm băm mã hóa: 
     $$\text{TOTP} = \text{Truncate}(\text{HMAC-SHA-1}(\text{SecretKey}, T))$$
3. **Hoạt động Độc lập:** Cả điện thoại và máy chủ đều chạy đồng hồ hệ thống độc lập. Do đó, điện thoại ở chế độ máy bay (Airplane Mode) vẫn sử dụng đồng hồ nội vi của chip di động để tự tính ra mã OTP 6 số chính xác **100% khớp với mã mà máy chủ kỳ vọng**, không tốn một byte dữ liệu mạng nào.

#### Lưu ý triển khai cho Ví Crypto:
* **Xử lý lệch giờ (Clock Drift):** Nếu đồng hồ nội vi của điện thoại bị lệch vài giây do mất mạng lâu ngày, Server sẽ áp dụng cơ chế **Time Window Drift Compensation** (chấp nhận mã OTP của bước thời gian $T-1$ hoặc $T+1$) để giao dịch không bị gián đoạn.
* **Gửi Transaction Offline:** Đối với bản thân giao dịch Crypto (cần đẩy lên Blockchain), ứng dụng ví sẽ ký giao dịch offline bằng Private Key + TOTP xác thực, sau đó lưu vào bộ nhớ đệm (Queue) và tự động phát sóng (Broadcast) lên mạng lưới Blockchain ngay khi máy tính/điện thoại kết nối lại Internet.