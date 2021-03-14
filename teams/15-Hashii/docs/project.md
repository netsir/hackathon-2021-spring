## NFTSwap

中文
===========================================================================================================================================================

### 项目背景/原由/要解决的问题
    
NFT是「non-fungible Token」,非同质化代币。在技术的角度上说是解决唯一性和独特价值token，在属性上它代表的是一种被赋予了某种价值（这种价值可能是艺术性、稀有性、数学性等）的新型数字资产。

 NFTSwap是一个智能协约框架，采用去中心质押池形成的NFT价值评估体系。每个可拍卖NFT对应一个质押池，用户可以通过质押来对NFT的的估值提供价值认可，通过NFTSwap托管拍卖的NFT需要支付最终成交10-20%的佣金，这个佣金将会分给提供价值认可提供者（VP）。
 
 NFTSwap对NFT的托管、拍卖、估值、质押提供一揽子协议框架，为NFT的去中心管理提供契约参考。

#### NFT托管

用户可以将自己持有的NFT资产，转入NFTSwap智能契约进行托管。在NFT托管期间，NFT持有者可以对NFTSwap下达指令，进行NFT的提取、拍卖、信托等操作。
### 项目技术设计

#### NFT托管

用户可以将自己持有的NFT资产，转入NFTSwap智能契约进行托管。在NFT托管期间，NFT持有者可以对NFTSwap下达指令，进行NFT的提取、拍卖、信托等操作。

#### NFT拍卖

NFT持有者，可以对智能契约发出拍卖指令。拍卖需要注明最高价与最低价，并设定拍卖期，拍卖期按照区块高度进行设置，拍卖期最短不得少于6000个区块。拍卖期间NFT持有者无法对NFT进行操作；拍卖一旦成功，NFT即刻转让给最高出价者，到期没有完成拍卖（流拍），NFT持有者可以对NFT进行下一步指令。
拍卖最高价与最低价一致，简称为一口价，即购买者必须按照出价进行行购买。
拍卖最高价设置为不可能达到的价格，称为天价，即购买者必须等到拍卖期结束，价高者得。
竞买者出价不小于最低价，且不小于当前最高出价，一旦竞买者出价达到最高价，交易即可达成。

#### 一、交易接口

1. 创建Nft艺术品

```rust
pub fn create(
  origin, 
  title: Vec<u8>,  // 标题
  url: Vec<u8>,  // 链接
  desc: Vec<u8> // 详情
)
```

2. 移除Nft

```rust
pub fn remove(
  origin, 
  nft_id: T::NftId // 艺术品Id
)
```

3. 转移Nft艺术品

```rust
pub fn transfer(
  origin, 
  target: T::AccountId, // 转移目标
  nft_id: T::NftId // 艺术品Id
)
```

4. 下拍卖单出售艺术品

```rust
pub fn order_sell(
  origin, 
  nft_id: T::NftId,  // 艺术品Id
  start_price: BalanceOf<T>, // 起拍价格
  end_price: BalanceOf<T>, // 结拍价格
  keep_block_num: T::BlockNumber // 拍卖最大保留区块数量
)
```

5. 竞拍Nft艺术品

```rust
pub fn order_buy(
  origin, 
  order_id: T::OrderId, // 订单Id
  price: BalanceOf<T> // 竞拍价格
)
```

6. 主动结算拍卖 // 用于到期结算

```rust
pub fn order_settlement(
  origin, 
  order_id: T::OrderId // 订单Id
)
```

7. 进行投票质押

```rust
pub fn vote_order(
  origin, 
  order_id: T::OrderId,  // 订单Id
  amount: BalanceOf<T> // 质押数量
)
```



#### 二、trait Type: 类型信息/常数

##### 注入类型

- NftId: Nft艺术品Id
- OrderId: 订单Id

##### 常数

- MinKeepBlockNumber: 拍卖订单最小保留区块数
- MaxKeepBlockNumber: 拍卖订单最大保留区块数
- MinimumPrice: 最小拍卖价格
- MinimumVotingLock: 最小质押投票数量
- FixRate: 用于分润算法的固定利润常数
- ProfitRate: 参与质押的分润比例

##### 复合类型

- Nft艺术品

```rust
pub struct Nft {
	pub title: Vec<u8>, // 标题
	pub url: Vec<u8>, // 链接
	pub desc: Vec<u8>, // 详情
}
```

- 拍卖单

```rust
pub struct Order<OrderId, NftId, AccountId, Balance, BlockNumber> {
	pub order_id: OrderId, // 订单Id
	pub start_price: Balance, // 起拍价格
	pub end_price: Balance, // 结拍价格
	pub nft_id: NftId, // nftId
	pub create_block: BlockNumber, // 创建时区块数
	pub keep_block_num: BlockNumber, // 最大保留区块数
	pub owner: AccountId, // nft所有者
}
```

- 竞拍单

```rust
pub struct Bid<OrderId, AccountId, Balance> {
	pub order_id: OrderId, // 订单Id
	pub price: Balance, // 竞拍价格
	pub owner: AccountId, // 竞拍人
}
```

- 质押投票

```rust
pub struct Vote<OrderId, AccountId, Balance, BlockNumber> {
	pub order_id: OrderId, // 订单Id
	pub amount: Balance, // 质押数量
	pub keep_block_num: BlockNumber, // 通过拍卖单生成的质押区块长度
	pub owner: AccountId, // 质押者
}
```



#### 三、Storage: 存储数据结构

1. Map nftId -> nft详情， 用于存储所有nft

```rust
pub Nfts: map hasher(twox_64_concat) T::NftId => Option<Nft>;
```

2. Map nftId -> 账户Id， 用于记录nft所有者

```rust
pub NftAccount: map hasher(twox_64_concat) T::NftId => T::AccountId;
```

3. Map nftId -> 订单Id， 用于记录Nft对应的订单数据

```rust
pub NftOrder: map hasher(twox_64_concat) T::NftId => Option<T::OrderId>;
```

4. Map  订单Id -> 订单详情, 用于存储所有待完成的拍卖订单

```rust
pub Orders: map hasher(twox_64_concat) T::OrderId => Option<OrderOf<T>>;
```

5. Map 订单Id -> 当前最大出价，用于存储当前订单的最大出价

```rust
pub Bids: map hasher(twox_64_concat) T::OrderId => Option<BidOf<T>>;
```

6. Map 订单Id -> 质押投票列表, 用于存储质押列表

```rust
pub Votes: map hasher(twox_64_concat) T::OrderId => Vec<VoteOf<T>>;
```

7. NftId生成器，递增

```rust
pub NextNftId: T::NftId;
```

8. 拍卖订单Id生成器，递增

```
pub NextOrderId: T::OrderId;
```







### 项目现在做到的程度

已完成NFTSwap 1.0版本开发，支持用户在平台创建NFT、托管NFT、NFT拍卖、质押资产、参与拍卖等操作。


### 项目遇到的技术难点及解决方案

收益算法：


### 项目如果报名时已经做到一定高度 (之前已经开始做)，请列点说明在黑客松期间 (2月1日 - 3月15日) 所完成的事项/开发工作。

 期间完成了NFT创建、NFT托管、发起拍卖、价值认可、参与拍卖、收益自动结算等代码开发功能。

### 未来6个月的商业规划

2021年3月，NFTSwap框架在波卡生态推出，并开源。
2021年4月，成立NFT专项基金，扶持NFT生态，同时将NFTSwap框架在以太坊、币安智能链、火币链部署。
2021年5～9月，发起倡议成立NFT标准协会，联合生态伙伴制定NFT在艺术品、游戏、门票、身份、资产等多领域的解决方案，并将标准推广到主流公链。

### 市场定位及调研

#### 产品定位
全球最大的NFT资产价值估价拍卖平台。

#### 调研情况
与多家画廊、多名艺术家、行业KOL的沟通，对方都表达出强烈合作意愿，也都普遍认可NFTSwap框架可以解决NFT合理定价困难，宣发成本高等问题。

### 现在拥有的资源及项目运营到什么程度

举办北京最大线下艺术展的经验，正在筹办上海艺术展。

已合作了数百家画廊，有海量插画师和艺术家资源，可以为NFTSwap输入优质的NFT资产。

与国内头部区块链媒体平台有深度合作，有强大的媒体宣发能力。

======================================================================================================================================================
Englist
aaa
