# 章節13：Token 標準與穩定幣

> 代幣標準是鏈上資產的「通用介面」——正因為 ERC-20 統一了介面，任何錢包、交易所、DeFi 協議才能無縫支援成千上萬種代幣。本章涵蓋同質化（ERC-20）、非同質化（ERC-721）、多代幣（ERC-1155）標準，以及建立在其上的穩定幣機制與脫鉤風險。合約範例以 Solidity 表達介面，Python 模擬機制。

## 目錄
- [13.1 ERC-20 同質化代幣](#131-erc-20-同質化代幣)
- [13.2 ERC-721 與 NFT](#132-erc-721-與-nft)
- [13.3 ERC-1155 與延伸標準](#133-erc-1155-與延伸標準)
- [13.4 穩定幣機制設計](#134-穩定幣機制設計)
- [13.5 脫鉤風險與案例分析](#135-脫鉤風險與案例分析)

---

## 13.1 ERC-20 同質化代幣

ERC-20 是最基礎、最廣泛的代幣標準。「同質化（Fungible）」指每一單位完全等價可互換（如 1 USDC = 任何 1 USDC）。標準只規定一組函式介面，代幣的餘額其實就是合約裡的一個 `mapping`。

```solidity
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function approve(address spender, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

**approve / transferFrom 授權流程**——讓合約（如 DEX）能代你動用代幣：

```
使用者 ──approve(DEX, 100)──▶ 代幣合約設定 allowance[使用者][DEX] = 100
使用者 ──呼叫 swap()──▶ DEX ──transferFrom(使用者, DEX, 100)──▶ 代幣合約
                                       檢查並扣減 allowance
```

```python
class ERC20:
    def __init__(self, supply):
        self.balances = {"deployer": supply}
        self.allowance = {}                       # (owner, spender) -> amount
        self.total_supply = supply

    def transfer(self, sender, to, amount):
        assert self.balances.get(sender, 0) >= amount
        self.balances[sender] -= amount
        self.balances[to] = self.balances.get(to, 0) + amount
        return True

    def approve(self, owner, spender, amount):
        self.allowance[(owner, spender)] = amount
        return True

    def transfer_from(self, spender, owner, to, amount):
        assert self.allowance.get((owner, spender), 0) >= amount
        assert self.balances.get(owner, 0) >= amount
        self.allowance[(owner, spender)] -= amount
        self.balances[owner] -= amount
        self.balances[to] = self.balances.get(to, 0) + amount
        return True
```

**approve race condition（授權競態）**：若把 allowance 從 100 改成 50，spender 可能在你的變更交易上鏈前，搶先用掉舊的 100、再用新的 50，共花 150。防禦：先 `approve(0)` 再設新值，或用 `increaseAllowance/decreaseAllowance`。**ERC-2612 permit** 進一步用簽章（EIP-712）離線授權，省去單獨的 approve 交易與其 gas。

---

## 13.2 ERC-721 與 NFT

ERC-721 定義**非同質化代幣（NFT）**：每個 token 有獨一無二的 `tokenId`，彼此不可互換（如兩張不同的藝術品）。核心差異是所有權從「數量」變成「誰擁有哪個 id」：

```solidity
interface IERC721 {
    function ownerOf(uint256 tokenId) external view returns (address);
    function balanceOf(address owner) external view returns (uint256);
    function transferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;         // 授權單一 token
    function setApprovalForAll(address op, bool approved) external; // 授權全部
    function tokenURI(uint256 tokenId) external view returns (string); // metadata
}
```

- **metadata / tokenURI**：`tokenURI` 通常回傳一個指向 JSON 的 URL（含名稱、圖片、屬性）。圖片與 metadata 常存於 IPFS/Arweave 而非鏈上（鏈上存全圖太貴）——這也帶來「連結失效則 NFT 內容消失」的風險。
- **enumerable（ERC-721Enumerable）**：可選延伸，讓合約能列舉「某地址擁有的所有 token」與「全部 token」，方便前端展示，但增加寫入 gas。
- **royalty（ERC-2981）**：標準化「版稅」查詢介面，讓市場在二次交易時知道該付多少給創作者（是否執行由市場決定，非強制）。

```python
class ERC721:
    def __init__(self):
        self.owner_of = {}          # tokenId -> owner
        self.balance = {}

    def mint(self, to, token_id):
        assert token_id not in self.owner_of
        self.owner_of[token_id] = to
        self.balance[to] = self.balance.get(to, 0) + 1

    def transfer(self, sender, to, token_id):
        assert self.owner_of.get(token_id) == sender
        self.owner_of[token_id] = to
        self.balance[sender] -= 1
        self.balance[to] = self.balance.get(to, 0) + 1
```

---

## 13.3 ERC-1155 與延伸標準

**ERC-1155（多代幣標準）** 讓單一合約同時管理多種代幣——可同質、可非同質，並支援**批次操作**（一筆交易轉多種 token），大幅省 gas。遊戲道具（大量同質的「金幣」+ 少量非同質的「傳奇武器」）是典型場景。

```solidity
// 一份合約管理多個 id；balanceOf 需指定 (owner, id)
function balanceOf(address account, uint256 id) external view returns (uint256);
function safeBatchTransferFrom(
    address from, address to,
    uint256[] calldata ids, uint256[] calldata amounts, bytes calldata data
) external;   // 批次轉帳：一次搬多種道具
```

| 標準 | 同質性 | 一份合約的代幣種類 | 批次 | 典型用途 |
|------|--------|------------------|------|---------|
| ERC-20 | 同質 | 單一 | 否 | 貨幣、治理代幣 |
| ERC-721 | 非同質 | 單一系列 | 否 | 藝術品、收藏、身分 |
| ERC-1155 | 混合 | 多種 | 是 | 遊戲道具、多資產 |

**其他重要延伸**：

- **ERC-4626（金庫標準）**：把「存入資產、拿到代表份額的憑證代幣」的收益金庫介面標準化，讓 DeFi 收益協議可組合（回鏈 [章節08 DeFi](章節08_DeFi核心協議實作.md)）。
- **ERC-2612（permit）**：ERC-20 的簽章授權延伸（見 13.1）。
- **SBT（Soulbound Token，靈魂綁定代幣）**：不可轉讓的 NFT，用於身分、憑證、聲譽等「不該被買賣」的場景。

---

## 13.4 穩定幣機制設計

穩定幣旨在維持與某資產（通常 1 美元）的固定匯率，是鏈上金融的「記帳單位」與避風港。依維持錨定的方式分三類：

| 類型 | 錨定機制 | 代表 | 資本效率 | 主要風險 |
|------|---------|------|---------|---------|
| 法幣抵押 | 鏈下 1:1 儲備（美元/國債） | USDT、USDC | 高（1:1） | 中心化、儲備透明度、託管銀行 |
| 加密超額抵押 | 鏈上超額抵押 + 清算 | DAI | 低（>1:1） | 抵押品暴跌、清算失靈 |
| 演算法式 | 靠套利/鑄燒調節供給 | （UST，已崩） | 最高 | 死亡螺旋、信心崩潰 |

**加密超額抵押（DAI 型）機制與清算**——用 Python 模擬倉位健康度與清算：

```python
class CDP:
    """抵押債倉：抵押 ETH，鑄出穩定幣 DAI。"""
    LIQUIDATION_RATIO = 1.5          # 抵押率須 ≥150%
    PENALTY = 0.13                   # 清算罰金 13%

    def __init__(self, collateral_eth, eth_price, debt_dai):
        self.collateral = collateral_eth
        self.debt = debt_dai
        self.eth_price = eth_price

    def collateral_ratio(self):
        return (self.collateral * self.eth_price) / self.debt

    def is_safe(self):
        return self.collateral_ratio() >= self.LIQUIDATION_RATIO

    def liquidate(self):
        """跌破門檻 → 拍賣抵押品償債 + 罰金，維持系統償付能力。"""
        if self.is_safe():
            return "健康，無需清算"
        repay_value = self.debt * (1 + self.PENALTY)
        seized_eth = repay_value / self.eth_price
        return f"清算：沒收 {seized_eth:.3f} ETH 償還 {self.debt} DAI + 罰金"

cdp = CDP(collateral_eth=1.0, eth_price=3000, debt_dai=2500)  # 抵押率 120%
print(cdp.collateral_ratio())   # 1.2 → 低於 1.5
print(cdp.liquidate())          # 觸發清算
```

**維持錨定的關鍵**是套利與清算的良性循環：DAI < $1 時，借款人可折價買回 DAI 還債獲利（推升價格）；DAI > $1 時，可鑄新 DAI 賣出獲利（壓低價格）。超額抵押確保即使抵押品下跌，清算仍能覆蓋債務，系統保持償付能力。

---

## 13.5 脫鉤風險與案例分析

穩定幣「穩定」是設計目標，不是保證。歷史上三類穩定幣都曾脫鉤（depeg）。

**演算法穩定幣的死亡螺旋——UST/Luna（2022，蒸發約 $400 億）**：

```
UST 靠「1 UST 永遠可鑄/燒換 $1 的 Luna」維持錨定（供給彈性，無外部抵押）

正常：UST < $1 → 套利者燒 UST 鑄 Luna 獲利 → UST 供給減少 → 回到 $1

崩潰螺旋（信心一旦動搖）：
  大量拋售 UST → UST 脫鉤跌破 $1
        │
        ▼
  套利者燒 UST 換 Luna → Luna 供給暴增、價格暴跌
        │
        ▼
  Luna 崩跌 → 「1 UST 可換 $1 Luna」的承諾失去價值背書
        │
        ▼
  更多人恐慌拋售 UST ──────┐ 正回饋
        ▲                   │
        └───────────────────┘  數日內雙雙歸零
```

演算法穩定幣的根本弱點：錨定完全建立在「對治理代幣價值的信心」上，沒有外部硬資產兜底，信心崩潰即自我強化地崩盤。

**法幣抵押的儲備風險——USDC（2023，SVB 事件）**：USDC 短暫脫鉤至約 $0.87，原因不是機制缺陷，而是其發行方 Circle 有約 $33 億儲備存於倒閉的 Silicon Valley Bank，市場擔憂儲備無法足額兌付。存款獲全額擔保後，USDC 數日內回錨。教訓：法幣抵押穩定幣的風險轉移到了**鏈下託管與儲備品質**。

**Peg 防禦機制**：

- **PSM（Peg Stability Module）**：允許以 1:1 用其他穩定幣（如 USDC）直接鑄換 DAI，提供硬套利邊界，強力收斂價格（代價是引入對 USDC 的依賴）。
- **超額抵押緩衝 + 清算**：確保任何時刻抵押品價值 > 流通穩定幣，維持償付能力。
- **套利迴路**：健康的鑄/燒或清算誘因，讓偏離錨定時有人有利可圖去修正它。

> **🔄 技術更新（2025+）**：**RWA（真實世界資產）代幣化**成為主流敘事——代幣化美國國債（如 BlackRock BUIDL）為鏈上帶來有收益的「準穩定資產」，模糊了穩定幣與貨幣市場基金的界線。監管同步落地：歐盟 **MiCA** 對穩定幣發行與儲備設下嚴格要求，美國 **GENIUS Act（2025）** 建立聯邦穩定幣框架，要求 1:1 高品質流動儲備與定期揭露。合規、有儲備背書的穩定幣正取代演算法式與不透明發行方，成為機構採用的主軸。
