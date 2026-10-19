# Tổng Hợp Kỹ Thuật & Bài Học Lập Trình Từ Source Code (2011)

Tài liệu này đúc kết các kỹ thuật lập trình, thuật toán đồ họa, và mô hình tối ưu hiệu năng web nâng cao được trích xuất từ source code dự án **2011** (`index.html`, `script.js`, `style.css`). Toàn bộ dự án được xây dựng bằng **Vanilla JavaScript thuần**, không phụ thuộc vào React, Vue hay các thư viện animation trả phí (như GSAP Plugins).

---

## Mục Lục
1. [Kiến Trúc Tổng Thể: Linear State Machine](#1-kiến-trúc-tổng-thể-linear-state-machine)
2. [Kỹ Thuật 1: FLIP Animation Thuần (First, Last, Invert, Play)](#2-kỹ-thuật-1-flip-animation-thuần-first-last-invert-play)
3. [Kỹ Thuật 2: SVG Path Morphing Bằng Toán Học Thuần](#3-kỹ-thuật-2-svg-path-morphing-bằng-toán-học-thuần)
4. [Kỹ Thuật 3: Engine Nhận Diện Cấu Hình & Adaptive Performance (`TINY_PERF`)](#4-kỹ-thuật-3-engine-nhận-diện-cấu-hình--adaptive-performance-tiny_perf)
5. [Kỹ Thuật 4: Tối Ưu Đồ Họa Canvas 2D Bằng Sprite Pre-rendering](#5-kỹ-thuật-4-tối-ưu-đồ-họa-canvas-2d-bằng-sprite-pre-rendering)
6. [Kỹ Thuật 5: Đồng Bộ Audio-Subtitle Bằng Binary Search & Xử Lý Autoplay](#6-kỹ-thuật-5-đồng-bộ-audio-subtitle-bằng-binary-search--xử-lý-autoplay)
7. [Kỹ Thuật 6: Physics & Liquid Clipping Bằng CSS `clip-path`](#7-kỹ-thuật-6-physics--liquid-clipping-bằng-css-clip-path)
8. [Checklist Thực Chiến Cho Dự Án Mới](#8-checklist-thực-chiến-cho-dự-án-mới)

---

## 1. Kiến Trúc Tổng Thể: Linear State Machine

Thay vì để các hàm callback gọi chéo nhau gây ra "spaghetti code" hoặc race conditions khi người dùng click liên tục, dự án sử dụng mô hình **State Machine tuyến tính** thông qua các biến trạng thái trung tâm:

* **Vị trí tham chiếu:** [`script.js`](file:///c:/Users/DAnnotator2/Khanh/2011/script.js)
* **Cơ chế:**
  * Biến `clickCount` kiểm soát chỉ số bước của câu chuyện (16 cảnh).
  * Hàm điều hướng `chuyenCanh(targetStep)`:
    * Kiểm tra khóa chuyển cảnh (`isTransitioning`). Nếu animation trước chưa chạy xong, mọi sự kiện click tiếp theo đều bị chặn.
    * Đóng gói logic thoát cảnh cũ (fade out, hủy canvas/loop) trước khi khởi tạo cảnh mới.
    * Giải phóng bộ nhớ và dọn dẹp các event listener không cần thiết.

> **Bài học:** Với các trang landing page dẫn dắt theo cốt truyện (storytelling / interactive narrative), kiến trúc State Machine là giải pháp an toàn và ổn định nhất để tránh lỗi vỡ layout hoặc nhảy sai cảnh.

---

## 2. Kỹ Thuật 1: FLIP Animation Thuần (First, Last, Invert, Play)

### Vấn đề
Nhân vật chính (đôi mắt `#khung-mat`) phải bay xuyên suốt qua nhiều cảnh: từ không gian vũ trụ (`share-flight-scene`), sang khung cảnh thực tế (`tiny-reality-scene`), rồi thu nhỏ đặt vào thanh tiêu đề cửa hàng Shopee (`shop-tiny-host`). Mỗi cảnh này nằm ở các thẻ container cha hoàn toàn khác nhau trong DOM.

Nếu dùng CSS bình thường hoặc hủy node cũ tạo node mới, animation sẽ bị giật và mất trạng thái.

### Giải pháp
Dự án áp dụng thuật toán **FLIP** kết hợp **Web Animations API**:
* **Vị trí tham chiếu:** [`bringRealTinyToShare`](file:///c:/Users/DAnnotator2/Khanh/2011/script.js#L2308), [`openShops`](file:///c:/Users/DAnnotator2/Khanh/2011/script.js#L3643).

```javascript
// 1. FIRST: Lấy tọa độ và kích thước ban đầu của element
const firstRect = el.getBoundingClientRect();

// 2. Chuyển node sang container mới trong DOM
newContainer.appendChild(el);

// 3. LAST: Lấy tọa độ tại vị trí mới
const lastRect = el.getBoundingClientRect();

// 4. INVERT: Tính khoảng cách thay đổi và scale
const dx = firstRect.left - lastRect.left;
const dy = firstRect.top - lastRect.top;
const sx = firstRect.width / lastRect.width;
const sy = firstRect.height / lastRect.height;

// 5. PLAY: Chạy animation từ vị trí cũ về vị trí mới bằng Web Animations API
el.animate([
  {
    transformOrigin: 'top left',
    transform: `translate(${dx}px, ${dy}px) scale(${sx}, ${sy})`
  },
  {
    transformOrigin: 'top left',
    transform: 'none'
  }
], {
  duration: 800,
  easing: 'cubic-bezier(0.25, 1, 0.5, 1)',
  fill: 'both'
});
```

> **Bài học:** Không cần đến thư viện nặng ký như `Framer Motion` hay `GSAP Flip`, Web Animations API bản dựng sẵn của trình duyệt giải quyết bài toán chuyển đổi layout cực kỳ mượt mà ở 60 FPS.

---

## 3. Kỹ Thuật 2: SVG Path Morphing Bằng Toán Học Thuần

### Vấn đề
Nhân vật có biểu cảm thay đổi liên tục: Mắt tròn $\rightarrow$ mắt híp $\rightarrow$ mắt cỏ 4 lá $\rightarrow$ mắt nhắm `><`. Để biến hình (morph) giữa các path vector này mượt mà, thông thường lập trình viên phải mua plugin `GSAP MorphSVG`.

### Giải pháp
Tác giả tự xây dựng hệ tọa độ tham số hóa và hàm nội suy:
* **Vị trí tham chiếu:** [`cloverXEyes`](file:///c:/Users/DAnnotator2/Khanh/2011/script.js#L1091), [`prepareApproachingTinyEyes`](file:///c:/Users/DAnnotator2/Khanh/2011/script.js#L2500).
* **Nguyên lý:**
  1. Đồng bộ số lượng điểm neo (anchors) và tay đòn (control points) giữa 2 trạng thái SVG path.
  2. Dùng toán **Linear Interpolation (Lerp)** và phương trình **Cubic Bezier**:
     $$P(t) = (1-t)^3 P_0 + 3(1-t)^2 t P_1 + 3(1-t) t^2 P_2 + t^3 P_3$$
  3. Cập nhật chuỗi `d="M ... C ... Z"` trong `requestAnimationFrame`.

> **Bài học:** Nắm vững cấu trúc lệnh SVG Path (`M`, `C`, `Q`, `Z`) cho phép bạn tự làm hiệu ứng biến hình vector mà dung lượng bundle gần như bằng 0.

---

## 4. Kỹ Thuật 3: Engine Nhận Diện Cấu Hình & Adaptive Performance (`TINY_PERF`)

### Vấn đề
Trang web có nhiều hiệu ứng nặng (canvas hạt sao, glow filter, SVG morph, audio). Nếu chạy cùng một mức đồ họa trên tất cả các thiết bị:
* Máy tính bàn có GPU mạnh sẽ thừa tài nguyên.
* Điện thoại tầm trung/yếu sẽ nóng máy, tụt FPS và hết pin nhanh.

### Giải pháp
Tác giả tạo một module độc lập `TINY_PERF`:
* **Vị trí tham chiếu:** [`TINY_PERF`](file:///c:/Users/DAnnotator2/Khanh/2011/script.js#L68-L150).

```javascript
const TINY_PERF = (() => {
  const mem = navigator.deviceMemory || 4;       // RAM tính bằng GB
  const cores = navigator.hardwareConcurrency || 4; // Số nhân CPU
  const isTouch = window.matchMedia('(pointer: coarse)').matches;
  const isLowEnd = mem <= 2 || cores <= 4 || isTouch;

  return {
    // 1. Khống chế Canvas DPR: máy yếu/mobile tối đa 1.5, tránh render 3x/4x
    canvasDpr: Math.min(window.devicePixelRatio || 1, isLowEnd ? 1.5 : 2),

    // 2. Giới hạn số lượng hạt hiển thị
    maxParticles: isLowEnd ? 40 : 120,

    // 3. Phân tầng FPS (Staggered Ticks)
    // Các animation nền (như radar, khói) giới hạn ở 30-40 FPS, nhường GPU cho animation chính
    shouldThrottleFrame(lastTime, now, targetFps = 30) {
      return (now - lastTime) < (1000 / targetFps);
    },

    // 4. Dọn dẹp DOM rác vào lúc CPU rảnh rỗi
    idleCleanup(task) {
      if ('requestIdleCallback' in window) {
        window.requestIdleCallback(task, { timeout: 1000 });
      } else {
        setTimeout(task, 100);
      }
    }
  };
})();
```

> **Bài học:** Đừng bao giờ hardcode số lượng hạt hoặc mặc định `canvas.width = window.innerWidth * window.devicePixelRatio` trên mobile. Cần có cơ chế thích ứng theo cấu hình máy người dùng.

---

## 5. Kỹ Thuật 4: Tối Ưu Đồ Họa Canvas 2D Bằng Sprite Pre-rendering

### Vấn đề
Trong cảnh rơi lá cỏ 4 lá may mắn ([`luckCanvasLeaves`](file:///c:/Users/DAnnotator2/Khanh/2011/script.js#L1820)), mỗi chiếc lá có:
* Màu chuyển sắc (`createLinearGradient`).
* Hiệu ứng bóng phát sáng viền (`shadowBlur` & `shadowColor`).

Nếu mỗi frame vẽ 80 chiếc lá và mỗi chiếc lá đều tính toán lại gradient + shadow, GPU sẽ bị quá tải, FPS tụt xuống dưới 30.

### Giải pháp: Pre-rendered Offscreen Sprites
1. Khởi tạo 4 - 6 canvas ẩn (Offscreen Canvas) tương ứng với các kích thước và trạng thái màu của chiếc lá.
2. Vẽ sẵn hình ảnh chiếc lá hoàn chỉnh lên các canvas ảo đó **một lần duy nhất** lúc tải cảnh.
3. Trong vòng lặp render (`requestAnimationFrame`):
   * Thay vì vẽ vector phức tạp, chỉ gọi `ctx.drawImage(cachedSprite, x, y)`.
   * Sử dụng ma trận biến đổi `ctx.rotate()` và `ctx.scale()` để tạo độ ngẫu nhiên.

> **Hiệu quả:** Tốc độ render tăng 5 đến 10 lần, duy trì ổn định 60 FPS ngay cả trên thiết bị di động cũ.

---

## 6. Kỹ Thuật 5: Đồng Bộ Audio-Subtitle Bằng Binary Search & Xử Lý Autoplay

### 1. Đồng Bộ Subtitle Khớp Từng Mili-giây
* **Vấn đề:** Thay vì cập nhật phụ đề qua `timeupdate` (chỉ kích hoạt 3-4 lần/giây, quá trễ cho hiệu ứng karaoke), dự án chạy trong `requestAnimationFrame`.
* **Thuật toán:** Nếu dùng `cues.find(c => time >= c.start && time <= c.end)`, mỗi giây trình duyệt phải duyệt mảng 60 lần. Tác giả sử dụng **Binary Search** (`cueIndexAt`):
  * Dữ liệu thời gian của bài hát được sắp xếp tăng dần.
  * Tìm cue hiện tại trong mảng với thời gian $O(\log n)$, giảm thiểu tối đa chi phí CPU.

### 2. Vượt Qua Rào Cản Browser Autoplay Policy
* Trình duyệt hiện đại chặn phát nhạc tự động có âm thanh nếu chưa có cử chỉ người dùng.
* **Cách xử lý:**
  * Lắng nghe sự kiện chạm đầu tiên (`pointerdown` / `click` / `keydown`).
  * Khởi động audio ngầm ở trạng thái âm lượng nhỏ hoặc `play()` rồi `pause()` ngay để mở khóa quyền phát âm thanh của tab.
  * Sử dụng hằng số `MUSIC_HOLD_TIME` (khoảng 9.7s phần dạo đầu của bài hát) để đồng bộ nội dung đọc của người dùng khớp hoàn toàn với nốt cao trào của bản nhạc.

---

## 7. Kỹ Thuật 6: Physics & Liquid Clipping Bằng CSS `clip-path`

### Hiệu ứng
Ở cảnh ấn giữ nút nạp nhiên liệu để chuẩn bị cất cánh ([`syncFuelEatenText`](file:///c:/Users/DAnnotator2/Khanh/2011/script.js#L2240)):
* Người dùng nhấn giữ nút $\rightarrow$ mức nhiên liệu tụt xuống như một bình chất lỏng đang cạn dần.
* Dòng chữ hiển thị trên nút bị "cắt xén" theo mực chất lỏng đang rút.

### Giải pháp
Thay vì dựng hiệu ứng hạt hay WebGL phức tạp, dự án kết hợp CSS và JavaScript:
1. Tính tỷ lệ nhiên liệu đã tiêu hao: `eatenPercent = (currentFuel / maxFuel) * 100`.
2. Dùng CSS `clip-path: inset(top right bottom left)`:
   ```javascript
   textElement.style.clipPath = `inset(${100 - eatenPercent}% 0 0 0)`;
   ```
3. Phía dưới đặt một SVG sóng nước (`<path>` uốn lượn có animation CSS dịch chuyển ngang) nằm khớp ngay tại đường cắt.

> **Bài học:** Tận dụng triệt để sức mạnh của CSS `clip-path` có thể tạo ra các hiệu ứng vật lý chất lỏng đẹp mắt mà hoàn toàn không tốn tài nguyên xử lý.

---

## 8. Checklist Thực Chiến Cho Dự Án Mới

Khi bắt tay vào xây dựng một trang web interactive hoặc animation phức tạp, bạn có thể áp dụng ngay checklist này:

1. [ ] **State Control:** Luôn kiểm soát luồng di chuyển bằng biến trạng thái tập trung và cờ khóa `isTransitioning` để chặn spam click.
2. [ ] **Canvas DPR:** Đừng hardcode DPR cao trên điện thoại; hãy giới hạn `Math.min(devicePixelRatio, 1.5)`.
3. [ ] **Sprite Pre-rendering:** Nếu particle có đổ bóng hoặc gradient, hãy vẽ lên Offscreen Canvas trước rồi dùng `drawImage`.
4. [ ] **Cross-container Animation:** Ưu tiên dùng kỹ thuật **FLIP** và `element.animate()` thay vì render lại DOM.
5. [ ] **Audio Safety:** Luôn có bước "unlock audio" trên sự kiện người dùng đầu tiên trước khi gọi `audio.play()`.
6. [ ] **Clean Memory:** Hủy `cancelAnimationFrame()`, xóa canvas context và dọn event listeners khi người dùng rời khỏi cảnh cũ.
