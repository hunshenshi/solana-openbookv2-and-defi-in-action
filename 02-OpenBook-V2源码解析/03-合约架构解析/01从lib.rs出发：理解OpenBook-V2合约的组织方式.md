`OpenBook-V2`是用`Anchor`框架开发的Rust项目，Rust的`lib`项目程序的入口是`lib.rs`，用Anchor框架开发的Solana合约入口也是`lib.rs`，那接下来就从`lib.rs`入手，学习下OpenBook-V2的代码哲学。

OpenBook-V2是一个标准的基于Anchor框架开发的Solana合约项目，所以`lib.rs`位于`programs/openbook-v2/src`中，其代码大体分为三块内容，关键代码如下：

``` Rust
...
declare_id!("opnb2LAfJYbRMAHHvqjCwQxanZn7ReEHp1k81EohpZb");
...
#[program]
pub mod openbook_v2 {
    use super::*;

    /// Create a [`Market`](crate::state::Market) for a given token pair.
    #[allow(clippy::too_many_arguments)]
    pub fn create_market(
        ctx: Context<CreateMarket>,
        name: String,
        oracle_config: OracleConfigParams,
        quote_lot_size: i64,
        base_lot_size: i64,
        maker_fee: i64,
        taker_fee: i64,
        time_expiry: i64,
    ) -> Result<()> {
        #[cfg(feature = "enable-gpl")]
        instructions::create_market(
            ctx,
            name,
            oracle_config,
            quote_lot_size,
            base_lot_size,
            maker_fee,
            taker_fee,
            time_expiry,
        )?;
        Ok(())
    }

    /// Close a [`Market`](crate::state::Market) (only
    /// [`close_market_admin`](crate::state::Market::close_market_admin)).
    pub fn close_market(ctx: Context<CloseMarket>) -> Result<()> {
        #[cfg(feature = "enable-gpl")]
        instructions::close_market(ctx)?;
        Ok(())
    }
    ...
    pub fn place_order<'c: 'info, 'info>(
        ctx: Context<'_, '_, 'c, 'info, PlaceOrder<'info>>,
        args: PlaceOrderArgs,
    ) -> Result<Option<u128>> {
        ...
        #[cfg(feature = "enable-gpl")]
        return instructions::place_order(ctx, order, args.limit);

        #[cfg(not(feature = "enable-gpl"))]
        Ok(None)
    }
}
...
#[derive(AnchorSerialize, AnchorDeserialize, Debug, Copy, Clone)]
#[cfg_attr(feature = "arbitrary", derive(arbitrary::Arbitrary))]
pub struct PlaceOrderArgs {
    pub side: Side,
    pub price_lots: i64,
    pub max_base_lots: i64,
    pub max_quote_lots_including_fees: i64,
    pub client_order_id: u64,
    pub order_type: PlaceOrderType,
    pub expiry_timestamp: u64,
    pub self_trade_behavior: SelfTradeBehavior,
    // Maximum number of orders from the book to fill.
    //
    // Use this to limit compute used during order matching.
    // When the limit is reached, processing stops and the instruction succeeds.
    pub limit: u8,
}

#[derive(AnchorSerialize, AnchorDeserialize, Debug, Copy, Clone)]
#[cfg_attr(feature = "arbitrary", derive(arbitrary::Arbitrary))]
pub struct PlaceMultipleOrdersArgs {
    pub price_lots: i64,
    pub max_quote_lots_including_fees: i64,
    pub expiry_timestamp: u64,
}
...
```

* 第一部分使用`declare_id!`指明合约在链上的地址。
* 第二部分使用`#[program]`标记合约指令的集合，包括`create_market`、`close_market`和`place_order`等指令。
* 第三部分是从各个指令参数中提炼出的结构体，如`PlaceOrderArgs`和`PlaceMultipleOrdersArgs`等。
> 其中第一二部分是Anchor框架所要求的，第三部分是为了代码整洁而根据个人喜好提炼出来的。

## 指令集
`lib.rs`中最主要的逻辑就是使用宏`#[program]`标记的部分，这部分包括了整个合约的指令逻辑，这些指令分为四类，分别为交易池指令、订单指令和配置指令。
* 交易池指令
	包括`create_market`和`close_market`，主要负责交易池的创建。
* 订单指令
	包括`place_order`、`edit_order`和`cancel_all_orders`等与订单相关的创建、编辑与取消指令。
* 配置指令
	包括`settle_funds`、`sweep_fees`和`set_market_expired`等与交易池设置相关的指令。

接下来分析下`create_market`指令，来剖析下整体架构。
代码比较简单，其参数是`ctx: Context<CreateMarket>`和`name`、`oracle_config`等一些配置参数，然后就是调用`instructions::create_market`方法创建一个market，具体的实现逻辑在`instructions`模块中，这里还涉及到一个`ctx`的参数，传入的是一个`CreateMarket`类型的结构体，位于`accounts_ix`模块中。

## 功能模块
在上一节中，我们看到了 `lib.rs` 中以 `#[program]` 宏标注的指令集是`OpenBook-V2` 合约的核心接口部分，每个指令都是一个函数调用，真正的业务逻辑放到了`instructions`模块中，接下来就看下`OpenBook-V2`具体有哪些模块，主要负责哪些功能。

``` Shell
programs/openbook-v2/
├── src/
│   ├── lib.rs                 # 作为合约的入口，存放着对外暴露的所有指令
│   ├── accounts_ix/           # 定义了指令中所需账户的结构
│   ├── instructions/          # 合约指令逻辑的具体实现
│   ├── state/                 # 账户结构和链上数据
│   ├── accounts_zerocopy.rs   # 通过内存映射读取账户数据，避免序列化
│   ├── error.rs               # 自定义error信息
│   ├── logs.rs                # 将log通过栈方向记录，一种内存优化方式
│   ├── util.rs                # 一些工具func
```

主要模块是`instructions`、`accounts_ix`和`state`模块，而且交互也比较紧密，不过也可以将`instructions`和`accounts_ix`结合在一起先阅读，然后在单独阅读`state`模块，理解下`orderbook`设计的精妙之处。

更多内容可查看在github上的项目--[深入Solana OpenBook-V2源码分析与DeFi 合约实战](https://github.com/hunshenshi/solana-openbookv2-and-defi-in-action)