core->types->transaction.go

```go
// Transaction is an Ethereum transaction.
type Transaction struct {
	inner TxData    // Consensus contents of a transaction
	time  time.Time // Time first seen locally (spam avoidance)

	// caches
	hash atomic.Value
	size atomic.Value
	from atomic.Value
}
```

txdata

```go
// TxData is the underlying data of a transaction.
//
// This is implemented by DynamicFeeTx, LegacyTx and AccessListTx.
type TxData interface {
	txType() byte // returns the type ID
	copy() TxData // creates a deep copy and initializes all fields

	chainID() *big.Int
	accessList() AccessList
	data() []byte
	gas() uint64
	gasPrice() *big.Int
	gasTipCap() *big.Int
	gasFeeCap() *big.Int
	value() *big.Int
	nonce() uint64
	to() *common.Address

	rawSignatureValues() (v, r, s *big.Int)
	setSignatureValues(chainID, v, r, s *big.Int)
}
```

对应两个创建方法，内部调用newTransaction实现

1. 创建普通交易
2. 创建合约，to字段为nil

```go
func NewTransaction(nonce uint64, to common.Address, amount *big.Int, gasLimit uint64, gasPrice *big.Int, data []byte) *Transaction {
 return newTransaction(nonce, &to, amount, gasLimit, gasPrice, data)
}

func NewContractCreation(nonce uint64, amount *big.Int, gasLimit uint64, gasPrice *big.Int, data []byte) *Transaction {
 return newTransaction(nonce, nil, amount, gasLimit, gasPrice, data)
}
```

有一个Message字段和创建方法NewMessage，不知道是做什么用的，注释描述为

> // Message is a fully derived transaction and implements core.Message
>
> // NOTE: In a future PR this will be removed.

```go
type Message struct {
 to         *common.Address
 from       common.Address
 nonce      uint64
 amount     *big.Int
 gasLimit   uint64
 gasPrice   *big.Int
 data       []byte
 checkNonce bool
}
```

当一笔交易手续费过低时，可以调整gasprice，==此时其实是创建了一条与上条交易相同nonce值且GasPrice高于之前上条交易GasPrice一定百分比的数值的交易。==至于超出多少需要看交易池的参数设定。
