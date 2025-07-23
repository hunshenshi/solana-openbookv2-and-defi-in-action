`create_market`对外暴露的接口是在`lib.rs`中，代码如下：

``` Rust
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
```

其中`#[allow(clippy::too_many_arguments)]`宏是一个**编译器 lint 抑制宏**，就是告诉编译器允许这个函数有多个参数，而不触发`clippy`的告警。

`lib.rs`调用了`instructions`中的`create_market`，代码在`instructions/create_market.rs`中，接下来我们梳理下代码逻辑。

先看下参数校验的逻辑(对于合约来说参数校验尤为重要，避免了一些逻辑漏洞，防止别有用心之人绕过正常逻辑)
``` Rust
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
    let registration_time = Clock::get()?.unix_timestamp;

    require!(
        maker_fee.unsigned_abs() as i128 <= FEES_SCALE_FACTOR,
        OpenBookError::InvalidInputMarketFees
    );
    require!(
        taker_fee.unsigned_abs() as i128 <= FEES_SCALE_FACTOR,
        OpenBookError::InvalidInputMarketFees
    );
    require!(
        taker_fee >= 0 && (maker_fee >= 0 || maker_fee.abs() <= taker_fee),
        OpenBookError::InvalidInputMarketFees
    );

    require!(
        time_expiry == 0 || time_expiry > Clock::get()?.unix_timestamp,
        OpenBookError::InvalidInputMarketExpired
    );

    require_gt!(quote_lot_size, 0, OpenBookError::InvalidInputLots);
    require_gt!(base_lot_size, 0, OpenBookError::InvalidInputLots);

    let oracle_a = ctx.accounts.oracle_a.non_zero_key();
    let oracle_b = ctx.accounts.oracle_b.non_zero_key();

    if oracle_a.is_some() && oracle_b.is_some() {
        let oracle_a = AccountInfoRef::borrow(ctx.accounts.oracle_a.as_ref().unwrap())?;
        let oracle_b = AccountInfoRef::borrow(ctx.accounts.oracle_b.as_ref().unwrap())?;

        require_keys_neq!(*oracle_a.key, *oracle_b.key);
        require!(
            oracle::determine_oracle_type(&oracle_a)? == oracle::determine_oracle_type(&oracle_b)?,
            OpenBookError::InvalidOracleTypes
        );
    } else if oracle_a.is_some() {
        let oracle_a = AccountInfoRef::borrow(ctx.accounts.oracle_a.as_ref().unwrap())?;
        oracle::determine_oracle_type(&oracle_a)?;
    } else if oracle_b.is_some() {
        return Err(OpenBookError::InvalidSecondOracle.into());
    }
...
}
```
这里校验了`maker_fee`、`taker_fee`、`time_expiry`、`quote_lot_size`和`base_lot_size`，逻辑较为简单，不过这里有几个业务名词需要解释下：

* maker_fee
	做市商手续费。做市商是指挂单的账户，为市场提供流动性，其值允许为负
* taker_fee
	吃单手续费。吃单是指吃掉市场上的订单，会减少市场的流动性，其值通常为正
* quote_lot_size
	报价币的最小单位，俗称一手的数量。主要是为了控制精度，防止浮点误差
* base_lot_size
	基础币的最小单位，与`quote_lot_size`成对出现

还对`oracle_a`和`oracle_b`进行来校验，它们来自`Context<CreateMarket>`中，主要校验了它们是否有值，而且最少要保证`oracle_a`有值。

接下来`create_market`的核心代码片段：

``` Rust

    let mut openbook_market = ctx.accounts.market.load_init()?;
    *openbook_market = Market {
        market_authority: ctx.accounts.market_authority.key(),
        collect_fee_admin: ctx.accounts.collect_fee_admin.key(),
        open_orders_admin: ctx.accounts.open_orders_admin.non_zero_key(),
        consume_events_admin: ctx.accounts.consume_events_admin.non_zero_key(),
        close_market_admin: ctx.accounts.close_market_admin.non_zero_key(),
        bump: ctx.bumps.market_authority,
        base_decimals: ctx.accounts.base_mint.decimals,
        quote_decimals: ctx.accounts.quote_mint.decimals,
        padding1: Default::default(),
        time_expiry,
        name: fill_from_str(&name)?,
        bids: ctx.accounts.bids.key(),
        asks: ctx.accounts.asks.key(),
        event_heap: ctx.accounts.event_heap.key(),
        ...
    };

    let mut orderbook = Orderbook {
        bids: ctx.accounts.bids.load_init()?,
        asks: ctx.accounts.asks.load_init()?,
    };
    orderbook.init();

    let mut event_heap = ctx.accounts.event_heap.load_init()?;
    event_heap.init();
```
一切参数都合法之后，开始执行核心代码，创建`openbook_market`、`orderbook`、和`event_heap`，在创建时都是先调用`load_init()`对账户进行初始化并获取该账户的可变引用，然后再对该账户进行数据写入，之所以要调用`load_init()`，是因为`market`、`bids`、`asks`和`event_heap`在`CreateMarket`中都是`AccountLoader`类型，这种类型主要用于`Zero-Copy`账户。

**其中`openbook_market`通过参数赋值创建一个全新的市场账户，`orderbook`主要用于存储撮合交易，由`bids`买单薄和`asks`卖单薄组成，`event_heap`用于存储撮合、成交等事件，便于后续处理和查询。**

这里提到的`CreateMarket`数据结构是账户声明结构（`#[derive(Accounts)]`），用于支持 `create_market(ctx, ...)` 指令的账户校验与初始化，它定义了创建一个新市场所需的**所有账户**。我们下面逐段解释其 **结构设计、用途和背后的设计逻辑**。

具体代码如下：

``` Rust
#[event_cpi]
#[derive(Accounts)]
pub struct CreateMarket<'info> {
    #[account(
        init,
        payer = payer,
        space = 8 + std::mem::size_of::<Market>(),
    )]
    pub market: AccountLoader<'info, Market>,
    #[account(
        seeds = [b"Market".as_ref(), market.key().to_bytes().as_ref()],
        bump,
    )]
    /// CHECK:
    pub market_authority: UncheckedAccount<'info>,

    /// Accounts are initialized by client,
    /// anchor discriminator is set first when ix exits,
    #[account(zero)]
    pub bids: AccountLoader<'info, BookSide>,
    #[account(zero)]
    pub asks: AccountLoader<'info, BookSide>,
    #[account(zero)]
    pub event_heap: AccountLoader<'info, EventHeap>,

    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(
        init,
        payer = payer,
        associated_token::mint = base_mint,
        associated_token::authority = market_authority,
    )]
    pub market_base_vault: Account<'info, TokenAccount>,
    #[account(
        init,
        payer = payer,
        associated_token::mint = quote_mint,
        associated_token::authority = market_authority,
    )]
    pub market_quote_vault: Account<'info, TokenAccount>,

    #[account(constraint = base_mint.key() != quote_mint.key())]
    pub base_mint: Box<Account<'info, Mint>>,
    pub quote_mint: Box<Account<'info, Mint>>,

    pub system_program: Program<'info, System>,
    pub token_program: Program<'info, Token>,
    pub associated_token_program: Program<'info, AssociatedToken>,
    /// CHECK: The oracle can be one of several different account types
    pub oracle_a: Option<UncheckedAccount<'info>>,
    /// CHECK: The oracle can be one of several different account types
    pub oracle_b: Option<UncheckedAccount<'info>>,

    /// CHECK:
    pub collect_fee_admin: UncheckedAccount<'info>,
    /// CHECK:
    pub open_orders_admin: Option<UncheckedAccount<'info>>,
    /// CHECK:
    pub consume_events_admin: Option<UncheckedAccount<'info>>,
    /// CHECK:
    pub close_market_admin: Option<UncheckedAccount<'info>>,
}
```

* `market`就是要创建的账户，其使用`account`指定的约束有`init`、`payer`和`space`，其中`init`表示账户不存在，需要`Anchor`框架创建，`payer`指定了在创建账户时需要支付fee的账户，`space`指定要为这个账户分配多少**bytes**的空间，根据分配空间的大小支付租金。

> Solana中的账户是一个**定长内存结构**，创建之后空间大小不可以调整，新创建的账户必须指定`space`。

* `market_authority`是一个PDA账户，用于管理资金池，用**PDA主要是为了隔离状态与权限，并且方便程序调用**。使用`account`指定的约束有`seeds`和`bump`，其中`seeds`是生成PDA账户时用到的种子，`bump`是用来将地址推出椭圆曲线所使用的随机数字。

> PDA是程序派生账户
> 这里用的是`Market`字符串和`market`账户的私钥，之所以加上`market`账户的私钥，是为了每个`market`都有自己唯一的一个authority，每个`market`的资金都可以独立管理。

* `bids`、`asks`和`event_heap` 分别为买单树、卖单树和撮合事件堆，都有一个`#[account(zero)]`约束，表示该账户已经在`client`端预创建，只是数据全为0。这是一种优化`compute unit`和账户创建流程的高级技巧，通常用在账户结构较为复杂的情况下。
* `payer`是用来支付`fee`和`rent`的账户，数据会发生变化，所以使用`#[account(mut)]`进行约束。
* `market_base_vault`和`market_quote_vault`是成对出现的，`market_base_vault`是用于存放基础代币的资金池，而`market_quote_vault`是用来存放报价代币的资金池，比如`SOL/USDC`交易市场中，`SOL`就是`base`，而`USDC`则是`quote`，这是两个ATA账户。他们使用的`account`约束有`init`、`payer`、`associated_token::mint`和`associated_token::authority`，其中`init`、`payer`和之前的用法一样，`associated_token::mint`和`associated_token::authority`这两个约束是 Anchor 框架为自动创建 **ATA（Associated Token Account）** 提供的配置参数，用于告诉Anchor给指定 `authority` 创建一个特定 `mint` 的 Token 账户（ATA）。`associated_token::mint`是创建的ATA要持有的token，`associated_token::authority`指定了创建的ATA账户的owner。
* `base_mint`和`quote_mint`用于指定交易对的两个代币的 **Mint 账户**，即它们代表了哪两个 SPL Token 被用于交易。比如`SOL/USDC`交易市场中，`base_mint`就是`SOL`在链上的`mint`地址，`quote_mint`就是`USDC`在链上的`mint`地址。这里使用了`Box`类型，是Anchor的一种优化策略，可以减少复制，尤其适合不会被修改的 `Mint` 类型（只读）。
* `system_program`、`token_program`和`associated_token_program`都是系统级的程序，内置在链上，由Solana官方部署和维护。
* `oracle_a`和`oracle_b`设置oracle的地址
* `collect_fee_admin`、`open_orders_admin`、`consume_events_admin`和`close_market_admin`都是些管理员角色
`CreateMarket`账户声明结构中的账户就都已经梳理清楚了，但是这里还有个`Market`结构体，这里定义了`market`账户中的内容。部分代码如下：

``` Rust
#[account(zero_copy)]
#[derive(Debug)]
pub struct Market {
    /// PDA bump
    pub bump: u8,

    /// Number of decimals used for the base token.
    ///
    /// Used to convert the oracle's price into a native/native price.
    pub base_decimals: u8,
    pub quote_decimals: u8,

    pub padding1: [u8; 5],

    // Pda for signing vault txs
    pub market_authority: Pubkey,
	...
    pub base_mint: Pubkey,
    pub quote_mint: Pubkey,

    pub market_base_vault: Pubkey,
    pub base_deposit_total: u64,

    pub market_quote_vault: Pubkey,
    pub quote_deposit_total: u64,

    pub reserved: [u8; 128],
}
```

最后一段代码是发送事件，代码如下：

``` Rust
    emit_cpi!(MarketMetaDataLog {
        market: ctx.accounts.market.key(),
        name,
        base_mint: ctx.accounts.base_mint.key(),
        quote_mint: ctx.accounts.quote_mint.key(),
        base_decimals: ctx.accounts.base_mint.decimals,
        quote_decimals: ctx.accounts.quote_mint.decimals,
        base_lot_size,
        quote_lot_size,
    });
```
这里是通过调用`emit_cpi!`来发送事件日志的，该方法主要用于CPI之间的调用。