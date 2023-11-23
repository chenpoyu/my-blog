---
layout: post
title: "Postmortem：從失敗中學習的藝術"
date: 2023-11-24 09:00:00 +0800
categories: [Observability, SRE]
tags: [Postmortem, Blameless, Incident Response, SRE]
---

「誰搞砸的？」

這是最糟糕的問題。

**Blameless Postmortem**（無咎責事後檢討）的核心理念是：

**我們要找出『系統為什麼失敗』，而不是『誰犯了錯』。**

## Postmortem 是什麼？

Postmortem 是一份**事後分析報告**，記錄了：
- 發生了什麼事
- 為什麼會發生
- 我們如何解決
- 如何防止再次發生

## 為什麼要做 Postmortem？

### 傳統思維：找出「罪魁禍首」

- 「是誰把錯誤的 config 部署到 Production？」
- 「是誰沒有測試就上線？」
- 結果：大家開始互相指責，沒有人願意承擔責任

### Blameless 思維：改善系統

- 「為什麼錯誤的 config 可以被部署到 Production？」
- 「為什麼我們的測試沒有抓到這個問題？」
- 結果：找出系統的弱點，並加以改善

## Postmortem 的結構

### 基本結構

```markdown
# [服務名稱] - [事件簡述]

## 1. 事件摘要（Executive Summary）
- 發生時間
- 影響範圍
- 根本原因
- 解決時間

## 2. 時間軸（Timeline）
- 詳細的事件發展過程

## 3. 根本原因分析（Root Cause Analysis）
- 為什麼會發生
- 5 Whys 分析

## 4. 影響評估（Impact Assessment）
- 使用者影響
- 商業影響
- SLO 影響

## 5. 解決步驟（Resolution）
- 如何解決問題

## 6. 行動項目（Action Items）
- 短期改善
- 長期改善

## 7. 學到的教訓（Lessons Learned）
- 做得好的地方
- 需要改進的地方
```

## 實戰：Database Connection Pool Exhaustion

```markdown
# Order Service - Database Connection Pool Exhaustion

**Date**: 2023-11-15
**Author**: Alice Chen
**Reviewers**: Bob Wang, Charlie Liu
**Severity**: P1

## 1. 事件摘要

**發生時間**: 2023-11-15 14:30 - 15:45 (UTC+8)
**持續時間**: 1 小時 15 分鐘
**影響範圍**: 所有使用者無法下單
**根本原因**: Database Connection Pool 設定過小，無法應付流量高峰
**使用者影響**: 約 5,000 筆訂單失敗

## 2. 時間軸

| 時間 | 事件 | 參與者 |
|------|------|--------|
| 14:30 | PagerDuty 發送告警：Order Service High Error Rate | - |
| 14:32 | Alice 開始調查 | Alice |
| 14:35 | 檢查 Grafana Dashboard，發現 Database Connection Pool 用盡 | Alice |
| 14:38 | 檢查 Kibana Logs，看到大量 `Connection timeout` 錯誤 | Alice |
| 14:40 | 檢查資料庫，CPU/Memory 正常 | Alice |
| 14:45 | 嘗試增加 Connection Pool Size，但需要重啟服務 | Alice |
| 14:50 | 通知團隊即將重啟服務 | Alice |
| 14:55 | 重啟 Order Service，Connection Pool Size 從 10 增加到 50 | Alice |
| 15:00 | 服務恢復正常，錯誤率下降 | Alice |
| 15:15 | 監控 30 分鐘，確認穩定 | Alice, Bob |
| 15:45 | 關閉事件 | Alice |

## 3. 根本原因分析

### 5 Whys

1. **為什麼 Order Service 出現 High Error Rate？**
   - 因為 Database Connection Pool 用盡

2. **為什麼 Database Connection Pool 會用盡？**
   - 因為 Connection Pool Size 設定為 10，但流量高峰時需要 50 個連線

3. **為什麼 Connection Pool Size 設定這麼小？**
   - 因為我們使用預設值，沒有根據實際流量調整

4. **為什麼我們沒有根據實際流量調整？**
   - 因為我們沒有監控 Connection Pool 使用率

5. **為什麼我們沒有監控 Connection Pool 使用率？**
   - 因為我們沒有在 Observability Checklist 中加入 Connection Pool 監控

### 根本原因

**系統性問題**：
- 缺乏 Connection Pool 使用率監控
- 預設 Connection Pool Size 沒有根據負載測試調整
- 沒有 Load Testing 流程

**流程問題**：
- 沒有在上線前進行負載測試
- Observability Checklist 不完整

## 4. 影響評估

### 使用者影響

- **訂單失敗數**: 約 5,000 筆
- **影響使用者數**: 約 5,000 人
- **客服投訴**: 約 200 通電話

### 商業影響

- **營收損失**: 約 $50,000 USD（假設平均訂單金額 $10）
- **品牌形象**: 社群媒體上有負面評論

### SLO 影響

- **可用性 SLO**: 99.9%
- **實際可用性**: 98.5%（低於 SLO）
- **Error Budget 消耗**: 100%（Error Budget 已用盡）

## 5. 解決步驟

### 立即處理（14:30 - 15:00）

1. 確認問題：Database Connection Pool 用盡
2. 增加 Connection Pool Size：10 → 50
3. 重啟服務
4. 監控恢復狀況

### 後續處理（15:00 - 18:00）

1. 分析失敗的 5,000 筆訂單
2. 建立 Compensation Script，自動重試失敗的訂單
3. 通知客服團隊，準備回應客戶

## 6. 行動項目

### 短期改善（1 週內完成）

- [ ] **[P0]** 加入 Connection Pool 監控到 Grafana Dashboard
  - Owner: Alice
  - Deadline: 2023-11-22
  - Metrics:
    - `hikaricp_connections_active`
    - `hikaricp_connections_idle`
    - `hikaricp_connections_max`

- [ ] **[P0]** 加入 Connection Pool 告警規則
  - Owner: Bob
  - Deadline: 2023-11-22
  - Alert:
    ```yaml
    - alert: DatabaseConnectionPoolNearlyFull
      expr: hikaricp_connections_active / hikaricp_connections_max > 0.8
      for: 5m
    ```

- [ ] **[P0]** 根據負載測試結果，調整所有服務的 Connection Pool Size
  - Owner: Charlie
  - Deadline: 2023-11-22

### 長期改善（1 個月內完成）

- [ ] **[P1]** 建立標準化的負載測試流程
  - Owner: Alice
  - Deadline: 2023-12-15
  - Tools: K6, Grafana

- [ ] **[P1]** 完善 Observability Checklist
  - Owner: Bob
  - Deadline: 2023-12-15
  - Items:
    - Database Connection Pool
    - Thread Pool
    - Memory Usage
    - GC Metrics

- [ ] **[P2]** 建立 Auto-scaling 機制
  - Owner: Charlie
  - Deadline: 2023-12-31
  - Based on: Connection Pool Usage

- [ ] **[P2]** 建立 Circuit Breaker
  - Owner: Alice
  - Deadline: 2023-12-31
  - Tool: Resilience4j

## 7. 學到的教訓

### 做得好的地方

✅ **快速回應**：從告警到開始調查，只花了 2 分鐘
✅ **明確的 Runbook**：依照 Runbook 快速定位問題
✅ **有效的溝通**：即時通知團隊和客服

### 需要改進的地方

❌ **缺乏預防性監控**：沒有監控 Connection Pool 使用率
❌ **預設值沒有調整**：使用預設的 Connection Pool Size
❌ **缺乏負載測試**：上線前沒有進行負載測試
❌ **SLO 違反**：Error Budget 已用盡，需要暫停上線

## 8. 相關資料

### Dashboards
- [Order Service Dashboard](http://grafana:3000/d/order-service)
- [Database Dashboard](http://grafana:3000/d/database)

### Logs
- [Kibana - Order Service Error Logs](http://kibana:5601/app/discover#/order-service-errors)

### Alerts
- [PagerDuty - Order Service High Error Rate](http://pagerduty.com/incidents/12345)

### Metrics
```promql
rate(http_requests_total{service="order-service",status="500"}[5m])
hikaricp_connections_active / hikaricp_connections_max
```

### Related Incidents
- [2023-10-01 - Similar Connection Pool Issue](http://wiki/postmortems/2023-10-01)
```

## Postmortem 會議

### 會議流程

1. **回顧時間軸**（10 分鐘）
   - 主持人帶領大家回顧事件發展

2. **根本原因分析**（15 分鐘）
   - 使用 5 Whys 找出根本原因

3. **行動項目討論**（20 分鐘）
   - 決定短期和長期改善措施
   - 指定 Owner 和 Deadline

4. **總結**（5 分鐘）
   - 確認所有行動項目
   - 安排下次檢討會議

### 參與者

- **必須參與**：
  - 事件處理人員
  - 相關服務的 Owner
  - SRE Lead

- **選擇性參與**：
  - 產品經理
  - 其他團隊成員

## 自動化 Postmortem

### Postmortem Template Generator
{% raw %}
```python
# postmortem_generator.py
from datetime import datetime
from jinja2 import Template

template = Template("""
# {{ service }} - {{ title }}

**Date**: {{ date }}
**Author**: {{ author }}
**Severity**: {{ severity }}

## 1. 事件摘要

**發生時間**: {{ start_time }} - {{ end_time }}
**持續時間**: {{ duration }}
**影響範圍**: {{ impact }}
**根本原因**: {{ root_cause }}

## 2. 時間軸

{% for event in timeline %}
| {{ event.time }} | {{ event.description }} | {{ event.person }} |
{% endfor %}

## 3. 根本原因分析

### 5 Whys

{% for i, why in enumerate(five_whys, 1) %}
{{ i }}. **{{ why.question }}**
   - {{ why.answer }}
{% endfor %}

## 4. 影響評估

{{ impact_details }}

## 5. 解決步驟

{{ resolution }}

## 6. 行動項目

### 短期改善

{% for item in action_items.short_term %}
- [ ] **[{{ item.priority }}]** {{ item.description }}
  - Owner: {{ item.owner }}
  - Deadline: {{ item.deadline }}
{% endfor %}

### 長期改善

{% for item in action_items.long_term %}
- [ ] **[{{ item.priority }}]** {{ item.description }}
  - Owner: {{ item.owner }}
  - Deadline: {{ item.deadline }}
{% endfor %}

## 7. 學到的教訓

### 做得好的地方

{% for lesson in lessons.good %}
✅ {{ lesson }}
{% endfor %}

### 需要改進的地方

{% for lesson in lessons.bad %}
❌ {{ lesson }}
{% endfor %}
""")

def generate_postmortem(data):
    return template.render(data)

# 使用範例
data = {
    'service': 'Order Service',
    'title': 'Database Connection Pool Exhaustion',
    'date': '2023-11-15',
    'author': 'Alice Chen',
    'severity': 'P1',
    'start_time': '2023-11-15 14:30',
    'end_time': '2023-11-15 15:45',
    'duration': '1 hour 15 minutes',
    'impact': '所有使用者無法下單',
    'root_cause': 'Database Connection Pool 設定過小',
    'timeline': [
        {'time': '14:30', 'description': 'PagerDuty alert', 'person': '-'},
        {'time': '14:32', 'description': 'Started investigation', 'person': 'Alice'},
    ],
    'five_whys': [
        {'question': 'Why High Error Rate?', 'answer': 'Connection Pool exhausted'},
    ],
    'action_items': {
        'short_term': [
            {'priority': 'P0', 'description': 'Add Connection Pool monitoring', 'owner': 'Alice', 'deadline': '2023-11-22'},
        ],
        'long_term': [
            {'priority': 'P1', 'description': 'Create Load Testing process', 'owner': 'Bob', 'deadline': '2023-12-15'},
        ]
    },
    'lessons': {
        'good': ['Quick response', 'Clear Runbook'],
        'bad': ['No proactive monitoring', 'No load testing']
    }
}

print(generate_postmortem(data))
```
{% endraw %}

## 實戰建議

### 1. 定期檢討 Postmortem

每個月檢討所有 Postmortem，確保行動項目有被執行。

### 2. 建立 Postmortem 資料庫

把所有 Postmortem 放在一個地方，方便搜尋和學習。

### 3. 分享 Postmortem

定期分享 Postmortem 給全公司，讓大家都能從失敗中學習。

### 4. 獎勵分享失敗

建立鼓勵分享失敗的文化，而不是懲罰失敗。

---

**Postmortem 不是為了找出「誰犯了錯」，而是為了找出「系統為什麼失敗」。**

當你建立了 Blameless 文化，團隊才會願意分享失敗，並從中學習。
