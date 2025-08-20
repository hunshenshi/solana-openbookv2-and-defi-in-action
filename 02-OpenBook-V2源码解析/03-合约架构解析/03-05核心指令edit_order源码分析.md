通过`place_order`挂单之后，如果想修改订单用什么指令呢？需要使用`edit_order`指令，入口函数在`lib.rs`中，代码如下：

``` Rust
    pub fn edit_order<'c: 'info, 'info>(
        ctx: Context<'_, '_, 'c, 'info, PlaceOrder<'info>>,
        client_order_id: u64,
        expected_cancel_size: i64,
        place_order: PlaceOrderArgs,
    ) -> Result<Option<u128>> {
        require_gte!(
            place_order.price_lots,
            1,
            OpenBookError::InvalidInputPriceLots
        );

        let time_in_force = match Order::tif_from_expiry(place_order.expiry_timestamp) {
            Some(t) => t,
            None => {
                msg!("Order is already expired");
                return Ok(None);
            }
        };
        let order = Order {
            side: place_order.side,
            max_base_lots: place_order.max_base_lots,
            max_quote_lots_including_fees: place_order.max_quote_lots_including_fees,
            client_order_id: place_order.client_order_id,
            time_in_force,
            self_trade_behavior: place_order.self_trade_behavior,
            params: match place_order.order_type {
                PlaceOrderType::Market => OrderParams::Market,
                PlaceOrderType::ImmediateOrCancel => OrderParams::ImmediateOrCancel {
                    price_lots: place_order.price_lots,
                },
                PlaceOrderType::FillOrKill => OrderParams::FillOrKill {
                    price_lots: place_order.price_lots,
                },
                _ => OrderParams::Fixed {
                    price_lots: place_order.price_lots,
                    order_type: place_order.order_type.to_post_order_type()?,
                },
            },
        };
        #[cfg(feature = "enable-gpl")]
        return instructions::edit_order(
            ctx,
            client_order_id,
            expected_cancel_size,
            order,
            place_order.limit,
        );

        #[cfg(not(feature = "enable-gpl"))]
        Ok(None)
    }
```

代码逻辑比较简单，同样是构造了一个`Order`对象，然后调用`instructions/edit_order.rs`中的`edit_order`指令。

`lib::edit_order`与`place_order`相似，参数都包括`PlaceOrder`和`PlaceOrderArgs`，`PlaceOrder`是`edit_order`指令所需的所有账户，该参数直传给了`instructions::edit_order`，`PlaceOrderArgs`就是将所有order相关的参数封装成的一个结构体，然后根据这些内容构建一个`Order`对象，再传给`instructions::edit_order`，而`edit_order`还包括`client_order_id`和`expected_cancel_size`，`client_order_id`是用来定位要取消挂单，`expected_cancel_size`是客户端期望在挂单中取消的token数量，必须非负。

`lib::edit_order`中的参数几乎都传入了`instructions/edit_order.rs`中的`edit_order`指令，那就看下`instructions/edit_order.rs`中的具体代码：

``` Rust
pub fn edit_order<'c: 'info, 'info>(
    ctx: Context<'_, '_, 'c, 'info, PlaceOrder<'info>>,
    cancel_client_order_id: u64,
    expected_cancel_size: i64,
    mut order: Order,
    limit: u8,
) -> Result<Option<u128>> {
    require_gte!(
        expected_cancel_size,
        0,
        OpenBookError::InvalidInputCancelSize
    );

    let leaf_node_quantity = crate::instructions::cancel_order_by_client_order_id(
        Context::new(
            ctx.program_id,
            &mut ctx.accounts.to_cancel_order(),
            ctx.remaining_accounts,
            ctx.bumps.to_cancel_order(),
        ),
        cancel_client_order_id,
    )?;

    let filled_amount = expected_cancel_size - leaf_node_quantity;
    // note that order.max_base_lots is checked to be > 0 inside `place_order`
    if filled_amount > 0 && order.max_base_lots > filled_amount {
        // Do not reduce max_quote_lots_including_fees as implicitly it's limited by max_base_lots.
        order.max_base_lots -= filled_amount;
        return crate::instructions::place_order(ctx, order, limit);
    }
    Ok(None)
}
```

首先检查`expected_cancel_size`是否为非负，然后根据`cancel_client_order_id`取消旧单，最后根据`expected_cancel_size`决定是否下新单，如果下新单的话调用的还是`instructions::place_order`方法。

接下来看下如何根据`cancel_client_order_id`取消订单，调用的方法是`instructions::cancel_order_by_client_order_id`，在`instructions/cancel_order_by_client_order_id.rs`中，代码如下：

``` Rust
pub fn cancel_order_by_client_order_id(
    ctx: Context<CancelOrder>,
    client_order_id: u64,
) -> Result<i64> {
    let mut account = ctx.accounts.open_orders_account.load_mut()?;

    let market = ctx.accounts.market.load()?;
    let mut book = Orderbook {
        bids: ctx.accounts.bids.load_mut()?,
        asks: ctx.accounts.asks.load_mut()?,
    };

    book.cancel_all_orders(&mut account, *market, u8::MAX, None, Some(client_order_id))
}
```

`cancel_order_by_client_order_id`这里只是一些参数准备，取消订单的核心逻辑在`book.cancel_all_orders`中，代码如下：

``` Rust
    pub fn cancel_all_orders(
        &mut self,
        open_orders_account: &mut OpenOrdersAccount,
        market: Market,
        mut limit: u8,
        side_to_cancel_option: Option<Side>,
        client_id_option: Option<u64>,
    ) -> Result<i64> {
        let mut total_quantity = 0_i64;
        for i in 0..MAX_OPEN_ORDERS {
            let oo = open_orders_account.open_orders[i];
            if oo.is_free() {
                continue;
            }
            ...
            if let Some(client_id) = client_id_option {
                if client_id != oo.client_id {
                    continue;
                }
            }
            ...
            let order_id = oo.id;

            let cancel_result = self.cancel_order(
                open_orders_account,
                order_id,
                order_side_and_tree,
                market,
                None,
            );
            if cancel_result.is_anchor_error_with_code(OpenBookError::OrderIdNotFound.into()) {
                msg!(
                    "order {} was not found on orderbook, expired or filled already",
                    order_id
                );
            } else {
                total_quantity += cancel_result?.quantity;
            }

            limit -= 1;
        }
        Ok(total_quantity)
    }
```

主要流程是根据`client_order_id`遍历`open_orders_account`中的订单，找到`order_id`，然后从订单树中移除该旧单，再从`open_orders_account`中移除该旧单信息。

`cancel_order_by_client_order_id`会返回取消订单token的总量`leaf_node_quantity`，然后根据`expected_cancel_size`决定是否挂新单，具体规则为：
- 当`expected_cancel_size`大于`leaf_node_quantity`时，则挂一个新单，新单数量需要减去两者的差值`filled_amount`
- 否则不挂新单，只取消旧单

更多内容可查看在github上的项目--[深入Solana OpenBook-V2源码分析与DeFi 合约实战](https://github.com/hunshenshi/solana-openbookv2-and-defi-in-action)