# 章節8：DeFi 核心協議實作

> **注意**：本章節的原始筆記仍在規劃中，以下為根據 DeFi 核心概念整理的架構內容。

## 目錄
- [8.1 AMM 定價公式](#81-amm-定價公式)
  - [AMM 定價模型原理](#amm-定價模型原理)
  - [交易滑點與套利機制](#交易滑點與套利機制)
  - [常見 AMM 類型](#常見-amm-類型)
- [8.2 流動性與 Impermanent Loss](#82-流動性與-impermanent-loss)
- [8.3 去中心化借貸](#83-去中心化借貸)
- [8.4 去中心化衍生品](#84-去中心化衍生品)
- [8.5 Oracle 解決方案](#85-oracle-解決方案)

---

## 8.1 AMM 定價公式

### AMM 定價模型原理

AMM (Automated Market Maker) 是 DeFi 中取代傳統訂單簿的核心創新，通過數學公式自動維護流動性。

#### 恆定乘積公式 (Uniswap V2)

```
x × y = k

x = Token A 儲備量
y = Token B 儲備量
k = 常數（交易前後保持不變）
```

**價格計算**
```python
class ConstantProductAMM:
    def __init__(self, reserve_x: int, reserve_y: int):
        self.reserve_x = reserve_x  # Token A 儲量
        self.reserve_y = reserve_y  # Token B 儲量
        self.k = reserve_x * reserve_y
    
    def spot_price(self) -> float:
        """即時價格 = y / x（不含手續費）"""
        return self.reserve_y / self.reserve_x
    
    def get_amount_out(self, amount_in: int, zero_for_one: bool, fee_rate: float = 0.003) -> int:
        """
        計算輸入 amount_in 個 Token A 可換出多少 Token B
        Uniswap V2 公式（含 0.3% 手續費）
        """
        amount_in_with_fee = int(amount_in * (1 - fee_rate))
        
        if zero_for_one:  # A → B
            # amount_out = reserve_y × amount_in_with_fee / (reserve_x + amount_in_with_fee)
            amount_out = self.reserve_y * amount_in_with_fee // (self.reserve_x + amount_in_with_fee)
        else:  # B → A
            amount_out = self.reserve_x * amount_in_with_fee // (self.reserve_y + amount_in_with_fee)
        
        return amount_out
    
    def execute_swap(self, amount_in: int, zero_for_one: bool, fee_rate: float = 0.003) -> int:
        """執行交換並更新儲量"""
        amount_out = self.get_amount_out(amount_in, zero_for_one, fee_rate)
        
        if zero_for_one:
            self.reserve_x += amount_in
            self.reserve_y -= amount_out
        else:
            self.reserve_y += amount_in
            self.reserve_x -= amount_out
        
        # 驗證恆定乘積（允許手續費導致的微小增長）
        assert self.reserve_x * self.reserve_y >= self.k
        self.k = self.reserve_x * self.reserve_y
        
        return amount_out
```

---

### 交易滑點與套利機制

#### 滑點計算

滑點 = 實際成交價格與預期價格的偏差，主要由交易量相對於池子大小決定。

```python
def calculate_slippage(reserve_x: int, reserve_y: int, amount_in: int) -> float:
    """計算交易滑點"""
    spot_price = reserve_y / reserve_x
    
    # 實際成交後的平均價格
    amount_out = reserve_y * amount_in // (reserve_x + amount_in)
    actual_avg_price = amount_out / amount_in
    
    slippage = (spot_price - actual_avg_price) / spot_price
    return slippage

# 示例：100 ETH 池子，換 1 ETH
print(calculate_slippage(100, 100000, 1))    # ~1% 滑點
print(calculate_slippage(100, 100000, 10))   # ~9% 滑點
print(calculate_slippage(100, 100000, 50))   # ~33% 滑點
```

#### 套利機制

套利者通過不同交易所間的價格差異賺取利潤，同時幫助價格趨向均衡：

```python
class ArbitrageBot:
    def __init__(self, pool_a: ConstantProductAMM, pool_b: ConstantProductAMM):
        self.pool_a = pool_a
        self.pool_b = pool_b
    
    def find_arbitrage(self, token_amount: int) -> dict:
        """尋找套利機會"""
        price_a = self.pool_a.spot_price()
        price_b = self.pool_b.spot_price()
        
        if price_a < price_b:
            # 在 A 買，在 B 賣
            buy_pool, sell_pool = self.pool_a, self.pool_b
        else:
            buy_pool, sell_pool = self.pool_b, self.pool_a
        
        # 計算套利收益
        tokens_received = buy_pool.get_amount_out(token_amount, True)
        profit_in_eth = sell_pool.get_amount_out(tokens_received, False)
        net_profit = profit_in_eth - token_amount
        
        return {
            'profitable': net_profit > 0,
            'net_profit': net_profit,
            'price_a': price_a,
            'price_b': price_b
        }
```

---

### 常見 AMM 類型

#### Uniswap V3：集中流動性

```python
class ConcentratedLiquidityAMM:
    """
    Uniswap V3：LP 可以在指定價格區間 [price_lower, price_upper] 內提供流動性
    提高資本效率（相比 V2 提高 100-4000 倍）
    """
    
    def __init__(self, price_lower: float, price_upper: float, liquidity: float):
        self.sqrt_price_lower = price_lower ** 0.5
        self.sqrt_price_upper = price_upper ** 0.5
        self.liquidity = liquidity  # 流動性 L
    
    def get_amount_x(self, current_sqrt_price: float) -> float:
        """計算區間內的 Token X 數量"""
        if current_sqrt_price >= self.sqrt_price_upper:
            return 0
        sqrt_p = min(current_sqrt_price, self.sqrt_price_upper)
        return self.liquidity * (self.sqrt_price_upper - sqrt_p) / (sqrt_p * self.sqrt_price_upper)
    
    def get_amount_y(self, current_sqrt_price: float) -> float:
        """計算區間內的 Token Y 數量"""
        if current_sqrt_price <= self.sqrt_price_lower:
            return 0
        sqrt_p = min(current_sqrt_price, self.sqrt_price_upper)
        return self.liquidity * (sqrt_p - self.sqrt_price_lower)
```

#### 各 AMM 類型比較

| AMM 類型 | 代表協議 | 公式 | 適用場景 |
|---------|---------|------|---------|
| 恆定乘積 | Uniswap V2 | x·y=k | 一般代幣對 |
| 集中流動性 | Uniswap V3 | 區間流動性 | 穩定幣對、主流資產 |
| 穩定幣特化 | Curve | StableSwap | 穩定幣/同質資產 |
| 多資產池 | Balancer | x₁^w₁·x₂^w₂...=k | 指數型投資組合 |

> **🔄 技術更新（2025+）**：
> - **Uniswap V4（2024年底）**：引入「Hooks」機制，允許開發者在流動性操作的各階段（swap 前後、LP 存取前後）插入自定義邏輯，可實現動態費率、鏈上限價單、MEV 保護等功能；同時採用 Singleton 合約架構（所有池共用一個合約），大幅降低 Gas。
> - **Intent-based 交換（UniswapX、CoW Protocol）**：用戶簽署「意圖」（我想用 X 換至少 Y 個 Z），由求解者（Solver）競爭最優路徑，解決 MEV 搶跑和滑點問題，成為新一代交換架構主流。

---

## 8.2 流動性與 Impermanent Loss

#### 無常損失原理

```python
def impermanent_loss(price_ratio_change: float) -> float:
    """
    計算無常損失
    price_ratio_change: 相對價格變化倍數（例如 2.0 = 漲 2 倍，0.5 = 跌 50%）
    
    公式：IL = 2√P/(1+P) - 1，其中 P = price_ratio_change
    """
    import math
    P = price_ratio_change
    hold_value = (1 + P) / 2  # 如果不提供流動性，持有的價值
    pool_value = math.sqrt(P)  # 提供流動性後的相對價值
    return pool_value / hold_value - 1

# 示例
print(f"價格漲 2 倍：IL = {impermanent_loss(2):.2%}")   # -5.72%
print(f"價格漲 5 倍：IL = {impermanent_loss(5):.2%}")   # -25.46%
print(f"價格漲 10 倍：IL = {impermanent_loss(10):.2%}") # -42.46%
```

**關鍵洞見**：
- 無常損失只在提取流動性時才「實現」
- 手續費收入需要超過無常損失才有利可圖
- 穩定幣對的無常損失接近 0（價格波動極小）

---

## 8.3 去中心化借貸

#### Aave/Compound 借貸模型

```python
class LendingPool:
    """去中心化借貸池（AAVE/Compound 風格）"""
    
    def __init__(self, asset: str, optimal_utilization: float = 0.8):
        self.asset = asset
        self.total_deposited = 0
        self.total_borrowed = 0
        self.U_optimal = optimal_utilization  # 最優利用率
        
        # 利率模型參數
        self.base_rate = 0.02        # 2% 基礎利率
        self.slope1 = 0.04           # 低利用率斜率
        self.slope2 = 0.75           # 高利用率斜率（U > U_optimal 時急劇上升）
    
    @property
    def utilization_rate(self) -> float:
        if self.total_deposited == 0:
            return 0
        return self.total_borrowed / self.total_deposited
    
    def borrow_rate(self) -> float:
        """利率曲線：利用率越高，借貸成本越高"""
        U = self.utilization_rate
        if U <= self.U_optimal:
            return self.base_rate + (U / self.U_optimal) * self.slope1
        else:
            excess = (U - self.U_optimal) / (1 - self.U_optimal)
            return self.base_rate + self.slope1 + excess * self.slope2
    
    def deposit_rate(self) -> float:
        """存款利率 = 借貸利率 × 利用率 × (1 - 協議費用)"""
        protocol_fee = 0.1  # 10% 協議抽成
        return self.borrow_rate() * self.utilization_rate * (1 - protocol_fee)
    
    def calculate_health_factor(self, collateral_value: float, debt_value: float, 
                                 liquidation_threshold: float = 0.8) -> float:
        """
        健康因子：>1 安全，<1 可被清算
        HF = (抵押品 × 清算閾值) / 債務
        """
        return (collateral_value * liquidation_threshold) / debt_value if debt_value > 0 else float('inf')
    
    def is_liquidatable(self, collateral_value: float, debt_value: float) -> bool:
        return self.calculate_health_factor(collateral_value, debt_value) < 1.0
```

---

## 8.4 去中心化衍生品

#### 去中心化永續合約

```python
class PerpetualContract:
    """去中心化永續合約（GMX/dYdX 風格）"""
    
    def __init__(self, underlying: str, max_leverage: int = 30):
        self.underlying = underlying
        self.max_leverage = max_leverage
        self.open_interest_long = 0
        self.open_interest_short = 0
    
    def funding_rate(self) -> float:
        """
        資金費率：平衡多空倉位
        多頭超多 → 多頭付費給空頭（促使更多空頭開倉）
        """
        oi_long = self.open_interest_long
        oi_short = self.open_interest_short
        
        if oi_long == 0 and oi_short == 0:
            return 0
        
        imbalance = (oi_long - oi_short) / (oi_long + oi_short)
        base_funding_rate = 0.0001  # 0.01% per 8 hours
        
        return imbalance * base_funding_rate
    
    def open_position(self, user: str, side: str, collateral: float, leverage: float):
        """開倉"""
        assert leverage <= self.max_leverage, f"槓桿超過上限 {self.max_leverage}x"
        assert side in ['long', 'short']
        
        size = collateral * leverage
        
        if side == 'long':
            self.open_interest_long += size
        else:
            self.open_interest_short += size
        
        return {
            'position_size': size,
            'collateral': collateral,
            'leverage': leverage,
            'liquidation_price': self._calculate_liq_price(side, collateral, size)
        }
    
    def _calculate_liq_price(self, side, collateral, size, maintenance_margin=0.005):
        """計算清算價格"""
        entry_price = self.get_mark_price()
        if side == 'long':
            return entry_price * (1 - collateral/size + maintenance_margin)
        else:
            return entry_price * (1 + collateral/size - maintenance_margin)
```

---

## 8.5 Oracle 解決方案

#### 去中心化預言機

```python
class ChainlinkOracle:
    """Chainlink 去中心化預言機模型"""
    
    def __init__(self, aggregator_address: str):
        self.aggregator = aggregator_address
        self.heartbeat = 3600  # 最大更新間隔（秒）
    
    def get_latest_price(self) -> dict:
        """獲取最新價格"""
        round_data = self._query_aggregator()
        
        # 安全檢查
        assert round_data['answeredInRound'] >= round_data['roundId'], "過期的回合資料"
        assert time.time() - round_data['updatedAt'] < self.heartbeat, "價格過期"
        assert round_data['answer'] > 0, "無效價格"
        
        return {
            'price': round_data['answer'],
            'decimals': 8,  # Chainlink 使用 8 位小數
            'updated_at': round_data['updatedAt'],
            'round_id': round_data['roundId']
        }

class TWAPOracle:
    """TWAP (時間加權平均價格) 預言機 - Uniswap V3 內建"""
    
    def __init__(self, pool, observation_period: int = 1800):
        self.pool = pool
        self.period = observation_period  # 30 分鐘
    
    def get_twap(self) -> float:
        """計算過去 30 分鐘的 TWAP"""
        current_tick_cumulative = self.pool.observe(0)
        past_tick_cumulative = self.pool.observe(self.period)
        
        tick_avg = (current_tick_cumulative - past_tick_cumulative) / self.period
        return 1.0001 ** tick_avg  # 將 tick 轉換為價格

# Oracle 安全問題
class OracleManipulationDefense:
    """防範預言機操控"""
    
    def __init__(self, primary_oracle, backup_oracle, max_deviation: float = 0.05):
        self.primary = primary_oracle
        self.backup = backup_oracle
        self.max_deviation = max_deviation  # 最大允許偏差 5%
    
    def get_safe_price(self) -> float:
        price1 = self.primary.get_latest_price()['price']
        price2 = self.backup.get_latest_price()['price']
        
        deviation = abs(price1 - price2) / price1
        if deviation > self.max_deviation:
            raise OracleManipulationError(
                f"價格偏差過大：{deviation:.2%}，可能存在操控"
            )
        
        return (price1 + price2) / 2  # 取平均值
```

#### 主要 Oracle 解決方案比較

| Oracle | 類型 | 去中心化程度 | 更新頻率 | 安全性 |
|--------|------|-----------|---------|-------|
| Chainlink | 聚合器網絡 | 高 | 每小時或偏差 | 高 |
| Uniswap TWAP | 鏈上 AMM | 最高 | 即時 | 中（可操控） |
| Band Protocol | 去中心化 | 高 | 按需 | 高 |
| Pyth Network | 機構數據 | 中 | 亞秒級 | 中-高 |

> **🔄 技術更新（2025+）**：
> - **Pyth Network 崛起**：Pyth 的 Pull Oracle 模式（由 dApp 在需要時主動拉取並推送價格，而非主動推送到所有鏈）已成為 Solana 和多鏈 DeFi 主流，延遲達毫秒級，已整合超過 90 條鏈。
> - **EigenLayer AVS Oracle**：透過再質押（Restaking）為 Oracle 提供經濟安全性，驗證者的 ETH 質押也保護 Oracle 的誠實性，是下一代去信任 Oracle 架構。
> - **MEV（最大可提取價值）**：2024-2025 年 MEV 機器人（三明治攻擊、套利）從 Ethereum 主網提取超 10 億美元。Flashbots SUAVE（Single Unifying Auction for Value Expression）嘗試將 MEV 機制標準化並對用戶友好，是 DeFi 基礎設施的重要議題。
> - **EigenLayer / 再質押（Restaking）**：允許 ETH 質押者將相同 ETH 同時為多個「主動驗證服務（AVS）」提供安全保障，LRTs（Liquid Restaking Tokens，如 ezETH、rsETH）衍生出新型 DeFi 生態，到 2025 年 TVL 超 100 億美元。
