上一章节梳理了`instructions::place_order`中的代码逻辑，其中核心的撮合交易并没有展开，本篇文章将对其进行展开介绍。在`instructions::place_order`中的代码如下：

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

其返回值`total_base_taken_native`表示本次撮合**实际成交**的 base 资产数量（以 native 单位计），`total_quote_taken_native`表示本次撮合**实际成交**的 quote 资产数量（以 native 单位计），`posted_base_native`表示本次下单后，**实际挂在**订单簿上的 base 资产数量（以 native 单位计），`posted_quote_native`表示本次下单后，**实际挂在**订单簿上的 quote 资产数量（以 native 单位计），`taker_fees`表示本次撮合中，taker（吃单方）需要支付的手续费，`maker_fees`表示本次撮合中，maker（挂单方）需要支付的手续费。

`book.new_order`在`state/orderbook/book.rs`中，是`Orderbook`的一个公开方法，`Orderbook`是订单簿，由买单树和卖单树组成，代码如下：

``` Rust
pub struct Orderbook<'a> {
    pub bids: RefMut<'a, BookSide>,
    pub asks: RefMut<'a, BookSide>,
}
```

`new_order`又是一个奇长无比的方法，照旧对它进行分块介绍，主要分为**数据准备**、**订单撮合**、**统计撮合结果**、**计算相关费用**、**更新订单树**和**挂单**。

### 数据准备
这一部分主要做一些数据准备工作，比如订单方向、对手方向、最大base lot、最大quote lot以及order id等等，下面详细介绍每个变量的作用。

``` Rust
        let market = open_book_market;

        let side = order.side;

        let other_side = side.invert_side();
        let post_only = order.is_post_only();
        let fill_or_kill = order.is_fill_or_kill();
        let mut post_target = order.post_target();
        let (price_lots, price_data) = order.price(now_ts, oracle_price_lots, self)?;

        // generate new order id
        let order_id = market.gen_order_id(side, price_data);

        let order_max_base_lots = order.max_base_lots;
        let order_max_quote_lots = if side == Side::Bid && !post_only {
market.subtract_taker_fees(order.max_quote_lots_including_fees)
        } else {
            order.max_quote_lots_including_fees
        };

        require_gte!(
            market.max_base_lots(),
            order_max_base_lots,
            OpenBookError::InvalidInputLotsSize
        );

        require_gte!(
            market.max_quote_lots(),
            order_max_quote_lots,
            OpenBookError::InvalidInputLotsSize
        );

        let mut remaining_base_lots = order_max_base_lots;
        let mut remaining_quote_lots = order_max_quote_lots;
        let mut decremented_quote_lots = 0_i64;

        let mut referrer_amount = 0_u64;
        let mut maker_rebates_acc = 0_u64;

        let mut matched_order_changes: Vec<(BookSideOrderHandle, i64)> = vec![];
        let mut matched_order_deletes: Vec<(BookSideOrderTree, u128)> = vec![];
        let mut number_of_dropped_expired_orders = 0;
        let mut number_of_processed_fill_events = 0;
```

- `market`是订单所在的market
- `side`是订单的方向，`other_side`是对手单的方向
- `post_only`标识该订单是否为**只挂单**
- `fill_or_kill`标识该订单是否为完全执行或者取消
- `post_target`标识该订单会挂在哪个订单树上
- `price_lots`是该订单的目标价格，即实际撮合价格，用于成交判断和资金结算，而`price_data`也是该订单目标价格的一种体现，主要用于订单ID `order_id`的生成，用于高效查找、排序、支持特殊订单类型
- `order_max_base_lots`表示最多base的lots数，`order_max_quote_lots`是最多quote的lots数，不过如果是买单并且不是只挂单(`post_only`为false)的话，`order_max_quote_lots`的值需要减去手续费fee
- `remaining_base_lots`和`remaining_quote_lots`分别表示在撮合交易中还需要撮合的base和quote数量
- `decremented_quote_lots`表示不收取taker手续费的quote数量
- `referrer_amount`表示推荐人获得的返佣数
- `maker_rebates_acc`表示maker获得的返利数，
- `matched_order_changes`和`matched_order_deletes`中存储的是在撮合过程中需要调整的挂单
- `number_of_dropped_expired_orders`和`number_of_processed_fill_events`是两个计数值，表示在撮合过程中记录的过期需要删除的订单数和执行的撮合数。

### 订单撮合
数据准备好之后，就开始遍历订单簿进行订单撮合了，主要逻辑就是在for循环中遍历**整个订单树**，根据对手单匹配`price_lots`，然后根据`remaining_base_lots`或`remaining_quote_lots`等条件来决定是否退出循环，代码如下：

``` Rust
        let opposing_bookside = self.bookside_mut(other_side);
        for best_opposing in opposing_bookside.iter_all_including_invalid(now_ts, oracle_price_lots)
        {
            if remaining_base_lots == 0 || remaining_quote_lots == 0 {
                break;
            }

            if !best_opposing.is_valid() {
                // Remove the order from the book unless we've done that enough
                if number_of_dropped_expired_orders < DROP_EXPIRED_ORDER_LIMIT {
                    number_of_dropped_expired_orders += 1;
                    let event = OutEvent::new(
                        other_side,
                        best_opposing.node.owner_slot,
                        now_ts,
                        event_heap.header.seq_num,
                        best_opposing.node.owner,
                        best_opposing.node.quantity,
                    );

                    process_out_event(
                        event,
                        market,
                        event_heap,
                        open_orders_account.as_deref_mut(),
                        owner,
                        remaining_accs,
                    )?;
                    matched_order_deletes
                        .push((best_opposing.handle.order_tree, best_opposing.node.key));
                }
                continue;
            }

            let best_opposing_price = best_opposing.price_lots;

            if !side.is_price_within_limit(best_opposing_price, price_lots) {
                break;
            }
            if post_only {
                msg!("Order could not be placed due to PostOnly");
                post_target = None;
                break; // return silently to not fail other instructions in tx
            }
            if limit == 0 {
                msg!("Order matching limit reached");
                post_target = None;
                break;
            }

            let max_match_by_quote = remaining_quote_lots / best_opposing_price;
            // Do not post orders in the book due to bad pricing and negative spread
            if max_match_by_quote == 0 {
                post_target = None;
                break;
            }

            let match_base_lots = remaining_base_lots
                .min(best_opposing.node.quantity)
                .min(max_match_by_quote);
            let match_quote_lots = match_base_lots * best_opposing_price;

            // Self-trade behaviour
            if open_orders_account.is_some() && owner == &best_opposing.node.owner {
                match order.self_trade_behavior {
                    SelfTradeBehavior::DecrementTake => {
                        // remember all decremented quote lots to only charge fees on not-self-trades
                        decremented_quote_lots += match_quote_lots;
                    }
                    SelfTradeBehavior::CancelProvide => {
                        // The open orders acc is always present in this case, no need event_heap
                        open_orders_account.as_mut().unwrap().cancel_order(
                            best_opposing.node.owner_slot as usize,
                            best_opposing.node.quantity,
                            *market,
                        );
                        matched_order_deletes
                            .push((best_opposing.handle.order_tree, best_opposing.node.key));

                        // skip actual matching
                        continue;
                    }
                    SelfTradeBehavior::AbortTransaction => {
                        return err!(OpenBookError::WouldSelfTrade)
                    }
                }
                assert!(order.self_trade_behavior == SelfTradeBehavior::DecrementTake);
            } else {
                maker_rebates_acc +=
                    market.maker_rebate_floor((match_quote_lots * market.quote_lot_size) as u64);
            }

            remaining_base_lots -= match_base_lots;
            remaining_quote_lots -= match_quote_lots;
            assert!(remaining_quote_lots >= 0);

            let new_best_opposing_quantity = best_opposing.node.quantity - match_base_lots;
            let maker_out = new_best_opposing_quantity == 0;
            if maker_out {
                matched_order_deletes
                    .push((best_opposing.handle.order_tree, best_opposing.node.key));
            } else {
                matched_order_changes.push((best_opposing.handle, new_best_opposing_quantity));
            }

            let fill = FillEvent::new(
                side,
                maker_out,
                best_opposing.node.owner_slot,
                now_ts,
                market.seq_num,
                best_opposing.node.owner,
                best_opposing.node.client_order_id,
                best_opposing.node.timestamp,
                *owner,
                order.client_order_id,
                best_opposing_price,
                best_opposing.node.peg_limit,
                match_base_lots,
            );

            emit_stack(TakerSignatureLog {
                market: *market_pk,
                seq_num: market.seq_num,
            });

            process_fill_event(
                fill,
                market,
                event_heap,
                remaining_accs,
                &mut number_of_processed_fill_events,
            )?;

            limit -= 1;
        }
```

在撮合订单的过程中，还掺杂些辅助逻辑，接下来就来探究下具体的逻辑。

**首先**，由于这个`iterator`中包含`invalid`订单，所以会首先判断下**挂单**是否有效`if !best_opposing.is_valid()`，无效的话则将其放入`matched_order_deletes`中，并生成一个**出对事件(OutEvent)**，执行`process_out_event`对挂单进行处理。

**然后**，根据遍历到的有效挂单与`price_lots`进行匹配，如果是买单Bid的话，对手单低于`price_lots`即符合匹配条件，如果是卖单ask的话，对手单高于`price_lots`即符合匹配条件，判断是否匹配的代码如下：

``` Rust
    pub fn is_price_within_limit(self: &Side, price: i64, limit: i64) -> bool {
        match self {
            Side::Bid => price <= limit,
            Side::Ask => price >= limit,
        }
    }
```

**接下来**，匹配到挂单之后，计算出可以匹配的`match_base_lots`和`match_quote_lots`，以及剩余待匹配的`remaining_base_lots`和`remaining_quote_lots`。

**随后**，根据此次匹配的`match_base_lots`计算出该挂单是否还剩`new_best_opposing_quantity`，如果为0，则将该挂单放入`matched_order_deletes`中，随后进行删除，如果不为0，则放入`matched_order_changes`中，随后更新最新的挂单状态。

**最后**，生成一个**执行事件(FillEvent)**，执行`process_fill_event`对`maker`挂单进行处理，更新`maker`的持仓状态。

其中还有一段逻辑没有说明，那就是需要判断是否为**自交易**，当`open_orders_account.is_some() && owner == &best_opposing.node.owner`时，执行的撮合为自交易类型，根据下单时选择的策略决定是否继续交易，如果继续交易的话`decremented_quote_lots`中记录匹配的quote lots数，这一部分不收取`taker`费用。

### 统计撮合结果
遍历结束之后，计算出一共撮合的base和quote数量

``` Rust
        let total_quote_lots_taken = order_max_quote_lots - remaining_quote_lots;
        let total_base_lots_taken = order.max_base_lots - remaining_base_lots;
        assert!(total_quote_lots_taken >= 0);
        assert!(total_base_lots_taken >= 0);

        let total_base_taken_native = (total_base_lots_taken * market.base_lot_size) as u64;
        let total_quote_taken_native = (total_quote_lots_taken * market.quote_lot_size) as u64;
```

### 计算相关费用
相关费用包括taker费用、返佣referrer费用、市场累计的手续费总额，以及不存在open_orders_account时统计的taker_volume。代码如下：

``` Rust
        let mut taker_fees_native = 0_u64;
        if total_quote_lots_taken > 0 || total_base_lots_taken > 0 {
            let total_quote_taken_native_wo_self =
                ((total_quote_lots_taken - decremented_quote_lots) * market.quote_lot_size) as u64;

            if total_quote_taken_native_wo_self > 0 {
                taker_fees_native = market.taker_fees_ceil(total_quote_taken_native_wo_self);

                // Only account taker fees now. Maker fees accounted once processing the event
                referrer_amount = taker_fees_native - maker_rebates_acc;
                market.fees_accrued += referrer_amount as u128;
            };

            if let Some(open_orders_account) = &mut open_orders_account {
                open_orders_account.execute_taker(
                    market,
                    side,
                    total_base_taken_native,
                    total_quote_taken_native,
                    taker_fees_native,
                    referrer_amount,
                );
            } else {
                market.taker_volume_wo_oo += total_quote_taken_native as u128;
            }

            let (total_quantity_paid, total_quantity_received) = match side {
                Side::Bid => (
                    total_quote_taken_native + taker_fees_native,
                    total_base_taken_native,
                ),
                Side::Ask => (
                    total_base_taken_native,
                    total_quote_taken_native - taker_fees_native,
                ),
            };

            emit_stack(TotalOrderFillEvent {
                side: side.into(),
                taker: *owner,
                total_quantity_paid,
                total_quantity_received,
                fees: taker_fees_native,
            });
        }
```

* 计算taker费用时需要减去`decremented_quote_lots`，这部分是自交易的数据，需要扣除，算出`taker_fees_native`
* 返佣数`referrer_amount`就是`taker_fees_native`中减去maker返利`maker_rebates_acc`的值
* 市场累计的可分配的手续费`fees_accrued`是referrer_amount的累积，这部分可分配给推荐人或市场(如果没有推荐人就分配给市场)
* 如果存在`open_orders_account`则更新taker的持仓，否则统计市场中的`taker_volume_wo_oo`，这个记录的是没有 open_orders_account 的 taker 成交量，保证市场统计完整。
* 算出需要支付的`total_quantity_paid`和接收的`total_quantity_received`数量

### 更新订单树
在撮合订单中，需要变更的挂单会更新到`matched_order_changes`和`matched_order_deletes`中，现在对订单树进行更新

``` Rust
        for (handle, new_quantity) in matched_order_changes {
            opposing_bookside
                .node_mut(handle.node)
                .unwrap()
                .as_leaf_mut()
                .unwrap()
                .quantity = new_quantity;
        }
        for (component, key) in matched_order_deletes {
            let _removed_leaf = opposing_bookside.remove_by_key(component, key).unwrap();
        }
```

### 挂单
最后撮合结束之后，还需要计算是否有还有剩余`remaining_quote_lots`，如果有，则进行挂单操作。

**首先**，看下`remaining_quote_lots`的计算方式，代码如下：

``` Rust
        let taker_fees_lots =
            (taker_fees_native as i64 + market.quote_lot_size - 1) / market.quote_lot_size;

        remaining_quote_lots =
            order.max_quote_lots_including_fees - total_quote_lots_taken - taker_fees_lots;
```

先对`taker_fees_native`进行向上取整算出lots数，然后从`max_quote_lots_including_fees`减去撮合的`total_quote_lots_taken`和手续费`taker_fees_lots`，就是剩下的`remaining_quote_lots`。

**然后**，根据`remaining_quote_lots`计算所需的base数`book_base_quantity_lots`，代码如下：

``` Rust
        let book_base_quantity_lots = {
            remaining_quote_lots -= market.maker_fees_ceil(remaining_quote_lots);
            remaining_base_lots.min(remaining_quote_lots / price)
        };
```

从`remaining_quote_lots`中减去maker的费用(一般市场的maker费用都是负或者零，为了鼓励流动性)，再除以`price`。

**接下来**，就可以挂单了，但是还是要先检查下订单是否为`fill_or_kill`，如果是则不挂单，返回`error`

**最后**，就是生成新的订单，挂到对应的订单树上。挂单之前还会对订单树进行一些常规操作，包括清除过期的挂单和如果满了就删除一些挂单。相关代码如下：

``` Rust
        if let Some(order_tree_target) = post_target {
            require_gte!(
                market.max_quote_lots(),
                book_base_quantity_lots * price,
                OpenBookError::InvalidPostAmount
            );

            posted_base_native = book_base_quantity_lots * market.base_lot_size;
            posted_quote_native = book_base_quantity_lots * price * market.quote_lot_size;

            // Open orders always exists in this case
            let open_orders = open_orders_account.as_mut().unwrap();

            // Subtract maker fees in bid.
            if side == Side::Bid {
                maker_fees_native = market
                    .maker_fees_ceil(posted_quote_native)
                    .try_into()
                    .unwrap();

                open_orders.position.locked_maker_fees += maker_fees_native;
            }

            let bookside = self.bookside_mut(side);
            // Drop an expired order if possible
            if let Some(expired_order) = bookside.remove_one_expired(order_tree_target, now_ts) {
                let event = OutEvent::new(
                    side,
                    expired_order.owner_slot,
                    now_ts,
                    event_heap.header.seq_num,
                    expired_order.owner,
                    expired_order.quantity,
                );
                process_out_event(
                    event,
                    market,
                    event_heap,
                    Some(open_orders),
                    owner,
                    remaining_accs,
                )?;
            }

            if bookside.is_full() {
                // If this bid is higher than lowest bid, boot that bid and insert this one
                let (worst_order, worst_price) =
                    bookside.remove_worst(now_ts, oracle_price_lots).unwrap();
                // OpenBookErrorCode::OutOfSpace
                require!(
                    side.is_price_better(price_lots, worst_price),
                    OpenBookError::SomeError
                );
                let event = OutEvent::new(
                    side,
                    worst_order.owner_slot,
                    now_ts,
                    event_heap.header.seq_num,
                    worst_order.owner,
                    worst_order.quantity,
                );
                process_out_event(
                    event,
                    market,
                    event_heap,
                    Some(open_orders),
                    owner,
                    remaining_accs,
                )?;
            }

            let owner_slot = open_orders.next_order_slot()?;
            let new_order = LeafNode::new(
                owner_slot as u8,
                order_id,
                *owner,
                book_base_quantity_lots,
                now_ts,
                order.time_in_force,
                order.peg_limit(),
                order.client_order_id,
            );
            let _result = bookside.insert_leaf(order_tree_target, &new_order)?;

            open_orders.add_order(
                side,
                order_tree_target,
                &new_order,
                order.client_order_id,
                price,
            );
        }
```

挂单时，会先从该订单树中`remove_one_expired`以释放空间，这里也会生成一个**出对事件(OutEvent)**，执行`process_out_event`对挂单进行处理。还会判断`is_full`，满之后会移除一个最坏的挂单，同样也会生成一个**出对事件(OutEvent)**，执行`process_out_event`对挂单进行处理。

优化空间结束之后，就是挂单操作了，包括两个流程，一个是向`open_orders_account`中`add_order`，另一个就是向订单树中`insert_leaf`。

至此`new_order`的逻辑也梳理结束了，`place_order`的流程也告一段落，其中还有很多关于订单薄的细节并没有展开，这个随后再专门介绍。

更多内容可查看在github上的项目--[深入Solana OpenBook-V2源码分析与DeFi 合约实战](https://github.com/hunshenshi/solana-openbookv2-and-defi-in-action)