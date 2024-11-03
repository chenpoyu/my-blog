---
layout: post
title: "Flutter 自訂動畫：從基礎到進階"
date: 2024-11-04 10:40:00 +0800
categories: [行動開發, Flutter]
tags: [Flutter, Animation, CustomPainter]
---

動畫讓 App 更生動。這週深入研究 Flutter 動畫系統，從簡單的隱式動畫到複雜的自訂動畫。

## 隱式動畫 (Implicit Animations)

最簡單的動畫方式。

### AnimatedContainer

```dart
class ChargingStationCard extends StatefulWidget {
  @override
  _ChargingStationCardState createState() => _ChargingStationCardState();
}

class _ChargingStationCardState extends State<ChargingStationCard> {
  bool _isExpanded = false;
  
  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _isExpanded = !_isExpanded),
      child: AnimatedContainer(
        duration: Duration(milliseconds: 300),
        curve: Curves.easeInOut,
        height: _isExpanded ? 200 : 100,
        decoration: BoxDecoration(
          color: _isExpanded ? Colors.blue : Colors.grey[300],
          borderRadius: BorderRadius.circular(_isExpanded ? 20 : 10),
          boxShadow: [
            BoxShadow(
              color: Colors.black.withOpacity(_isExpanded ? 0.3 : 0.1),
              blurRadius: _isExpanded ? 10 : 5,
              offset: Offset(0, _isExpanded ? 5 : 2),
            ),
          ],
        ),
        child: Center(
          child: Text(
            _isExpanded ? '充電站詳情' : '點擊展開',
            style: TextStyle(
              color: _isExpanded ? Colors.white : Colors.black,
              fontSize: _isExpanded ? 18 : 14,
            ),
          ),
        ),
      ),
    );
  }
}
```

### AnimatedOpacity

淡入淡出效果。

```dart
class ChargingStatusIndicator extends StatefulWidget {
  final bool isCharging;
  
  const ChargingStatusIndicator({required this.isCharging});
  
  @override
  _ChargingStatusIndicatorState createState() => 
      _ChargingStatusIndicatorState();
}

class _ChargingStatusIndicatorState 
    extends State<ChargingStatusIndicator> {
  bool _visible = false;
  
  @override
  void initState() {
    super.initState();
    // 延遲顯示
    Future.delayed(Duration(milliseconds: 100), () {
      if (mounted) setState(() => _visible = true);
    });
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedOpacity(
      opacity: _visible ? 1.0 : 0.0,
      duration: Duration(milliseconds: 500),
      child: Container(
        width: 20,
        height: 20,
        decoration: BoxDecoration(
          color: widget.isCharging ? Colors.green : Colors.grey,
          shape: BoxShape.circle,
        ),
      ),
    );
  }
}
```

### AnimatedCrossFade

兩個 Widget 之間切換。

```dart
class ChargingButton extends StatefulWidget {
  @override
  _ChargingButtonState createState() => _ChargingButtonState();
}

class _ChargingButtonState extends State<ChargingButton> {
  bool _isCharging = false;
  
  @override
  Widget build(BuildContext context) {
    return AnimatedCrossFade(
      firstChild: ElevatedButton(
        onPressed: () => setState(() => _isCharging = true),
        child: Text('開始充電'),
      ),
      secondChild: ElevatedButton(
        onPressed: () => setState(() => _isCharging = false),
        style: ElevatedButton.styleFrom(backgroundColor: Colors.red),
        child: Text('停止充電'),
      ),
      crossFadeState: _isCharging 
          ? CrossFadeState.showSecond 
          : CrossFadeState.showFirst,
      duration: Duration(milliseconds: 300),
    );
  }
}
```

## 顯式動畫 (Explicit Animations)

需要 AnimationController 控制。

### 基礎設定

```dart
class ChargingProgressIndicator extends StatefulWidget {
  @override
  _ChargingProgressIndicatorState createState() => 
      _ChargingProgressIndicatorState();
}

class _ChargingProgressIndicatorState 
    extends State<ChargingProgressIndicator>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: Duration(seconds: 3),
      vsync: this,
    );
    
    _animation = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: Curves.easeInOut,
      ),
    );
    
    _controller.forward();
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, child) {
        return CircularProgressIndicator(
          value: _animation.value,
        );
      },
    );
  }
}
```

### 序列動畫

一個接著一個執行。

```dart
class ChargingSequence extends StatefulWidget {
  @override
  _ChargingSequenceState createState() => _ChargingSequenceState();
}

class _ChargingSequenceState extends State<ChargingSequence>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _scaleAnimation;
  late Animation<double> _opacityAnimation;
  late Animation<double> _rotationAnimation;
  
  @override
  void initState() {
    super.initState();
    
    _controller = AnimationController(
      duration: Duration(seconds: 2),
      vsync: this,
    );
    
    // 0.0-0.3: 放大
    _scaleAnimation = Tween<double>(begin: 0.5, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: Interval(0.0, 0.3, curve: Curves.easeOut),
      ),
    );
    
    // 0.3-0.6: 淡入
    _opacityAnimation = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: Interval(0.3, 0.6, curve: Curves.easeIn),
      ),
    );
    
    // 0.6-1.0: 旋轉
    _rotationAnimation = Tween<double>(begin: 0.0, end: 1.0).animate(
      CurvedAnimation(
        parent: _controller,
        curve: Interval(0.6, 1.0, curve: Curves.easeInOut),
      ),
    );
    
    _controller.forward();
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return Transform.scale(
          scale: _scaleAnimation.value,
          child: Opacity(
            opacity: _opacityAnimation.value,
            child: Transform.rotate(
              angle: _rotationAnimation.value * 2 * 3.14159,
              child: Icon(Icons.ev_station, size: 50),
            ),
          ),
        );
      },
    );
  }
}
```

## Hero 動畫

頁面切換時的共享元素動畫。

```dart
// 列表頁
class StationListItem extends StatelessWidget {
  final Station station;
  
  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () {
        Navigator.push(
          context,
          MaterialPageRoute(
            builder: (context) => StationDetailPage(station: station),
          ),
        );
      },
      child: Hero(
        tag: 'station-${station.id}',
        child: Image.network(station.imageUrl),
      ),
    );
  }
}

// 詳情頁
class StationDetailPage extends StatelessWidget {
  final Station station;
  
  const StationDetailPage({required this.station});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          Hero(
            tag: 'station-${station.id}',
            child: Image.network(station.imageUrl),
          ),
          // 其他詳情
        ],
      ),
    );
  }
}
```

## CustomPainter 自訂繪製

完全自訂動畫。

### 充電進度環

```dart
class ChargingRingPainter extends CustomPainter {
  final double progress; // 0.0 - 1.0
  final Color color;
  
  ChargingRingPainter({required this.progress, required this.color});
  
  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = size.width / 2;
    
    // 背景圓環
    final bgPaint = Paint()
      ..color = Colors.grey[300]!
      ..style = PaintingStyle.stroke
      ..strokeWidth = 10
      ..strokeCap = StrokeCap.round;
    
    canvas.drawCircle(center, radius, bgPaint);
    
    // 進度圓環
    final progressPaint = Paint()
      ..color = color
      ..style = PaintingStyle.stroke
      ..strokeWidth = 10
      ..strokeCap = StrokeCap.round;
    
    final sweepAngle = 2 * 3.14159 * progress;
    
    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      -3.14159 / 2, // 從頂部開始
      sweepAngle,
      false,
      progressPaint,
    );
    
    // 中間文字
    final textPainter = TextPainter(
      text: TextSpan(
        text: '${(progress * 100).toInt()}%',
        style: TextStyle(
          color: Colors.black,
          fontSize: 24,
          fontWeight: FontWeight.bold,
        ),
      ),
      textDirection: TextDirection.ltr,
    );
    
    textPainter.layout();
    textPainter.paint(
      canvas,
      Offset(
        center.dx - textPainter.width / 2,
        center.dy - textPainter.height / 2,
      ),
    );
  }
  
  @override
  bool shouldRepaint(ChargingRingPainter oldDelegate) {
    return oldDelegate.progress != progress || oldDelegate.color != color;
  }
}

// 使用
class ChargingRingWidget extends StatefulWidget {
  @override
  _ChargingRingWidgetState createState() => _ChargingRingWidgetState();
}

class _ChargingRingWidgetState extends State<ChargingRingWidget>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: Duration(seconds: 5),
      vsync: this,
    )..repeat();
  }
  
  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _controller,
      builder: (context, child) {
        return CustomPaint(
          size: Size(200, 200),
          painter: ChargingRingPainter(
            progress: _controller.value,
            color: Colors.green,
          ),
        );
      },
    );
  }
}
```

### 波浪動畫

```dart
class WavePainter extends CustomPainter {
  final double animationValue;
  
  WavePainter({required this.animationValue});
  
  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()
      ..color = Colors.blue.withOpacity(0.5)
      ..style = PaintingStyle.fill;
    
    final path = Path();
    
    // 波浪起點
    path.moveTo(0, size.height * 0.7);
    
    // 繪製波浪
    for (double i = 0; i <= size.width; i++) {
      path.lineTo(
        i,
        size.height * 0.7 + 
            20 * sin((i / size.width * 2 * 3.14159) + (animationValue * 2 * 3.14159)),
      );
    }
    
    // 封閉路徑
    path.lineTo(size.width, size.height);
    path.lineTo(0, size.height);
    path.close();
    
    canvas.drawPath(path, paint);
  }
  
  @override
  bool shouldRepaint(WavePainter oldDelegate) {
    return oldDelegate.animationValue != animationValue;
  }
}
```

## 實務技巧

動畫時長建議：
- 簡單變化：200-300ms
- 複雜動畫：300-500ms
- 頁面切換：250-350ms

使用 `vsync` 節省資源，畫面不可見時自動暫停。

複雜動畫考慮用 Flare/Rive，設計師可直接產出。

避免在 `build` 裡建立 AnimationController，會造成記憶體洩漏。

測試動畫效果時用實機，模擬器可能不流暢。

動畫太多反而干擾使用者，適度就好。

下週研究 Flutter 與 Native 的深度整合：Platform Channels 和 FFI。
