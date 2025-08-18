前面的内容已将`place_order`挂单指令的主要流程梳理完毕，现在梳理下与其相似的指令`place_take_order`，这个指令用于立即撮合并结算（taker）订单，不挂单、不留在订单簿上，适合做市商、套利、用户一键买卖等场景。

其入口函数依然在`lib.rs`中，代码如下：

``` Rust
    pub fn place_take_order<'c: 'info, 'info>(
        ctx: Context<'_, '_, 'c, 'info, PlaceTakeOrder<'info>>,
        args: PlaceTakeOrderArgs,
    ) -> Result<()> {
        require_gte!(args.price_lots, 1, OpenBookError::InvalidInputPriceLots);

        let order = Order {
            side: args.side,
            max_base_lots: args.max_base_lots,
            max_quote_lots_including_fees: args.max_quote_lots_including_fees,
            client_order_id: 0,
            time_in_force: 0,
            self_trade_behavior: SelfTradeBehavior::default(),
            params: match args.order_type {
                PlaceOrderType::Market => OrderParams::Market,
                PlaceOrderType::ImmediateOrCancel => OrderParams::ImmediateOrCancel {
                    price_lots: args.price_lots,
                },
                PlaceOrderType::FillOrKill => OrderParams::FillOrKill {
                    price_lots: args.price_lots,
                },
                _ => return Err(OpenBookError::InvalidInputOrderType.into()),
            },
        };

        #[cfg(feature = "enable-gpl")]
        instructions::place_take_order(ctx, order, args.limit)?;
        Ok(())
    }
```

同样是构造了一个`Order`对象，然后调用`instructions/place_take_order.rs`中的`place_take_order`指令，只是由于场景不一样，`Order`对象的构建参数有些不同。

`lib::place_take_order`的参数是`PlaceTakeOrder`和`PlaceTakeOrderArgs`，`PlaceTakeOrder`是`place_take_order`指令所需的所有账户，该参数直传给了`instructions::place_take_order`，`PlaceTakeOrderArgs`就是将所需参数封装成的一个结构体，然后根据这些内容构建一个`Order`对象，再传给`instructions::place_take_order`。

## `place_take_order`核心逻辑
接下来逐步分析`instructions::place_take_order`，代码如下：

``` Rust
pub fn place_take_order<'c: 'info, 'info>(
    ctx: Context<'_, '_, 'c, 'info, PlaceTakeOrder<'info>>,
    order: Order,
    limit: u8,
) -> Result<()> {
	...
    Ok(())
}
```

此方法与`place_order`指令参数类似，只是该指令所需账户不一样，其封装在`PlaceTakeOrder`中，该代码在`accounts_ix/place_take_order.rs`，根据代码理解其下单时都需要哪些账户。代码如下：

``` Rust
#[derive(Accounts)]
pub struct PlaceTakeOrder<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,
    #[account(mut)]
    pub penalty_payer: Signer<'info>,

    #[account(
        mut,
        has_one = bids,
        has_one = asks,
        has_one = event_heap,
        has_one = market_base_vault,
        has_one = market_quote_vault,
        has_one = market_authority,
        constraint = market.load()?.oracle_a == oracle_a.non_zero_key(),
        constraint = market.load()?.oracle_b == oracle_b.non_zero_key(),
        constraint = market.load()?.open_orders_admin == open_orders_admin.non_zero_key() @ OpenBookError::InvalidOpenOrdersAdmin
    )]
    pub market: AccountLoader<'info, Market>,
    /// CHECK: checked on has_one in market
    pub market_authority: UncheckedAccount<'info>,
    #[account(mut)]
    pub bids: AccountLoader<'info, BookSide>,
    #[account(mut)]
    pub asks: AccountLoader<'info, BookSide>,
    #[account(mut)]
    pub market_base_vault: Account<'info, TokenAccount>,
    #[account(mut)]
    pub market_quote_vault: Account<'info, TokenAccount>,
    #[account(mut)]
    pub event_heap: AccountLoader<'info, EventHeap>,

    #[account(
        mut,
        token::mint = market_base_vault.mint
    )]
    pub user_base_account: Box<Account<'info, TokenAccount>>,
    #[account(
        mut,
        token::mint = market_quote_vault.mint
    )]
    pub user_quote_account: Box<Account<'info, TokenAccount>>,

    /// CHECK: The oracle can be one of several different account types and the pubkey is checked above
    pub oracle_a: Option<UncheckedAccount<'info>>,
    /// CHECK: The oracle can be one of several different account types and the pubkey is checked above
    pub oracle_b: Option<UncheckedAccount<'info>>,

    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub open_orders_admin: Option<Signer<'info>>,
}
```

`penalty_payer`是`Signer`类型，用于支付`event_heap`中的事件费用。
`market`、`market_authority`、`bids`、`asks`、`market_base_vault`、`market_quote_vault`和`event_heap`都是`market`相关的，其中`market_base_vault`和`market_quote_vault`是`base`和`quote`在市场中的资金池。
`user_base_account`和`user_quote_account`是用户的资金账户，其中的`token::mint`约束与市场中的`base`和`quote`的`mint`一样。
`oracle_a`和`oracle_b`分别为外部预言机账户，用于一些市场限价控制或者风控。
这里增加了`system_program`，是因为在`place_take_order`指令中有些系统程序调用。

参数梳理完了，现在梳理下下单的内部逻辑，其流程与`place_order`类似，这里将整块逻辑划分五块进行拆分，主要分为**市场状态加载**、**撮合订单**、**计算用户转入和提取**、**事件堆惩罚**和**资产转账**。

### 市场状态加载
市场状态加载代码如下：

``` Rust
    let clock = Clock::get()?;

    let mut market = ctx.accounts.market.load_mut()?;
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

首先加载market账户和撮合交易时需要用到的订单树bids/asks账户，账户加载之后，还需要一些校验以保证代码的合法性，最后加载event_heap和oracle_price_lots，event_heap主要用于存放事件，然后根据事件堆的长度判断是否需要惩罚。

### 撮合订单
撮合订单调用`book.new_order`进行撮合和挂单，返回订单执行结果（成交量、返佣和taker费用），与`place_order`中的`new_order`不同的是，这里不需要挂单量，因为这个撮合订单时不需要挂单。代码如下：

``` Rust
    let OrderWithAmounts {
        total_base_taken_native,
        total_quote_taken_native,
        referrer_amount,
        taker_fees,
        ..
    } = book.new_order(
        &order,
        &mut market,
        &ctx.accounts.market.key(),
        &mut event_heap,
        oracle_price_lots,
        None,
        &ctx.accounts.signer.key(),
        now_ts,
        limit,
        ctx.remaining_accounts,
    )?;
```

### 计算用户转入和提取
撮合订单之后会返回撮合的base和quote总量，分别是`total_base_taken_native`和`total_quote_taken_native`，还会返回返佣`referrer_amount`和take费用`taker_fees`，利用这些费用就可以算出需要转入和提取。代码如下：

``` Rust
    let makers_rebates = taker_fees - referrer_amount;

    let (deposit_amount, withdraw_amount) = match side {
        Side::Bid => {
            let total_quote_including_fees = total_quote_taken_native + makers_rebates;
            market.base_deposit_total -= total_base_taken_native;
            market.quote_deposit_total += total_quote_including_fees;
            (total_quote_including_fees, total_base_taken_native)
        }
        Side::Ask => {
            let total_quote_discounting_fees = total_quote_taken_native - makers_rebates;
            market.base_deposit_total += total_base_taken_native;
            market.quote_deposit_total -= total_quote_discounting_fees;
            (total_base_taken_native, total_quote_discounting_fees)
        }
    };
```

taker订单需要收取手续费，所以转入或者提取的quote需要包含此费用。

### 事件堆惩罚
如果在撮合订单的过程中向事件堆中增加事件，则需要支付相应的惩罚费用，之所以要支付费用是为了激励用户及时消费事件堆中的事件，避免事件堆无限增长。

``` Rust
    if event_heap.len() > event_heap_size_before {
        system_program_transfer(
            PENALTY_EVENT_HEAP,
            &ctx.accounts.system_program,
            &ctx.accounts.penalty_payer,
            &ctx.accounts.market,
        )?;
    }
```

这个支付的惩罚费是原生token，所以调用的是`system_program_transfer`。

### 资产转账
在**计算用户转入和提取**的代码逻辑中计算出需要转入和转出的数量，现在对其进行转账操作。相关代码如下：

``` Rust
	let seeds = market_seeds!(market, ctx.accounts.market.key());
...
    let (user_deposit_acc, user_withdraw_acc, market_deposit_acc, market_withdraw_acc) = match side
    {
        Side::Bid => (
            &ctx.accounts.user_quote_account,
            &ctx.accounts.user_base_account,
            &ctx.accounts.market_quote_vault,
            &ctx.accounts.market_base_vault,
        ),
        Side::Ask => (
            &ctx.accounts.user_base_account,
            &ctx.accounts.user_quote_account,
            &ctx.accounts.market_base_vault,
            &ctx.accounts.market_quote_vault,
        ),
    };

    token_transfer(
        deposit_amount,
        &ctx.accounts.token_program,
        user_deposit_acc.as_ref(),
        market_deposit_acc,
        &ctx.accounts.signer,
    )?;

    token_transfer_signed(
        withdraw_amount,
        &ctx.accounts.token_program,
        market_withdraw_acc,
        user_withdraw_acc.as_ref(),
        &ctx.accounts.market_authority,
        seeds,
    )?;
```

由于token交换，需要market的签名，所以这里需要先通过`market_seeds!`拿到market的`seeds`，然后调用`token_transfer_signed`将market中的token transfer到taker发起者账户中，而taker发起者账户向market转账时需调用`token_transfer`向market进行transfer。

通过跟读这个方法，了解到`place_take_order`的流程，也了解到了`place_order`与`place_take_order`的区别，同样其核心的撮合交易细节在`book.new_order`中，这部分内容已在上一篇中介绍过了，感兴趣的可以看下。

更多内容可查看在github上的项目--[深入Solana OpenBook-V2源码分析与DeFi 合约实战](https://github.com/hunshenshi/solana-openbookv2-and-defi-in-action)