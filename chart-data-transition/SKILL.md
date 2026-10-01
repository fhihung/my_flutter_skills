---
name: chart-data-transition
description: >-
  Animate mượt chart tự vẽ (CustomPainter) khi data VÀ trục cùng đổi: nội suy
  trên tỉ lệ 0..1 của vùng vẽ thay vì trên giá trị, nội suy trục để mốc 0 /
  lưới / nhãn tick trượt cùng nhịp với bar, crossfade tick, chụp trạng thái
  đang hiển thị để nối tiếp animation bị ngắt, nội suy bề rộng cột. Use when
  chart animation giật / nhảy khi đổi data, đổi filter / kỳ / Luỹ kế, trục X/Y
  tự co giãn theo data, bar trượt sai hướng, lưới hoặc nhãn trục nhảy, hoặc khi
  viết renderer chart custom mới có animation.
---

# Chart data transition (trục + data cùng đổi)

Áp dụng cho chart **tự vẽ** (CustomPainter) có trục số: bar ngang/dọc,
bullet, diverging, range, dot plot, lollipop… Chart dùng thư viện (Syncfusion,
fl_chart…) tự quản lý animation — chỉ áp dụng khi tự vẽ.

Quy ước trong code mẫu: `Axis` là model trục của bạn (`min`, `max`, `ticks`,
`span`, `fractionOf(v)` kẹp 0..1); đổi tên cho khớp project.

## Triệu chứng → nguyên nhân

| Triệu chứng | Nguyên nhân |
|---|---|
| Frame đầu bar co/giãn đột ngột rồi mới trượt | Trục nhảy ngay sang domain mới, bar lerp **giá trị cũ → mới** nhưng vẽ theo trục mới |
| Bar trượt sai hướng / dội ngược giữa chừng | Lerp riêng `min/max` trục và lerp riêng giá trị → vị trí pixel không đơn điệu |
| Mốc 0 / lưới không khớp đầu bar khi đang chạy | Lưới vẽ theo trục đích, bar vẽ theo trục khác |
| Đổi data liên tiếp → giật về giá trị cũ | Snapshot giá trị **đích cũ**, không phải thứ **đang hiển thị** |
| Vùng vẽ giật ngang | Cột text (giá trị / badge) đo theo data mới, đổi bề rộng tức thì |

## Nguyên tắc cốt lõi

**Animate vị trí, không animate giá trị.**

1. Đổi mỗi giá trị ra **tỉ lệ** `f = axis.fractionOf(v)` (0..1, không phụ thuộc
   bề rộng). Frame hiển thị = `lerp(fFrom, fTo, t)` — mỗi đầu bar đi thẳng từ
   chỗ đang thấy tới chỗ mới. `t = 0` giống hệt frame trước khi đổi data.
2. **Trục hiển thị** tại `t`: mọi giá trị cố định `v` nằm ở
   `lerp(from.f(v), to.f(v), t)`. Vì phép lerp hai hàm tuyến tính vẫn tuyến
   tính → biểu diễn lại được bằng một `Axis(min, max)` → mốc 0, lưới, nhãn
   tick tự khớp với bar (cùng công thức).
3. **Tick**: có ở cả 2 trục → trượt (opacity 1); chỉ ở trục cũ → mờ dần
   `1 - t`; chỉ ở trục mới → hiện dần `t`. Nhãn text: **lệch pha** (cũ mờ hết ở
   nửa đầu, mới hiện ở nửa sau) để chữ không đè nhau. Ngoài vùng vẽ → bỏ.
4. **Snapshot thứ đang hiển thị** khi data đổi (có thể giữa chừng animation
   trước): `from = lerp(fromCũ, toCũ, tHiệnTại)` — cho cả frame bar, trục,
   layout cột. Không bao giờ snapshot giá trị đích.
5. **Một tiến trình duy nhất** (`AnimationController` + curve) điều khiển bar,
   trục, lưới, nhãn, cột. Không dùng nhiều controller lệch nhịp.

## Workflow

```
- [ ] 1. Tách file motion: Frame (tỉ lệ) + lerpAxis + gridLines + tickLabels
- [ ] 2. Painter nhận to-values + from (Frame) + fromAxis + progress, tự lerp
- [ ] 3. State: snapshot "đang hiển thị" trong didUpdateWidget, restart controller
- [ ] 4. Header / nhãn trục + cột text: rebuild theo progress (AnimatedBuilder)
- [ ] 5. Cache layout đo text (không đo lại mỗi frame)
- [ ] 6. Test: t=0 trùng trạng thái cũ, ngắt giữa chừng, đổi trục, overflow
```

### 1. File motion (thuần Dart, không widget)

```dart
import 'dart:ui' show lerpDouble;

/// Vị trí theo tỉ lệ 0..1 — field null = không vẽ.
class BarFrame {
  final double zero;
  final double? value; // thêm field theo chart: prev, target, low, high…
  const BarFrame({required this.zero, this.value});

  factory BarFrame.of(Axis axis, {num? value}) => BarFrame(
    zero: axis.fractionOf(0),
    value: value == null ? null : axis.fractionOf(value),
  );

  /// Field mất ở [to] → bỏ; field mới (không có ở [from]) mọc từ mốc 0.
  static BarFrame lerp(BarFrame? from, BarFrame to, double t) {
    if (t >= 1) return to;
    final BarFrame start = from ?? BarFrame(zero: to.zero);
    double? field(double? a, double? b) =>
        b == null ? null : lerpDouble(a ?? start.zero, b, t);
    return BarFrame(
      zero: lerpDouble(start.zero, to.zero, t)!,
      value: field(start.value, to.value),
    );
  }
}

abstract final class ChartMotion {
  static double _raw(Axis a, num v) => a.span <= 0 ? 0 : (v - a.min) / a.span;

  /// Trục hiển thị tại [t]: f_t(v) = lerp(from.f(v), to.f(v), t).
  static Axis lerpAxis(Axis? from, Axis to, double t) {
    if (from == null || t >= 1 || from == to) return to;
    final double a = to.min, b = to.max;
    final double fa = lerpDouble(_raw(from, a), 0, t)!;
    final double fb = lerpDouble(_raw(from, b), 1, t)!;
    if (b <= a || (fb - fa).abs() < 1e-9) return to;
    final double scale = (fb - fa) / (b - a);
    final double min = a - fa / scale;
    return Axis(
      min: min,
      max: min + 1 / scale,
      ticks: t < 0.5 ? from.ticks : to.ticks,
    );
  }

  /// (fraction, opacity) cho lưới — bỏ mốc 0 (vẽ riêng) và tick ngoài vùng vẽ.
  static List<(double, double)> gridLines(Axis? from, Axis to, double t) {
    final Axis shown = lerpAxis(from, to, t);
    final bool animating = from != null && t < 1;
    final List<(double, double)> out = <(double, double)>[];
    void add(double tick, double opacity) {
      if (tick == 0 || opacity <= 0) return;
      final double f = _raw(shown, tick);
      if (f < -0.001 || f > 1.001) return;
      out.add((f.clamp(0.0, 1.0), opacity));
    }
    for (final double tick in to.ticks) {
      add(tick, !animating || from.ticks.contains(tick) ? 1 : t);
    }
    if (animating) {
      for (final double tick in from.ticks) {
        if (!to.ticks.contains(tick)) add(tick, 1 - t);
      }
    }
    return out;
  }
}
```

Nhãn tick header: cùng vị trí `shown.fractionOf(v) * trackWidth`, kẹp trong
vùng vẽ (`clamp(0, trackWidth - labelWidth)`), opacity lệch pha:

```dart
final double fadeIn = (t * 2 - 1).clamp(0.0, 1.0);
final double fadeOut = (1 - t * 2).clamp(0.0, 1.0);
// chung: 1 · chỉ mới: fadeIn · chỉ cũ: fadeOut · opacity 0 → không add
```

### 2. Painter

- Field public = **giá trị đích** (test + legend đọc được), thêm `from`
  (`BarFrame?`), `fromAxis` (`Axis?`), `progress` (`Animation<double>?`);
  `super(repaint: progress)` → bar repaint không cần rebuild widget.
- Trong `paint`: `t = progress?.value ?? 1`;
  `frame = BarFrame.lerp(from, BarFrame.of(axis, …), t)`;
  lưới từ `ChartMotion.gridLines(fromAxis, axis, t)` (màu
  `gridColor.withValues(alpha: gridColor.a * opacity)`); mốc 0 tại
  `frame.zero * width`, vẽ **sau** bar.
- Độ dài tối thiểu cho giá trị nhỏ (vd 2px) **chỉ áp khi đích ≠ 0** — đích 0 thì
  để bar co mượt về rỗng.
- `shouldRepaint` so cả `from`, `fromAxis`, `progress`.

### 3. State — snapshot + restart

```dart
Map<String, BarFrame> _fromFrames = {}; // key = id ổn định của dòng (label)
Axis? _fromAxis;
List<TickLabel>? _fromTickLabels;
Layout? _fromLayout;
Layout? _targetLayout, _shownLayout; // ghi lại mỗi frame build

double get _t => MediaQuery.disableAnimationsOf(context) ? 1 : _curve.value;

@override
void didUpdateWidget(covariant MyChart old) {
  super.didUpdateWidget(old);
  if (old.view == widget.view) return;
  _snapshotShown(old.view); // TRƯỚC khi reset hidden series / selection
  _controller.forward(from: 0);
}

void _snapshotShown(View oldView) {
  final Layout? target = _targetLayout;
  if (target == null) return;
  final double t = _t;
  _fromFrames = {
    for (final row in oldView.rows)
      row.id: BarFrame.lerp(
        _fromFrames[row.id],
        BarFrame.of(target.axis, value: visible ? row.value : null),
        t,
      ),
  };
  _fromAxis = ChartMotion.lerpAxis(_fromAxis, target.axis, t);
  _fromTickLabels =
      t < 0.5 && _fromTickLabels != null ? _fromTickLabels : target.tickLabels;
  _fromLayout = _shownLayout;
}
```

- Lần đầu hiện: `from == null` → bar mọc từ mốc 0, lưới đứng yên.
- Series đang ẩn (legend) → snapshot `null` cho field đó; hiện lại sau đổi
  data sẽ mọc từ 0.
- Toggle legend / chọn dòng **không** restart controller.

### 4. Bề rộng cột + header

Cột text đo theo data (giá trị dài/ngắn hơn) → nội suy bề rộng cột, vùng vẽ =
tổng − cột:

```dart
static Layout lerpColumns(Layout? from, Layout to, double t) {
  if (from == null || t >= 1) return to;
  // Bề rộng chart cũng đổi (xoay màn) → bỏ nội suy, tránh cột cũ tràn.
  if ((from.totalWidth - to.totalWidth).abs() > 0.5) return to;
  // lerpDouble từng cột; trackWidth = max(0, total - fixed(lerped))
}
```

Bọc phần header + rows trong `AnimatedBuilder(animation: _curve)` để cột +
nhãn tick rebuild theo `t`; ghi `_targetLayout` / `_shownLayout` trong builder.

### 5. Cache layout

`TextPainter` đo text tốn kém → cache layout theo key
`Object.hash(identityHashCode(view), width, textScaler, styles…)`; mỗi frame
animation chỉ lerp, không đo lại.

## Bẫy thường gặp

- Lerp `min/max` trục rồi map giá trị đã lerp → **không đơn điệu**, đúng là lỗi
  cần tránh. Luôn lerp tỉ lệ.
- `fractionOf` thường **kẹp** 0..1 — dùng bản `_raw` không kẹp để nội suy trục,
  chỉ kẹp khi vẽ.
- Key dòng phải ổn định (label / id), không dùng index nếu thứ tự có thể đổi.
- Không `hide()` OverlayPortal / tooltip trong lúc build — hoãn sang
  `addPostFrameCallback`.
- Respect `MediaQuery.disableAnimationsOf` → `progress = null`, `t = 1`.
- Màu lấy từ theme / design token của project; opacity bằng
  `withValues(alpha: …)` (không `withOpacity` — deprecated).

## Test bắt buộc

```dart
// t = 0 trùng trạng thái cũ: frame bắt đầu = vị trí cũ theo trục cũ
expect(p.from!.value, closeTo(p.fromAxis!.fractionOf(oldValue), 1e-9));

// giữa chừng nằm giữa from và to (đơn điệu)
final double mid = BarFrame.lerp(p.from, p.toFrame, p.progress!.value).value!;
expect(mid, inExclusiveRange(fromF, toF));

// đổi data lần nữa giữa chừng → from mới nằm giữa from/to cũ (không giật)

// lerpAxis: t=0 ≈ trục cũ, t=1 == trục mới; mọi v: shown.f(v) == lerp(f0, f1, t)
// gridLines: tick chung opacity 1, cũ 1-t, mới t, ngoài vùng vẽ bị bỏ
```

Test layout (overflow, nhãn tick không lấn cột) phải `pumpAndSettle()` sau
khi đổi data — giữa chừng nhãn cũ/mới cùng tồn tại là đúng thiết kế.
Chạy thêm test overflow ở width hẹp + text scale lớn khi đổi data **và** width
cùng lúc.
