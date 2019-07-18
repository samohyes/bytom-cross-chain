**简介**

Bystack是由比原链团队提出的一主多侧链架构的BaaS平台。其将区块链应用分为三层架构：底层账本层，侧链扩展层，业务适配层。底层账本层为Layer1，即为目前比较成熟的采用POW共识的Bytom公链。侧链扩展层为Layer2，为多侧链层，vapor侧链即处于Layer2。

![img](C:/Users/Xudon/AppData/Local/YNote/data/qqC54FDAFE71043EFC14E8EB26E62C5C10/098180d3f0be4abea911ab980f534f2b/%E6%8D%95%E8%8E%B7.png)

(图片来自Bystack白皮书)

Vapor侧链采用DPOS和BBFT共识，TPS可以达到数万。此处就分析一下连接Bytom主链和Vapor侧链的跨链模型。





**主侧链协同工作模型**

![img](C:/Users/Xudon/AppData/Local/YNote/data/qqC54FDAFE71043EFC14E8EB26E62C5C10/6a0bfe5a1689471cb33ec95d78e67615/321e55de6d324d2f8f846a3f090c79ee.jpg)

**1.技术细节**

(1)跨链模型架构

在Bystack的主侧链协同工作模型中，包括有主链、侧链和Federation。主链为bytom，采用基于对AI 计算友好型PoW（工作量证明）算法，主要负责价值锚定，价值传输和可信存证。侧链为Vapor,采用DPOS+BBFT共识，高TPS满足垂直领域业务。主链和侧链之间的资产流通主要依靠Federation。

(2)节点类型

跨链模型中的节点主要有收集人、验证人和联邦成员。收集人监控联邦地址，收集交易后生成Claim交易进行跨链。验证人则是侧链的出块人。联邦成员由侧链的用户投票通过选举产生，负责生成新的联邦合约地址。

(3)跨链交易流程

- 主链到侧链

主链用户将代币发送至联邦合约地址，收集人监控联邦地址，发现跨链交易后生成Claim交易，发送至侧链

- 侧链到主链

侧链用户发起提现交易，销毁侧链资产。收集人监控侧链至主链交易，向主链地址发送对应数量资产。最后联邦在侧链生成一笔完成提现的操作交易。



**2.代码解析**

跨链代码主要处于federation文件夹下，这里就这部分代码进行一个介绍。



(1)keeper启动

整个跨链的关键在于同步主链和侧链的区块，并处理区块中的跨链交易。这部份代码主要在mainchain_keerper.go和sidechain_keerper.go两部分中，分别对应处理主链和侧链的区块。keeper在Run函数中启动。

```go
func (m *mainchainKeeper) Run() {
	ticker := time.NewTicker(time.Duration(m.cfg.SyncSeconds) * time.Second)
	for ; true; <-ticker.C {
		for {
			isUpdate, err := m.syncBlock()
			if err != nil {
				//..
			}
			if !isUpdate {
				break
			}
		}
	}
}
```
Run函数中首先生成一个定时的Ticker，规定每隔SyncSeconds秒同步一次区块，处理区块中的交易。



(2)主侧链同步区块

Run函数会调用syncBlock函数同步区块。

```go
func (m *mainchainKeeper) syncBlock() (bool, error) {
	chain := &orm.Chain{Name: m.chainName}
	if err := m.db.Where(chain).First(chain).Error; err != nil {
		return false, errors.Wrap(err, "query chain")
	}

	height, err := m.node.GetBlockCount()
	//..
	if height <= chain.BlockHeight+m.cfg.Confirmations {
		return false, nil
	}

	nextBlockStr, txStatus, err := m.node.GetBlockByHeight(chain.BlockHeight + 1)
	//..
	nextBlock := &types.Block{}
	if err := nextBlock.UnmarshalText([]byte(nextBlockStr)); err != nil {
		return false, errors.New("Unmarshal nextBlock")
	}
	if nextBlock.PreviousBlockHash.String() != chain.BlockHash {
		//...
		return false, ErrInconsistentDB
	}

	if err := m.tryAttachBlock(chain, nextBlock, txStatus); err != nil {
		return false, err
	}

	return true, nil
}
```




这个函数受限会根据chainName从数据库中取出对应的chain。然后利用GetBlockCount函数获得chain的高度。然后进行一个伪确定性的检测。

height <= chain.BlockHeight+m.cfg.Confirmations 

主要是为了判断链上的资产是否已经不可逆。这里Confirmations的值被设为10。如果不进行这个等待不可逆的过程，很可能主链资产跨链后，主链的最长链改变，导致这笔交易没有在主链被打包，而侧链却增加了相应的资产。在此之后，通过GetBlockByHeight函数获得chain的下一个区块。

nextBlockStr, txStatus, err := m.node.GetBlockByHeight(chain.BlockHeight + 1) 

这里必须满足下个区块的上一个区块哈希等于当前chain中的这个头部区块哈希。这也符合区块链的定义。

```go
if nextBlock.PreviousBlockHash.String() != chain.BlockHash {
    //..
}
```

在此之后，通过调用tryAttachBlock函数进一步调用processBlock函数处理区块。



(3)区块处理

processBlock函数会判断区块中交易是否为跨链的deposit或者是withdraw，并分别调用对应的函数去进行处理。

```go
func (m *mainchainKeeper) processBlock(chain *orm.Chain, block *types.Block, txStatus *bc.TransactionStatus) error {
	if err := m.processIssuing(block.Transactions); err != nil {
		return err
	}

	for i, tx := range block.Transactions {
		if m.isDepositTx(tx) {
			if err := m.processDepositTx(chain, block, txStatus, uint64(i), tx); err != nil {
				return err
			}
		}

		if m.isWithdrawalTx(tx) {
			if err := m.processWithdrawalTx(chain, block, uint64(i), tx); err != nil {
				return err
			}
		}
	}

	return m.processChainInfo(chain, block)
}
```



在这的processIssuing函数，它内部会遍历所有交易输入Input的资产类型，也就是AssetID。当这个AssetID不存在的时候，则会去在系统中创建一个对应的资产类型。每个Asset对应的数据结构如下所示。

```go
m.assetStore.Add(&orm.Asset{
AssetID:           assetID.String(),
IssuanceProgram:   hex.EncodeToString(inp.IssuanceProgram),
VMVersion:         inp.VMVersion,
RawDefinitionByte: hex.EncodeToString(inp.AssetDefinition),
})
```

在processBlock函数中，还会判断区块中每笔交易是否为跨链交易。主要通过isDepositTx和isWithdrawalTx函数进行判断。

```go
func (m *mainchainKeeper) isDepositTx(tx *types.Tx) bool {
	for _, output := range tx.Outputs {
		if bytes.Equal(output.OutputCommitment.ControlProgram, m.fedProg) {
			return true
		}
	}
	return false
}

func (m *mainchainKeeper) isWithdrawalTx(tx *types.Tx) bool {
	for _, input := range tx.Inputs {
		if bytes.Equal(input.ControlProgram(), m.fedProg) {
			return true
		}
	}
	return false
}
```

看一下这两个函数，主要还是通过比较交易中的control program这个标识和mainchainKeeper这个结构体中的fedProg进行比较，如果相同则为跨链交易。fedProg在结构体中为一个字节数组。

```go
type mainchainKeeper struct {
	cfg        *config.Chain
	db         *gorm.DB
	node       *service.Node
	chainName  string
	assetStore *database.AssetStore
	fedProg    []byte
}
```



(4)跨链交易(主链到侧链的deposit)处理

这部分主要分为主链到侧链的deposit和侧链到主链的withdraw。先看比较复杂的主链到侧链的deposit这部分代码的处理。

```go
func (m *mainchainKeeper) processDepositTx(chain *orm.Chain, block *types.Block, txStatus *bc.TransactionStatus, txIndex uint64, tx *types.Tx) error {
	//..

	rawTx, err := tx.MarshalText()
	if err != nil {
		return err
	}

	ormTx := &orm.CrossTransaction{
	      //..
	}
	if err := m.db.Create(ormTx).Error; err != nil {
		return errors.Wrap(err, fmt.Sprintf("create mainchain DepositTx %s", tx.ID.String()))
	}

	statusFail := txStatus.VerifyStatus[txIndex].StatusFail
	crossChainInputs, err := m.getCrossChainReqs(ormTx.ID, tx, statusFail)
	if err != nil {
		return err
	}

	for _, input := range crossChainInputs {
		if err := m.db.Create(input).Error; err != nil {
			return errors.Wrap(err, fmt.Sprintf("create DepositFromMainchain input: txid(%s), pos(%d)", tx.ID.String(), input.SourcePos))
		}
	}

	return nil
}
```

这里它创建了一个跨链交易orm。具体的结构如下。可以看到，这里它的结构体中包括有source和dest的字段。

```go
ormTx := &orm.CrossTransaction{
		ChainID:              chain.ID,
		SourceBlockHeight:    block.Height,
		SourceBlockTimestamp: block.Timestamp,
		SourceBlockHash:      blockHash.String(),
		SourceTxIndex:        txIndex,
		SourceMuxID:          muxID.String(),
		SourceTxHash:         tx.ID.String(),
		SourceRawTransaction: string(rawTx),
		DestBlockHeight:      sql.NullInt64{Valid: false},
		DestBlockTimestamp:   sql.NullInt64{Valid: false},
		DestBlockHash:        sql.NullString{Valid: false},
		DestTxIndex:          sql.NullInt64{Valid: false},
		DestTxHash:           sql.NullString{Valid: false},
		Status:               common.CrossTxPendingStatus,
	}
```

创建这笔跨链交易后，它会将交易存入数据库中。

```go
if err := m.db.Create(ormTx).Error; err != nil {
		return errors.Wrap(err, fmt.Sprintf("create mainchain DepositTx %s", tx.ID.String()))
}
```

在此之后，这里会调用getCrossChainReqs。这个函数内部较为复杂，主要作用就是遍历交易的输出，返回一个跨链交易的请求数组。具体看下这个函数。

```go
func (m *mainchainKeeper) getCrossChainReqs(crossTransactionID uint64, tx *types.Tx, statusFail bool) ([]*orm.CrossTransactionReq, error) {
	//..
	switch {
	case segwit.IsP2WPKHScript(prog):
		//..
	case segwit.IsP2WSHScript(prog):
		//..
	}

	reqs := []*orm.CrossTransactionReq{}
	for i, rawOutput := range tx.Outputs {
		//..

		req := &orm.CrossTransactionReq{
			//..
		}
		reqs = append(reqs, req)
	}
	return reqs, nil
}
```

很显然，这个地方的交易类型有pay to public key hash 和 pay to script hash这两种。这里会根据不同的交易类型进行一个地址的获取。

```go
switch {
	case segwit.IsP2WPKHScript(prog):
		if pubHash, err := segwit.GetHashFromStandardProg(prog); err == nil {
			fromAddress = wallet.BuildP2PKHAddress(pubHash, &vaporConsensus.MainNetParams)
			toAddress = wallet.BuildP2PKHAddress(pubHash, &vaporConsensus.VaporNetParams)
		}
	case segwit.IsP2WSHScript(prog):
		if scriptHash, err := segwit.GetHashFromStandardProg(prog); err == nil {
			fromAddress = wallet.BuildP2SHAddress(scriptHash, &vaporConsensus.MainNetParams)
			toAddress = wallet.BuildP2SHAddress(scriptHash, &vaporConsensus.VaporNetParams)
		}
	}
```

在此之后，函数会遍历所有交易的输出，然后创建跨链交易请求，具体的结构如下。

```go
req := &orm.CrossTransactionReq{
   CrossTransactionID: crossTransactionID,
   SourcePos:          uint64(i),
   AssetID:            asset.ID,
   AssetAmount:        rawOutput.OutputCommitment.AssetAmount.Amount,
   Script:             script,
   FromAddress:        fromAddress,
   ToAddress:          toAddress,
   }
```

创建完所有的跨链交易请求后，返回到processDepositTx中一个crossChainInputs数组中，并存入db。

```go
for _, input := range crossChainInputs {
		if err := m.db.Create(input).Error; err != nil {
			return errors.Wrap(err, fmt.Sprintf("create DepositFromMainchain input: txid(%s), pos(%d)", tx.ID.String(), input.SourcePos))
		}
}
```

到这里，对主链到侧链的deposit已经处理完毕。



(5)跨链交易(侧链到主链的withdraw)交易处理

这部分比较复杂的逻辑主要在sidechain_keeper.go中的processWithdrawalTx函数中。这部分逻辑和上面主链到侧链的deposit逻辑类似。同样是创建了orm.crossTransaction结构体，唯一的改变就是交易的souce和dest相反。这里就不作具体描述了。







**3.跨链优缺点**

- 优点

(1) 跨链模型、代码较为完整。当前有很多项目使用跨链技术，但是真正实现跨链的寥寥无几。

(2) 可以根据不同需求实现侧链，满足多种场景



- 缺点

(1) 跨链速度较慢，需等待10个区块确认，这在目前Bytom网络上所需时间为30分钟左右

(2) 相较于comos、polkadot等项目，开发者要开发侧链接入主网成本较大

(3) 只支持资产跨链，不支持跨链智能合约调用





**4.跨链模型平行对比Cosmos**

- 可扩展性

bystack的主测链协同工作模型依靠Federation，未形成通用协议。其他开发者想要接入其跨链网络难度较大。Cosmos采用ibc协议，可扩展性较强。

- 代码开发进度

vapor侧链已经能够实现跨链。Cosmos目前暂无成熟跨链项目出现，ibc协议处于最终开发阶段。

- 跨链模型

vapor为主侧链模型，Cosmos为Hub-Zone的中继链模型。





**5.参考建议**

- 侧链使用bbft共识，非POW的情况下，无需等待10个交易确认，增快跨链速度