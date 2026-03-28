# test.py 程式碼說明

## 概述

這是一個 Open WebUI 的 **Filter（過濾器）插件**，主要功能是限制使用者與 AI 的對話輪數，防止對話過長。

---

## 結構說明

### 1. `Valves`（系統設定）

```python
class Valves(BaseModel):
    priority: int = Field(default=0, ...)
    max_turns: int = Field(default=8, ...)
```

- 由管理員設定的全域參數
- `priority`：此過濾器的執行優先順序，預設為 0
- `max_turns`：整個系統允許的最大對話輪數，預設為 **8 輪**

---

### 2. `UserValves`（使用者設定）

```python
class UserValves(BaseModel):
    max_turns: int = Field(default=4, ...)
```

- 由個別使用者自訂的參數
- `max_turns`：該使用者允許的最大對話輪數，預設為 **4 輪**

---

### 3. `__init__`（初始化）

```python
def __init__(self):
    self.valves = self.Valves()
```

- 初始化時載入系統設定（`Valves`）
- 備註中提到可啟用 `self.file_handler = True` 來自訂檔案處理邏輯

---

### 4. `inlet`（請求前處理）

```python
def inlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
```

- 在請求送到 AI API **之前**執行
- 取得目前對話的訊息數量
- 計算有效的最大輪數：取使用者設定與系統設定的**較小值**
  ```python
  max_turns = min(__user__["valves"].max_turns, self.valves.max_turns)
  ```
- 若對話輪數超過限制，則拋出例外錯誤，阻止請求繼續
- 適用對象：角色為 `user` 或 `admin` 的使用者

---

### 5. `outlet`（回應後處理）

```python
def outlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
```

- 在 AI API **回應之後**執行
- 目前僅印出回應內容，直接回傳，未做額外處理
- 可在此加入回應分析或修改邏輯

---

## 運作流程

```
使用者發送訊息
      ↓
  inlet() 檢查對話輪數
      ↓
  超過限制？ → 是 → 拋出錯誤，停止請求
      ↓ 否
  送到 AI API 處理
      ↓
  outlet() 處理回應
      ↓
  回傳給使用者
```

---

## 重點邏輯

最大輪數取「使用者設定」與「系統設定」的**最小值**，確保使用者無法自行調高超過系統上限。

| 設定來源 | 預設值 |
|----------|--------|
| 系統（Valves）| 8 輪 |
| 使用者（UserValves）| 4 輪 |
| 實際生效 | min(4, 8) = **4 輪** |
