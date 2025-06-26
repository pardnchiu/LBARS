- 1 分鐘健康檢查 / 10 分鐘故障恢復 / 5 分鐘 MaxConn 動態調整
- 請求時間 + Ping延遲 + 響應速度 + LLM 延遲加權計算
- 連線池
- 記憶體預分配
- 動態調整 MaxConn
- 故障/超時立即降級，0 延遲重試下個後端
- proxypass 處理請求，timer 處理健康檢查
- 提供 `healthPath` / `llmCheckPath` 參數用於提供自訂路徑檢查健康與 LLM 響應速度
- `llmCheckPath`: 最小 token 測量 API 延遲
- 故障 Email 通知

```mermaid
graph TD
 A[請求] --> B{健康列表是否為空}
 B -->|是| B1[回傳 503]
 B -->|否| B2[選擇後端]
 
 B2 --> C{檢查 MaxConn}
 C -->|Conn < MaxConn<br>conn++| C1{15秒內收到響應?}
 C -->|Conn >= MaxConn<br>下一個| B2
 
 C1 -->|成功<br/>conn--| C11[轉發至客戶端]
 C1 -->|超時/錯誤<br/>立即降級| C12[移至故障列表]
 
 C12 --> D{健康列表為空?}
 D -->|否<br>下一個| B2
 D -->|是| Z
 
 C11 --> Z[請求完成]
 
 JJ -.-> B2
 NN -.-> B2
 
 subgraph "背景健康檢查系統"
 DD[健康檢查<br/>1分鐘]
 EE[故障檢查<br/>10分鐘]
 GG[動態計分<br/>5分鐘]
 
 DD -->|併發| HH{健康狀態<br>若有 llmCheckPath 則也必須有回應}
 HH -->|正常| II[維持健康]
 HH -->|故障| JJ[移至故障列表]
 
 EE -->|併發| LL{健康狀態<br>若有 llmCheckPath 則也必須有回應}
 LL -->|正常| NN[移回健康列表]
 LL -->|故障| OO[保持故障狀態]
 
 GG --> PP{效能評分<br>可設置最高最低}
 PP -->|快| QQ[MaxConn +10]
 PP -->|慢| RR[MaxConn -5]
 end
```
