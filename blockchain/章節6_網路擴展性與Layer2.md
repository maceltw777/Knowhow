# 章節6：網路擴展性與 Layer 2

## 目錄
- [6.2 Rollups 架構](#62-rollups-架構)
  - [Optimistic Rollups 架構設計](#optimistic-rollups-架構設計)
  - [零知識證明 Rollups 技術](#零知識證明-rollups-技術)
- [6.3 分片設計](#63-分片設計)
  - [分片技術與共識分離](#分片技術與共識分離)

---

## 6.2 Rollups 架構

### Optimistic Rollups 架構設計

Optimistic Rollup 將大量 L2 交易批量提交到 L1，默認所有批次有效（樂觀假設），任何人可在挑戰期內提交欺詐證明。

#### 核心設計原則

| 機制 | 說明 |
|------|------|
| 挑戰期 | 默認 7 天（足夠時間讓任何人提交欺詐證明） |
| 欺詐證明 | 二分查找定位到錯誤的單一交易，鏈上重新執行 |
| 數據可用性 | 所有 L2 交易資料以 calldata 形式發佈到 L1 |
| 橋接機制 | 存款立即可用，提款需等待挑戰期 |

#### OptimisticRollup 主類

```python
from dataclasses import dataclass, field
from typing import List, Dict, Optional
from enum import Enum
import hashlib
import time

class BatchStatus(Enum):
    PENDING = "pending"
    CHALLENGED = "challenged"
    FINALIZED = "finalized"
    REVERTED = "reverted"

@dataclass
class L2Transaction:
    from_address: str
    to_address: str
    amount: int
    nonce: int
    signature: bytes

@dataclass
class StateBatch:
    batch_id: int
    prev_state_root: str
    new_state_root: str
    transactions: List[L2Transaction]
    l1_block_number: int
    submitter: str
    timestamp: int
    status: BatchStatus = BatchStatus.PENDING

class OptimisticRollup:
    CHALLENGE_PERIOD = 7 * 24 * 3600  # 7 天（秒）
    
    def __init__(self, l1_contract_address: str):
        self.l1_contract = l1_contract_address
        self.batches: Dict[int, StateBatch] = {}
        self.state_root = self._genesis_state_root()
        self.l2_balances: Dict[str, int] = {}
        self.batch_counter = 0
    
    def submit_batch(self, transactions: List[L2Transaction], sequencer: str) -> int:
        """提交一批 L2 交易到 L1"""
        prev_root = self.state_root
        new_state = self._apply_transactions(transactions)
        new_root = self._compute_state_root(new_state)
        
        batch = StateBatch(
            batch_id=self.batch_counter,
            prev_state_root=prev_root,
            new_state_root=new_root,
            transactions=transactions,
            l1_block_number=self._get_l1_block(),
            submitter=sequencer,
            timestamp=int(time.time())
        )
        
        self.batches[self.batch_counter] = batch
        self.state_root = new_root
        self.batch_counter += 1
        
        # 將交易資料發佈到 L1（calldata）
        self._publish_to_l1(batch)
        
        return batch.batch_id
    
    def _apply_transactions(self, transactions):
        new_state = dict(self.l2_balances)
        for tx in transactions:
            if new_state.get(tx.from_address, 0) >= tx.amount:
                new_state[tx.from_address] = new_state.get(tx.from_address, 0) - tx.amount
                new_state[tx.to_address] = new_state.get(tx.to_address, 0) + tx.amount
        return new_state
    
    def _compute_state_root(self, state: dict) -> str:
        sorted_items = sorted(state.items())
        data = str(sorted_items).encode()
        return hashlib.sha256(data).hexdigest()
    
    def finalize_batch(self, batch_id: int):
        """挑戰期結束後最終化批次"""
        batch = self.batches[batch_id]
        elapsed = int(time.time()) - batch.timestamp
        
        if elapsed < self.CHALLENGE_PERIOD:
            raise Exception(f"挑戰期未結束，剩餘 {self.CHALLENGE_PERIOD - elapsed} 秒")
        
        if batch.status != BatchStatus.PENDING:
            raise Exception(f"批次狀態無效：{batch.status}")
        
        batch.status = BatchStatus.FINALIZED
```

#### FraudProofSystem 欺詐證明系統

```python
class FraudProofSystem:
    """欺詐證明：二分查找定位錯誤交易，鏈上重新執行"""
    
    def __init__(self, rollup: OptimisticRollup):
        self.rollup = rollup
    
    def challenge_batch(self, batch_id: int, challenger: str) -> str:
        """發起對批次的挑戰"""
        batch = self.rollup.batches[batch_id]
        
        if batch.status != BatchStatus.PENDING:
            raise Exception("批次不在挑戰狀態")
        
        # 二分查找定位第一個錯誤的交易
        error_index = self._binary_search_error(batch)
        
        if error_index is None:
            raise Exception("批次是有效的，無法發起挑戰")
        
        # 生成單交易欺詐證明
        proof = self._generate_single_tx_proof(batch, error_index)
        
        # 在 L1 上驗證並罰沒提交者
        if self._verify_on_l1(proof):
            self._slash_proposer(batch, challenger)
            batch.status = BatchStatus.REVERTED
            return f"挑戰成功！挑戰者 {challenger} 獲得獎勵"
        
        return "挑戰失敗"
    
    def _binary_search_error(self, batch: StateBatch) -> Optional[int]:
        """二分查找定位第一個使狀態根不符的交易"""
        txs = batch.transactions
        n = len(txs)
        
        # 模擬執行並記錄每步的狀態根
        state_roots = [batch.prev_state_root]
        current_state = self.rollup._load_state(batch.prev_state_root)
        
        for tx in txs:
            current_state = self.rollup._apply_single_tx(current_state, tx)
            state_roots.append(self.rollup._compute_state_root(current_state))
        
        # 檢查最終狀態根是否與提交一致
        if state_roots[-1] == batch.new_state_root:
            return None  # 批次是有效的
        
        # 二分查找
        lo, hi = 0, n - 1
        while lo < hi:
            mid = (lo + hi) // 2
            if state_roots[mid + 1] == self._claimed_root_at(batch, mid):
                lo = mid + 1
            else:
                hi = mid
        
        return lo  # 第一個錯誤的交易索引
    
    def _generate_single_tx_proof(self, batch, tx_index):
        """生成包含前狀態、交易、後狀態的證明"""
        tx = batch.transactions[tx_index]
        pre_state = self._get_state_root_at(batch, tx_index)
        post_state = self._get_state_root_at(batch, tx_index + 1)
        
        # 生成 Merkle 證明（證明前狀態中的帳戶餘額）
        merkle_proofs = self._generate_account_proofs(pre_state, tx)
        
        return {
            'batch_id': batch.batch_id,
            'tx_index': tx_index,
            'transaction': tx,
            'pre_state_root': pre_state,
            'claimed_post_state_root': post_state,
            'account_proofs': merkle_proofs
        }
    
    def _slash_proposer(self, batch: StateBatch, challenger: str):
        """罰沒惡意提交者的保證金並獎勵挑戰者"""
        bond = self.rollup.get_proposer_bond(batch.submitter)
        reward = bond // 2  # 50% 給挑戰者，50% 銷毀
        
        self.rollup.transfer(batch.submitter, challenger, reward)
        self.rollup.burn(batch.submitter, bond - reward)
```

#### OptimisticRollupBridge 橋接協議

```python
class OptimisticRollupBridge:
    """L1 ↔ L2 資產橋接"""
    
    def deposit_to_l2(self, user: str, amount: int, l2_recipient: str) -> str:
        """
        L1 → L2 存款（即時）
        1. 用戶在 L1 鎖定 ETH
        2. Sequencer 在 L2 鑄造對應餘額
        """
        # 1. 在 L1 合約鎖定資金
        deposit_tx = self.l1_contract.lock_funds(user, amount)
        
        # 2. 等待 L1 確認
        l1_block = self.wait_l1_confirmations(deposit_tx, confirmations=12)
        
        # 3. L2 鑄造
        self.l2_state.mint(l2_recipient, amount)
        
        deposit_id = f"dep_{deposit_tx.hash}"
        return deposit_id
    
    def initiate_withdrawal(self, l2_user: str, amount: int, l1_recipient: str) -> str:
        """
        L2 → L1 提款（需要等待 7 天挑戰期）
        """
        # 1. 在 L2 銷毀代幣
        self.l2_state.burn(l2_user, amount)
        
        # 2. 提交提款請求到 L1
        withdrawal = self.l1_contract.record_withdrawal(
            l2_user=l2_user,
            l1_recipient=l1_recipient,
            amount=amount,
            l2_state_root=self.l2_state.current_root(),
            timestamp=int(time.time())
        )
        
        return withdrawal.id  # 提款需要等待挑戰期後調用 finalize_withdrawal
    
    def finalize_withdrawal(self, withdrawal_id: str):
        """7 天後最終化提款，釋放 L1 資金"""
        withdrawal = self.l1_contract.get_withdrawal(withdrawal_id)
        
        # 驗證挑戰期已過
        if time.time() - withdrawal.timestamp < 7 * 24 * 3600:
            raise Exception("挑戰期未結束")
        
        # 驗證包含提款的 L2 批次已最終化
        if not self.l1_contract.is_batch_finalized(withdrawal.batch_id):
            raise Exception("批次未最終化")
        
        # 釋放 L1 資金
        self.l1_contract.release_funds(withdrawal.l1_recipient, withdrawal.amount)

class MessagePassing:
    """L1 ↔ L2 跨層消息傳遞"""
    
    def send_l1_to_l2(self, sender: str, target_l2_contract: str, 
                      calldata: bytes, gas_limit: int):
        """發送 L1 到 L2 的消息"""
        msg_hash = hashlib.sha256(
            sender.encode() + target_l2_contract.encode() + calldata
        ).hexdigest()
        
        # L1 合約記錄消息
        self.l1_contract.enqueue_message(msg_hash, gas_limit)
        
        # Sequencer 在 L2 執行消息（保序）
        return msg_hash
```

---

### 零知識證明 Rollups 技術

ZK-Rollup 使用零知識證明（通常是 zk-SNARKs 或 zk-STARKs）來驗證 L2 交易批次的正確性，無需挑戰期。

#### 核心優勢對比 Optimistic Rollup

| 特性 | Optimistic Rollup | ZK Rollup |
|------|-------------------|-----------|
| 提款等待期 | 7 天 | 即時（證明驗證後）|
| 數據可用性 | 完整 calldata | 只需狀態差異 |
| 計算開銷 | 低（L2 側） | 高（証明生成） |
| EVM 相容性 | 完全（Arbitrum/Optimism） | 有限（zkEVM 仍在完善）|

> **🔄 技術更新（2025+）**：zkEVM 成熟度已大幅提升，Vitalik 定義了 4 種類型：Type 1（完全等效 EVM，如 Taiko）、Type 2（幾乎等效，如 Scroll、Polygon zkEVM）、Type 3/4（高效能但相容性有取捨，如 zkSync Era、StarkNet）。到 2025 年，主流 zkEVM 均已達 Type 2 以上，EVM 相容性問題基本解決。

#### ZKRollupCircuit 電路設計

```python
class ZKRollupCircuit:
    """
    定義 ZK-Rollup 的狀態轉移電路
    約束：new_state_root = apply_batch(old_state_root, transactions)
    """
    
    def __init__(self, batch_size: int, tree_depth: int):
        self.batch_size = batch_size
        self.tree_depth = tree_depth  # Merkle 樹深度（log₂ of account count）
    
    def define_constraints(self, builder):
        """定義所有電路約束"""
        
        # 公共輸入（L1 合約可驗證）
        old_root = builder.public_input('old_state_root')
        new_root = builder.public_input('new_state_root')
        tx_batch_hash = builder.public_input('tx_batch_hash')
        
        # 私有 witness（只有 Prover 知道）
        transactions = builder.private_input('transactions')
        merkle_paths = builder.private_input('merkle_paths')
        
        # 約束 1：所有交易簽名有效
        for tx in transactions:
            builder.assert_valid_signature(
                tx.signature, tx.sender_pubkey, tx.hash()
            )
        
        # 約束 2：交易批次哈希一致
        computed_hash = builder.compute_batch_hash(transactions)
        builder.assert_equal(computed_hash, tx_batch_hash)
        
        # 約束 3：狀態轉移正確
        computed_new_root = self.apply_transactions_circuit(
            builder, old_root, transactions, merkle_paths
        )
        builder.assert_equal(computed_new_root, new_root)
    
    def apply_transactions_circuit(self, builder, state_root, transactions, merkle_paths):
        """電路中的狀態轉移邏輯"""
        current_root = state_root
        
        for i, tx in enumerate(transactions):
            sender_path = merkle_paths[i]['sender']
            receiver_path = merkle_paths[i]['receiver']
            
            # 驗證發送者餘額（Merkle 包含性證明）
            builder.assert_merkle_inclusion(
                current_root, tx.sender, tx.sender_balance, sender_path
            )
            
            # 約束：發送者餘額足夠
            builder.assert_gte(tx.sender_balance, tx.amount + tx.fee)
            
            # 更新發送者餘額
            new_sender_balance = builder.sub(tx.sender_balance, tx.amount + tx.fee)
            intermediate_root_1 = builder.compute_merkle_update(
                current_root, tx.sender, new_sender_balance, sender_path
            )
            
            # 更新接收者餘額（Merkle 更新）
            new_receiver_balance = builder.add(tx.receiver_balance, tx.amount)
            current_root = builder.compute_merkle_update(
                intermediate_root_1, tx.receiver, new_receiver_balance, receiver_path
            )
        
        return current_root
```

#### WitnessGenerator

```python
class WitnessGenerator:
    """生成 ZK 電路所需的 witness 數據"""
    
    def __init__(self, state_tree):
        self.state_tree = state_tree
    
    def generate_batch_witness(self, transactions: List[L2Transaction]):
        """為一批交易生成 witness"""
        witnesses = []
        current_state = self.state_tree.clone()
        
        for tx in transactions:
            # 獲取交易前的帳戶狀態
            sender_account = current_state.get_account(tx.from_address)
            receiver_account = current_state.get_account(tx.to_address)
            
            # 生成 Merkle 路徑（包含性證明）
            sender_merkle_path = current_state.generate_merkle_proof(tx.from_address)
            
            # 更新發送者
            current_state.update_balance(tx.from_address, sender_account.balance - tx.amount - tx.fee)
            
            receiver_merkle_path = current_state.generate_merkle_proof(tx.to_address)
            
            # 更新接收者
            current_state.update_balance(tx.to_address, receiver_account.balance + tx.amount)
            
            witnesses.append({
                'transaction': tx,
                'sender_balance': sender_account.balance,
                'receiver_balance': receiver_account.balance,
                'sender_nonce': sender_account.nonce,
                'sender_merkle_path': sender_merkle_path,
                'receiver_merkle_path': receiver_merkle_path,
            })
        
        return witnesses, current_state.root()
```

#### ZKProofGenerator

```python
class ZKProofGenerator:
    """使用 Groth16 生成 ZK-Rollup 證明"""
    
    def __init__(self, proving_key, circuit):
        self.proving_key = proving_key
        self.circuit = circuit
    
    def generate_batch_proof(self, 
                             old_state_root: str,
                             new_state_root: str,
                             transactions: list,
                             witnesses: list):
        """生成批次狀態轉移的 ZK 證明"""
        
        public_inputs = {
            'old_state_root': old_state_root,
            'new_state_root': new_state_root,
            'tx_batch_hash': self._compute_batch_hash(transactions)
        }
        
        private_witnesses = {
            'transactions': transactions,
            'merkle_paths': [w['sender_merkle_path'] for w in witnesses],
            'balances': [(w['sender_balance'], w['receiver_balance']) for w in witnesses]
        }
        
        # 調用 Groth16 Prover（或 PLONK 等其他證明系統）
        proof = groth16_prove(
            proving_key=self.proving_key,
            public_inputs=public_inputs,
            private_witnesses=private_witnesses
        )
        
        return proof
    
    def _compute_batch_hash(self, transactions):
        data = b''.join(
            tx.from_address.encode() + tx.to_address.encode() + 
            tx.amount.to_bytes(32, 'big')
            for tx in transactions
        )
        return hashlib.sha256(data).hexdigest()

class ZKRollupOperator:
    """ZK-Rollup 運算者：收集交易、生成證明、提交 L1"""
    
    def __init__(self, circuit, proving_key, l1_contract):
        self.circuit = circuit
        self.generator = ZKProofGenerator(proving_key, circuit)
        self.l1_contract = l1_contract
        self.pending_txs = []
        self.BATCH_SIZE = 1000
    
    def add_transaction(self, tx: L2Transaction):
        self.pending_txs.append(tx)
        if len(self.pending_txs) >= self.BATCH_SIZE:
            self.process_batch()
    
    def process_batch(self):
        if not self.pending_txs:
            return
        
        batch = self.pending_txs[:self.BATCH_SIZE]
        self.pending_txs = self.pending_txs[self.BATCH_SIZE:]
        
        # 生成 witness
        old_root = self.l2_state.current_root()
        witness_gen = WitnessGenerator(self.l2_state)
        witnesses, new_root = witness_gen.generate_batch_witness(batch)
        
        # 生成 ZK 證明（可能需要幾分鐘到幾十分鐘）
        proof = self.generator.generate_batch_proof(
            old_state_root=old_root,
            new_state_root=new_root,
            transactions=batch,
            witnesses=witnesses
        )
        
        # 提交 L1（無需等待挑戰期）
        self.l1_contract.submit_batch_with_proof(
            old_root=old_root,
            new_root=new_root,
            proof=proof,
            transactions_calldata=self._encode_calldata(batch)
        )
```

---

## 6.3 分片設計

### 分片技術與共識分離

分片將區塊鏈網路劃分為多個並行的分片，每個分片只處理部分交易，實現水平擴展。

#### 核心設計參數

```python
SHARD_CONFIG = {
    'num_shards': 64,        # 64 個分片
    'validators_per_shard': 128,  # 每個分片 128 個驗證者
    'min_validators': 2048,  # 最少 128×64/4 = 保安全的最少驗證者數
    'epoch_length': 32,      # slots per epoch
    'slot_time': 12,         # 秒
}
```

#### ShardedBlockchain 實現

```python
from dataclasses import dataclass, field
import random

@dataclass
class ShardBlock:
    shard_id: int
    slot: int
    parent_hash: str
    transactions: list
    state_root: str
    producer: str

@dataclass 
class Shard:
    shard_id: int
    state_root: str = ''
    pending_txs: list = field(default_factory=list)
    block_history: list = field(default_factory=list)
    
    def select_block_producer(self, validators, slot) -> str:
        """VRF 基礎的區塊提議者選擇"""
        # 使用 VRF（可驗證隨機函數）確定性地選擇提議者
        vrf_seed = f"{slot}:{self.shard_id}".encode()
        vrf_output = int(hashlib.sha256(vrf_seed).hexdigest(), 16)
        producer_index = vrf_output % len(validators)
        return validators[producer_index]
    
    def produce_block(self, producer, slot):
        txs = self.pending_txs[:1000]  # 最多 1000 筆交易/塊
        self.pending_txs = self.pending_txs[1000:]
        
        new_state = self._apply_transactions(txs)
        block = ShardBlock(
            shard_id=self.shard_id,
            slot=slot,
            parent_hash=self.block_history[-1].hash if self.block_history else '0x0',
            transactions=txs,
            state_root=new_state,
            producer=producer
        )
        
        self.block_history.append(block)
        self.state_root = new_state
        return block

class ShardedBlockchain:
    def __init__(self, num_shards=64, validators_per_shard=128):
        self.num_shards = num_shards
        self.validators_per_shard = validators_per_shard
        self.shards = {i: Shard(shard_id=i) for i in range(num_shards)}
        self.beacon_chain = BeaconChain(num_shards)
        self.validator_assignments = {}  # validator → shard_id
    
    def assign_validators(self, all_validators: list, epoch: int):
        """隨機分配驗證者到分片（每 epoch 重新分配）"""
        random.shuffle(all_validators)
        
        for i, validator in enumerate(all_validators):
            shard_id = i % self.num_shards
            self.validator_assignments[validator] = shard_id
    
    def process_slot(self, slot: int):
        """處理一個 Slot：所有分片並行出塊"""
        shard_blocks = []
        
        for shard_id, shard in self.shards.items():
            shard_validators = self._get_shard_validators(shard_id)
            producer = shard.select_block_producer(shard_validators, slot)
            block = shard.produce_block(producer, slot)
            shard_blocks.append(block)
        
        # 信標鏈聚合分片塊頭
        self.beacon_chain.process_shard_blocks(shard_blocks, slot)
    
    def _get_shard_validators(self, shard_id: int) -> list:
        return [v for v, s in self.validator_assignments.items() if s == shard_id]
```

#### BeaconChain 信標鏈

```python
@dataclass
class ShardHeader:
    shard_id: int
    state_root: str
    block_hash: str
    attestations: list  # 委員會的簽名

class BeaconChain:
    def __init__(self, num_shards: int):
        self.num_shards = num_shards
        self.shard_headers: Dict[int, List[ShardHeader]] = {}
        self.finalized_state = {}
    
    def process_shard_blocks(self, shard_blocks: list, slot: int):
        """聚合各分片的區塊頭到信標鏈"""
        for block in shard_blocks:
            if block.shard_id not in self.shard_headers:
                self.shard_headers[block.shard_id] = []
            
            header = ShardHeader(
                shard_id=block.shard_id,
                state_root=block.state_root,
                block_hash=self._compute_block_hash(block),
                attestations=[]  # 委員會成員的簽名
            )
            self.shard_headers[block.shard_id].append(header)
    
    def apply_casper_ffg(self, epoch: int):
        """Casper FFG 最終性：2/3 委員會投票 → 最終化"""
        for shard_id in range(self.num_shards):
            headers = self.shard_headers.get(shard_id, [])
            for header in headers:
                if self._has_quorum(header.attestations):
                    self.finalized_state[shard_id] = header.state_root
```

#### CrossShardManager 跨分片通訊

```python
class CrossShardManager:
    """跨分片交易：二階段協議（鎖定-釋放）"""
    
    def __init__(self, blockchain: ShardedBlockchain):
        self.blockchain = blockchain
        self.pending_cross_shard: dict = {}  # tx_id → pending transfer
    
    def initiate_cross_shard_transfer(self, 
                                       from_shard: int, 
                                       to_shard: int,
                                       sender: str, 
                                       recipient: str,
                                       amount: int) -> str:
        """
        跨分片轉帳：兩階段協議
        Phase 1: 在源分片鎖定資金
        Phase 2: 在目標分片解鎖資金
        """
        tx_id = self._generate_tx_id(from_shard, to_shard, sender, amount)
        
        # Phase 1: 鎖定源分片的資金
        source_shard = self.blockchain.shards[from_shard]
        source_shard.lock_balance(sender, amount, tx_id)
        
        # 生成跨分片收據（包含 Merkle 證明）
        receipt = self._generate_receipt(from_shard, sender, amount, tx_id)
        
        # 記錄待處理的跨分片交易
        self.pending_cross_shard[tx_id] = {
            'from_shard': from_shard,
            'to_shard': to_shard,
            'recipient': recipient,
            'amount': amount,
            'receipt': receipt,
            'status': 'LOCKED'
        }
        
        return tx_id
    
    def finalize_cross_shard_transfer(self, tx_id: str):
        """Phase 2: 在目標分片完成轉帳"""
        pending = self.pending_cross_shard.get(tx_id)
        if not pending:
            raise Exception("跨分片交易不存在")
        
        # 驗證源分片的收據（Merkle 包含性證明）
        receipt = pending['receipt']
        if not self._verify_receipt_inclusion(receipt, pending['from_shard']):
            raise Exception("收據驗證失敗")
        
        # 在目標分片解鎖/鑄造資金
        target_shard = self.blockchain.shards[pending['to_shard']]
        target_shard.credit_balance(pending['recipient'], pending['amount'])
        
        # 從源分片銷毀鎖定的資金
        source_shard = self.blockchain.shards[pending['from_shard']]
        source_shard.burn_locked_balance(tx_id)
        
        pending['status'] = 'COMPLETED'
    
    def _generate_receipt(self, shard_id, sender, amount, tx_id):
        """生成跨分片收據，包含狀態 Merkle 證明"""
        shard = self.blockchain.shards[shard_id]
        merkle_proof = shard.generate_state_proof(sender)
        
        return {
            'tx_id': tx_id,
            'shard_id': shard_id,
            'sender': sender,
            'amount': amount,
            'state_root': shard.state_root,
            'merkle_proof': merkle_proof
        }
```

#### 分片技術挑戰

| 挑戰 | 說明 | 解決方案 |
|------|------|---------|
| 數據可用性 | 分片數據可能被扣留 | 糾刪碼 (Erasure Coding) + 抽樣 |
| 跨分片通訊 | 原子性難保證 | 2PC 協議 + 收據機制 |
| 驗證者管理 | 分片切換後舊驗證者仍有機會作惡 | 短暫分片持有期 + Slashing |
| 負載均衡 | 交易不均勻分佈到分片 | 基於帳戶/交易哈希的路由 |

#### 以太坊 Danksharding 路線圖
```
2023: EIP-4844 Proto-Danksharding
  → 添加 blob 交易類型（~128KB/blob，6個blob/block）
  → 為 Rollup 提供臨時數據存儲
  → 不完整分片，但大幅降低 Rollup 成本

2025+: 完整 Danksharding
  → 64 個數據分片
  → 每個分片 ~128KB blob
  → 全網 ~16 MB/block 數據可用性
  → 配合 KZG 承諾 + 數據可用性採樣 (DAS)
```

> **🔄 技術更新（2025+）**：
> - **EIP-4844（Dencun 升級，2024年3月）**：已正式上線，blob 費用市場獨立於 Gas，Rollup 交易成本降低 90%+，Arbitrum/Optimism 用戶手續費大幅下降。
> - **PeerDAS（EIP-7594）**：完整 Danksharding 的中間步驟，透過 P2P 節點採樣來提供數據可用性，預計 2025-2026 年上線。
> - **Arbitrum Stylus（2024）**：允許以 Rust / C / C++ 編寫智能合約並編譯為 WASM，在 L2 執行時比 Solidity 快 10-100 倍，且互相可組合。
> - **Based Rollup**：新型 Rollup 架構，由 L1 驗證者（proposer）直接排序 L2 交易，繼承 L1 的去中心化特性，代表項目為 Taiko。
> - **主要 L2 生態（2025）**：Arbitrum One、OP Mainnet、Base（Coinbase）、zkSync Era、Starknet、Scroll、Linea、Blast、Mode 等已形成多元競爭格局，Base 日交易量已超過 Ethereum 主網。
