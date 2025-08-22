之前已经梳理了部分指令，接下来梳理下`place_orders`和`cancel_all_and_place_orders`指令，这两个指令的逻辑一样，只是指令签名不一样。先看下`place_orders`和`cancel_all_and_place_orders`指令的签名：

``` Rust
    pub fn place_orders<'c: 'info, 'info>(
        ctx: Context<'_, '_, 'c, 'info, CancelAllAndPlaceOrders<'info>>,
        orders_type: PlaceOrderType,
        bids: Vec<PlaceMultipleOrdersArgs>,
        asks: Vec<PlaceMultipleOrdersArgs>,
        limit: u8,
    ) -> Result<Vec<Option<u128>>> {
        ...
    }

    pub fn cancel_all_and_place_orders<'c: 'info, 'info>(
        ctx: Context<'_, '_, 'c, 'info, CancelAllAndPlaceOrders<'info>>,
        orders_type: PlaceOrderType,
        bids: Vec<PlaceMultipleOrdersArgs>,
        asks: Vec<PlaceMultipleOrdersArgs>,
        limit: u8,
    ) -> Result<Vec<Option<u128>>> {
        ...
    }
```

这两个指令参数都一样，只是指令名不一样，先分析下各个参数吧。

 参数包括`ctx`、`orders_type`、`bids`、`asks`和`limit`，其中`ctx`中是`CancelAllAndPlaceOrders`，包含了指令所需的账户，`orders_type`是`PlaceOrderType`类型，指明了批量订单的类型，`bids`和`asks`是`PlaceMultipleOrdersArgs`类型的`Vec`，存放的是买单和卖单的信息，`limit`是撮合限制。重点看下`CancelAllAndPlaceOrders`中的账户吧，代码如下：

``` Rust
#[derive(Accounts)]
pub struct CancelAllAndPlaceOrders<'info> {
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
        token::mint = market_quote_vault.mint
    )]
    pub user_quote_account: Account<'info, TokenAccount>,

    #[account(
        mut,
        token::mint = market_base_vault.mint
    )]
    pub user_base_account: Account<'info, TokenAccount>,

    #[account(
        mut,
        has_one = bids,
        has_one = asks,
        has_one = event_heap,
        has_one = market_base_vault,
        has_one = market_quote_vault,
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

    #[account(mut)]
    pub market_quote_vault: Box<Account<'info, TokenAccount>>,
    #[account(mut)]
    pub market_base_vault: Box<Account<'info, TokenAccount>>,

    /// CHECK: The oracle can be one of several different account types and the pubkey is checked above
    pub oracle_a: Option<UncheckedAccount<'info>>,
    /// CHECK: The oracle can be one of several different account types and the pubkey is checked above
    pub oracle_b: Option<UncheckedAccount<'info>>,

    pub token_program: Program<'info, Token>,
}
```

`place_orders`和`cancel_all_and_place_orders`指令所需账户与`place_order`类似，其中`signer`用来支付fee，`open_orders_account`用于存储挂单信息，由于同时存在买单和卖单，所以需要`user_quote_account`和`user_base_account`分别用于支付quote和base，以及market用于接收quote和base的账户`market_quote_vault`和`market_base_vault`，剩下就是一些market相关的账户，如`market`、`bids`、`asks`和`event_heap`，这些账户中的约束在之前的指令中都介绍过了，这里就不再介绍了。

`orders_type`标注这批order的类型，可以是`Market`、`ImmediateOrCancel`、`FillOrKill`和`Fixed`。`bids`和`asks`是`PlaceMultipleOrdersArgs`类型，存储了订单的lot价格和quote，其结构体如下：

``` Rust
pub struct PlaceMultipleOrdersArgs {
    pub price_lots: i64,
    pub max_quote_lots_including_fees: i64,
    pub expiry_timestamp: u64,
}
```

`place_orders`和`cancel_all_and_place_orders`指令中的逻辑如下：

``` Rust
        let n_bids = bids.len();

        let mut orders = vec![];
        for (i, order) in bids.into_iter().chain(asks).enumerate() {
            require_gte!(order.price_lots, 1, OpenBookError::InvalidInputPriceLots);

            let time_in_force = match Order::tif_from_expiry(order.expiry_timestamp) {
                Some(t) => t,
                None => {
                    msg!("Order is already expired");
                    continue;
                }
            };
            orders.push(Order {
                side: if i < n_bids { Side::Bid } else { Side::Ask },
                max_base_lots: i64::MIN, // this will be overriden to max_base_lots
                max_quote_lots_including_fees: order.max_quote_lots_including_fees,
                client_order_id: i as u64,
                time_in_force,
                self_trade_behavior: SelfTradeBehavior::CancelProvide,
                params: match orders_type {
                    PlaceOrderType::Market => OrderParams::Market,
                    PlaceOrderType::ImmediateOrCancel => OrderParams::ImmediateOrCancel {
                        price_lots: order.price_lots,
                    },
                    PlaceOrderType::FillOrKill => OrderParams::FillOrKill {
                        price_lots: order.price_lots,
                    },
                    _ => OrderParams::Fixed {
                        price_lots: order.price_lots,
                        order_type: orders_type.to_post_order_type()?,
                    },
                },
            });
        }

        #[cfg(feature = "enable-gpl")]
        return instructions::cancel_all_and_place_orders(ctx, false, orders, limit);

        #[cfg(not(feature = "enable-gpl"))]
        Ok(vec![])
```

这段逻辑较为简单，将`bids`和`asks`聚合为一个列表，然后进行遍历，遍历过程中将索引`n_bids`之前的订单设为买单bid，之后的订单设为卖单ask，生成`Order`对象放入`orders`列表中，最后调用`instructions::cancel_all_and_place_orders`，位于`src/instructions/cancel_all_and_place_orders.rs`中，方法签名如下如下：

``` Rust
pub fn cancel_all_and_place_orders<'c: 'info, 'info>(
    ctx: Context<'_, '_, 'c, 'info, CancelAllAndPlaceOrders<'info>>,
    cancel: bool,
    mut orders: Vec<Order>,
    limit: u8,
) -> Result<Vec<Option<u128>>> {
    ...
    Ok(order_ids)
}
```

这个方法的逻辑分为三部分，首先是**取消所有挂单**、**遍历创建新订单**和**token转移**。

### 取消所有挂单
挂单信息都存储在`open_orders_account`中，所以首先需要从`ctx`中获取`open_orders_account`账户以及对应的订单树`book`，最后调用订单树`book`的`book.cancel_all_orders`方法将该用户的旧单取消掉，取消时不止从订单树`book`中取消，还需要从`open_orders_account`中把订单取消掉。代码如下：

``` Rust
    let mut open_orders_account = ctx.accounts.open_orders_account.load_mut()?;
    let open_orders_account_pk = ctx.accounts.open_orders_account.key();

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
...
    if cancel {
        book.cancel_all_orders(&mut open_orders_account, *market, u8::MAX, None, None)?;
    }
```

### 遍历创建新订单
旧单取消之后，对`orders`进行遍历，然后调用`book.new_order`生成新的订单，代码如下：

``` Rust
    let now_ts: u64 = clock.unix_timestamp.try_into().unwrap();

    let oracle_price_lots = market.oracle_price_lots(
        AccountInfoRef::borrow_some(ctx.accounts.oracle_a.as_ref())?.as_ref(),
        AccountInfoRef::borrow_some(ctx.accounts.oracle_b.as_ref())?.as_ref(),
        clock.slot,
    )?;
    ...
    let mut base_amount = 0_u64;
    let mut quote_amount = 0_u64;
    let mut order_ids = Vec::new();
    for order in orders.iter_mut() {
        order.max_base_lots = market.max_base_lots();
        require_gte!(
            order.max_quote_lots_including_fees,
            0,
            OpenBookError::InvalidInputLots
        );

        match order.side {
            Side::Ask => {
                let max_available_base = ctx.accounts.user_base_account.amount
                    + open_orders_account.position.base_free_native
                    - base_amount;
                order.max_base_lots = std::cmp::min(
                    order.max_base_lots,
                    market.max_base_lots_from_lamports(max_available_base),
                );
            }
            Side::Bid => {
                let max_available_quote = ctx.accounts.user_quote_account.amount
                    + open_orders_account.position.quote_free_native
                    - quote_amount;
                order.max_quote_lots_including_fees = std::cmp::min(
                    order.max_quote_lots_including_fees,
                    market.max_quote_lots_from_lamports(max_available_quote),
                );
            }
        }

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
            order,
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

        match order.side {
            Side::Bid => {
                quote_amount = quote_amount
                    .checked_add(
                        total_quote_taken_native + posted_quote_native + taker_fees + maker_fees,
                    )
                    .ok_or(OpenBookError::InvalidInputOrdersAmounts)?;
            }
            Side::Ask => {
                base_amount = base_amount
                    .checked_add(total_base_taken_native + posted_base_native)
                    .ok_or(OpenBookError::InvalidInputOrdersAmounts)?;
            }
        };

        order_ids.push(order_id);
    }
```

### token转移
撮合完成之后，根据撮合的token数量决定需要向market转入多少数量的token。相关代码如下：

``` Rust
    let mut event_heap = ctx.accounts.event_heap.load_mut()?;
    let event_heap_size_before = event_heap.len();
	...
    let position = &mut open_orders_account.position;

    let free_base_to_lock = cmp::min(base_amount, position.base_free_native);
    let free_quote_to_lock = cmp::min(quote_amount, position.quote_free_native);

    let deposit_base_amount = base_amount - free_base_to_lock;
    let deposit_quote_amount = quote_amount - free_quote_to_lock;

    position.base_free_native -= free_base_to_lock;
    position.quote_free_native -= free_quote_to_lock;

    market.base_deposit_total += deposit_base_amount;
    market.quote_deposit_total += deposit_quote_amount;

    if event_heap.len() > event_heap_size_before {
        position.penalty_heap_count += 1;
    }

    token_transfer(
        deposit_quote_amount,
        &ctx.accounts.token_program,
        &ctx.accounts.user_quote_account,
        &ctx.accounts.market_quote_vault,
        &ctx.accounts.signer,
    )?;
    token_transfer(
        deposit_base_amount,
        &ctx.accounts.token_program,
        &ctx.accounts.user_base_account,
        &ctx.accounts.market_base_vault,
        &ctx.accounts.signer,
    )?;
```

具体的计算逻辑是从撮合的token数量`base_amount`和`quote_amount`中减去`open_orders_account.position`中free的token数量，得出的`deposit_base_amount`和`deposit_quote_amount`就是用户需要转入market中的token数量，同时还需要改变相应的状态值，比如`position`中的free数量和market中的deposit数量。

> 这里需要再次记忆的是撮合订单`new_order`中不涉及token的转移，只是一些记账操作，撮合订单之后，统一向market中转移token或者从market中转出token。

更多内容可查看在github上的项目--[深入Solana OpenBook-V2源码分析与DeFi 合约实战](https://github.com/hunshenshi/solana-openbookv2-and-defi-in-action)