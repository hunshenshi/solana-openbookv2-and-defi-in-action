终于到挂单`place_order`指令了，`place_order`是一个挂单指令，可以挂`bids`买单和`asks`卖单，而且它可以是maker也可以是taker，是一个功能非常强大的指令，而且非常高效和灵活。

在`lib.rs`中的入口函数是`place_order`，代码如下：

``` Rust
    pub fn place_order<'c: 'info, 'info>(
        ctx: Context<'_, '_, 'c, 'info, PlaceOrder<'info>>,
        args: PlaceOrderArgs,
    ) -> Result<Option<u128>> {
        require_gte!(args.price_lots, 1, OpenBookError::InvalidInputPriceLots);

        let time_in_force = match Order::tif_from_expiry(args.expiry_timestamp) {
            Some(t) => t,
            None => {
                msg!("Order is already expired");
                return Ok(None);
            }
        };
        let order = Order {
            side: args.side,
            max_base_lots: args.max_base_lots,
            max_quote_lots_including_fees: args.max_quote_lots_including_fees,
            client_order_id: args.client_order_id,
            time_in_force,
            self_trade_behavior: args.self_trade_behavior,
            params: match args.order_type {
                PlaceOrderType::Market => OrderParams::Market,
                PlaceOrderType::ImmediateOrCancel => OrderParams::ImmediateOrCancel {
                    price_lots: args.price_lots,
                },
                PlaceOrderType::FillOrKill => OrderParams::FillOrKill {
                    price_lots: args.price_lots,
                },
                _ => OrderParams::Fixed {
                    price_lots: args.price_lots,
                    order_type: args.order_type.to_post_order_type()?,
                },
            },
        };
        #[cfg(feature = "enable-gpl")]
        return instructions::place_order(ctx, order, args.limit);

        #[cfg(not(feature = "enable-gpl"))]
        Ok(None)
    }
```

代码逻辑比较简单，构造了一个`Order`对象，然后调用`instructions/place_order.rs`中的`place_order`指令。

`lib::place_order`的参数是`PlaceOrder`和`PlaceOrderArgs`，`PlaceOrder`是`place_order`指令所需的所有账户，该参数直传给了`instructions::place_order`，`PlaceOrderArgs`就是将所需参数封装成的一个结构体，然后根据这些内容构建一个`Order`对象，再传给`instructions::place_order`，`Order`的结构体如下：

``` Rust
pub struct Order {
    pub side: Side,

    /// Max base lots to buy/sell.
    pub max_base_lots: i64,

    /// Max quote lots to pay/receive including fees.
    pub max_quote_lots_including_fees: i64,

    /// Arbitrary user-controlled order id.
    pub client_order_id: u64,

    /// Number of seconds the order shall live, 0 meaning forever
    pub time_in_force: u16,

    /// Configure how matches with order of the same owner are handled
    pub self_trade_behavior: SelfTradeBehavior,

    /// Order type specific params
    pub params: OrderParams,
}
```

`side`是订单方向，表示是订单的买入还是卖出，`max_base_lots`表示订单最大买入或卖出的`base token`数量（以 lot 为单位），`max_quote_lots_including_fees`表示你愿意支付（买单）或收到（卖单）的最大`quote token`数量（以 lot 为单位），包含手续，`client_order_id`是用户自定义的订单id，方便用户自己跟踪订单，`time_in_force`表示订单的有效时间（单位：秒），`self_trade_behavior`记录的是当自己的订单互相撮合时的处理方式，是一个枚举类型`SelfTradeBehavior`，`params`是订单类型的具体参数，指定订单的类型，是市价单、限价单等，也是一个枚举类型`OrderParams`。

`SelfTradeBehavior`有三种类型，默认是`DecrementTake`类型，订单进行正常撮合，只是不收取手续费，`CancelProvide`取消自己的`maker`订单，继续与其它的订单进行`taker`撮合，`AbortTransaction`是取消整个交易。

``` Rust
pub enum SelfTradeBehavior {
    /// Both the maker and taker sides of the matched orders are decremented.
    /// This is equivalent to a normal order match, except for the fact that no fees are applied.
    #[default]
    DecrementTake = 0,

    /// Cancels the maker side of the trade, the taker side gets matched with other maker's orders.
    CancelProvide = 1,

    /// Cancels the whole transaction as soon as a self-matching scenario is encountered.
    AbortTransaction = 2,
}
```

`OrderParams`有五种类型，表示订单的类型，`Market`表示市价单，以市场当前最优价格立即成交，未成交部分直接取消，不挂单。`ImmediateOrCancel`也叫`IOC`订单，指定限价 `price_lots`，能成交的部分立即成交，不能成交的部分直接取消，不会挂单。`Fixed`是限价单，指定限价 `price_lots`和订单类型 `order_type`，如果不能成交，订单可以挂在订单簿上，等待成交。`OraclePegged`是指锚定预言机价格的限价单，需要指定`price_offset_lots`(在预言机价格基础上偏移多少 lot)、`order_type`和`peg_limit`(锚定价格的最大允许偏差，是一种保护机制，防止预言机异常时成交价格过离谱）。`FillOrKill`也叫`FOK`订单，指定限价`price_lots`，只有在能全部成交的情况下才会成交，否则整个订单直接取消，不会部分成交。

``` Rust
pub enum OrderParams {
    Market,
    ImmediateOrCancel {
        price_lots: i64,
    },
    Fixed {
        price_lots: i64,
        order_type: PostOrderType,
    },
    OraclePegged {
        price_offset_lots: i64,
        order_type: PostOrderType,
        peg_limit: i64,
    },
    FillOrKill {
        price_lots: i64,
    },
}
```

## `place_order`核心逻辑
接下来逐步分析下`instructions::place_order`，代码如下：

``` Rust
pub fn place_order<'c: 'info, 'info>(
    ctx: Context<'_, '_, 'c, 'info, PlaceOrder<'info>>,
    order: Order,
    limit: u8,
) -> Result<Option<u128>> {
    ...
    Ok(order_id)
}
```

此方法涉及三个参数，其中`PlaceOrder`是`place_order`指令所需的所有账户，`Order`封装了订单的一些通用属性，`limit`是一个阈值，防止指令消耗太多的资源，当达到此值时，停止后续流程，直接返回成功。

先看下`PlaceOrder`的代码，理解其下单时都需要哪些账户。

``` Rust
#[derive(Accounts)]
pub struct PlaceOrder<'info> {
    pub signer: Signer<'info>,
    #[account(
        mut,
        has_one = market,
        constraint = open_orders_account.load()?.is_owner_or_delegate(signer.key()) @ OpenBookError::NoOwnerOrDelegate
    )]
    pub open_orders_account: AccountLoader<'info, OpenOrdersAccount>,
    pub open_orders_admin: Option<Signer<'info>>,

    #[account(
        mut,
        token::mint = market_vault.mint
    )]
    pub user_token_account: Account<'info, TokenAccount>,

    #[account(
        mut,
        has_one = bids,
        has_one = asks,
        has_one = event_heap,
        constraint = market.load()?.oracle_a == oracle_a.non_zero_key(),
        constraint = market.load()?.oracle_b == oracle_b.non_zero_key(),
        constraint = market.load()?.open_orders_admin == open_orders_admin.non_zero_key() @ OpenBookError::InvalidOpenOrdersAdmin
    )]
    pub market: AccountLoader<'info, Market>,
    #[account(mut)]
    pub bids: AccountLoader<'info, BookSide>,
    #[account(mut)]
    pub asks: AccountLoader<'info, BookSide>,
    #[account(mut)]
    pub event_heap: AccountLoader<'info, EventHeap>,
    #[account(
        mut,
        // The side of the vault is checked inside the ix
        constraint = market.load()?.is_market_vault(market_vault.key())
    )]
    pub market_vault: Account<'info, TokenAccount>,

    /// CHECK: The oracle can be one of several different account types and the pubkey is checked above
    pub oracle_a: Option<UncheckedAccount<'info>>,
    /// CHECK: The oracle can be one of several different account types and the pubkey is checked above
    pub oracle_b: Option<UncheckedAccount<'info>>,

    pub token_program: Program<'info, Token>,
}
```

`signer`和`token_program`见过很多次，这里介绍下其他的账户。
`open_orders_account`是用户挂单账户，记录了用户当前的挂单、资产持仓等信息，因为会调整`open_orders_account`中的订单信息，所以约束中增加了`mut`，`has_one`是指该账户中的某个字段值等于`market`的账户地址，`constraint`是一个通用的验证机制，可以做任意逻辑判断，`has_one`的逻辑也可以用`constraint`实现。`open_orders_admin`是可选的**挂单管理员**，在某些特殊场景下需要挂单管理员进行多签验证。`user_token_account`是用户下单时支付`token`的账户(买单时则是quote token，卖单时则是base token)，涉及到转账token到market到资金池中，余额会发生变化，约束中增加了`mut`，增加`token::mint = market_vault.mint`约束确保token类型一致。
`market`、`bids`、`asks`、`event_heap`和`market_vault`都是与交易市场相关的账户，`market`记录当前订单的交易市场，`bids`和`asks`分别为该市场的买入订单薄和卖出订单簿，`event_heap`是事件堆，用于记录撮合结果，`market_vault`是市场的资金池账户。
`oracle_a`和`oracle_b`分别为外部预言机账户，用于一些市场限价控制或者风控。

参数已经梳理完了，现在梳理下下单的内部逻辑，这块实现代码量较大，这里将整块逻辑划分五块进行拆分，主要分为**市场状态加载**、**撮合订单**、**计算需锁定的用户保证金**、**事件堆惩罚计数**和**资产转账**。

### 市场状态加载
市场状态加载代码如下：

``` Rust
    let mut open_orders_account = ctx.accounts.open_orders_account.load_mut()?;
    let open_orders_account_pk = ctx.accounts.open_orders_account.key();

    let clock = Clock::get()?;

    let mut market = ctx.accounts.market.load_mut()?;
    require_keys_eq!(
        market.get_vault_by_side(order.side),
        ctx.accounts.market_vault.key(),
        OpenBookError::InvalidMarketVault
    );
    require!(
        !market.is_expired(clock.unix_timestamp),
        OpenBookError::MarketHasExpired
    );

    let mut book = Orderbook {
        bids: ctx.accounts.bids.load_mut()?,
        asks: ctx.accounts.asks.load_mut()?,
    };
    let mut event_heap = ctx.accounts.event_heap.load_mut()?;
    let event_heap_size_before = event_heap.len();

    let now_ts: u64 = clock.unix_timestamp.try_into().unwrap();

    let oracle_price_lots = market.oracle_price_lots(
        AccountInfoRef::borrow_some(ctx.accounts.oracle_a.as_ref())?.as_ref(),
        AccountInfoRef::borrow_some(ctx.accounts.oracle_b.as_ref())?.as_ref(),
        clock.slot,
    )?;
```

首先需要加载`open_orders_account`账户，因为未成交的订单需要挂载到该账户下，随后加载`market`账户和撮合交易时需要用到的订单树`bids/asks`账户，账户加载之后，还需要一些校验以保证代码的合法性，最后加载`event_heap`和`oracle_price_lots`，`event_heap`主要用于存放事件，然后根据事件堆的长度判断是否需要惩罚。

### 撮合订单
撮合订单调用`book.new_order`进行撮合和挂单，返回订单执行结果（成交量、挂单量、费用等）。

``` Rust
    let OrderWithAmounts {
        order_id,
        total_base_taken_native,
        total_quote_taken_native,
        posted_base_native,
        posted_quote_native,
        taker_fees,
        maker_fees,
        ..
    } = book.new_order(
        &order,
        &mut market,
        &ctx.accounts.market.key(),
        &mut event_heap,
        oracle_price_lots,
        Some(&mut open_orders_account),
        &open_orders_account_pk,
        now_ts,
        limit,
        ctx.remaining_accounts,
    )?;
```

### 计算需锁定的用户保证金
撮合订单之后算出所需的quote token或者base token，然后根据其计算需要锁定的资产数量（base/quote），并更新用户和市场的余额。

``` Rust
    let position = &mut open_orders_account.position;
    let deposit_amount = match order.side {
        Side::Bid => {
            let free_quote = position.quote_free_native;
            let max_quote_including_fees =
                total_quote_taken_native + posted_quote_native + taker_fees + maker_fees;

            let free_qty_to_lock = cmp::min(max_quote_including_fees, free_quote);
            let deposit_amount = max_quote_including_fees - free_qty_to_lock;

            // Update market deposit total
            position.quote_free_native -= free_qty_to_lock;
            market.quote_deposit_total += deposit_amount;

            deposit_amount
        }

        Side::Ask => {
            let free_base = position.base_free_native;
            let max_base_native = total_base_taken_native + posted_base_native;

            let free_qty_to_lock = cmp::min(max_base_native, free_base);
            let deposit_amount = max_base_native - free_qty_to_lock;

            // Update market deposit total
            position.base_free_native -= free_qty_to_lock;
            market.base_deposit_total += deposit_amount;

            deposit_amount
        }
    };
```

### 事件堆惩罚计数
事件堆惩罚计数就是判断事件堆长度增加，说明有新事件（如撮合），则对用户进行惩罚计数（如惩罚性费用）。

``` Rust
if event_heap.len() > event_heap_size_before {
        position.penalty_heap_count += 1;
    }
```

### 资产转账
之前算出用户需要缴纳的保证金`deposit_amount`，然后调用 token_transfer，将用户资产转入市场金库（vault）。

``` Rust
    token_transfer(
        deposit_amount,
        &ctx.accounts.token_program,
        &ctx.accounts.user_token_account,
        &ctx.accounts.market_vault,
        &ctx.accounts.signer,
    )?;
```

完成以上逻辑之后，最后返回`order_id`，`order_id`是在`book.new_order`中生成的。

通过跟读这个方法，了解到`place_order`的流程，但是其核心的撮合交易细节在`book.new_order`中，其具体的内容放在下一章节中介绍。

更多内容可查看在github上的项目--[深入Solana OpenBook-V2源码分析与DeFi 合约实战](https://github.com/hunshenshi/solana-openbookv2-and-defi-in-action)