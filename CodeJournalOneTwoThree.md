# CW20 Staking
#### This is a sample contract that releases a minimal form of staking derivatives. This is to be used for integration tests and as a foundation for other to build more complex logic upon.

Source: https://github.com/CosmWasm/cw-tokens/tree/main/contracts/cw20-staking
Owner: CosmWasm

-------------------------------------------------
## Functionality
This contract acts as CW20 token but has additional staking features. The cw20-staking contract hold list of balances for multiple addresses. This contract has no initial balance such that it mints and burns them based on delegations.

---------------------------

## State

The cw20-staking `state.rs` comes with 3 type wrapper as follows:
```rust
/// User can claims toke based on the state of this const
pub const CLAIMS: Claims = Claims::new("claims");
/// Investment stores an InvesmentInfo by wrapping it into Item
pub const INVESTMENT: Item<InvestmentInfo> = Item::new("invest");
/// Total Supply represent the total of the token supply
pub const TOTAL_SUPPLY: Item<Supply> = Item::new("total_supply");
```

The first state is `InvestmentInfo` which contains all information related to the owner and staking information. It can be used to help identify the redeem period, staking info, validator, etc.
```rust
/// Investment info is fixed at instantiation, and is used to control the function of the contract
#[cw_serde]
pub struct InvestmentInfo {
    /// Owner that created the contract and takes a cut
    pub owner: Addr,
    /// This is the denomination we can stake (and only one we accept for payments)
    pub bond_denom: String,
    /// This is the unbonding period of the native staking module
    /// We need this to only allow claims to be redeemed after the money has arrived
    pub unbonding_period: Duration,
    /// This is how much the owner takes as a cut when someone unbonds
    pub exit_tax: Decimal,
    /// All tokens are bonded to this validator
    pub validator: String,
    /// This is the minimum amount we will pull out to reinvest, as well as a minimum
    /// that can be unbonded (to avoid needless staking tx)
    pub min_withdrawal: Uint128,
}
```

The second state is `Supply` which is used to track the supply of ERC20 tokens staked in the network.
```rust
/// Supply is dynamic and tracks the current supply of staked and ERC20 tokens.
#[cw_serde]
#[derive(Default)]
pub struct Supply {
    /// issued is how many derivative tokens this contract has issued
    pub issued: Uint128,
    /// bonded is how many native tokens exist bonded to the validator
    pub bonded: Uint128,
    /// claims is how many tokens need to be reserved paying back those who unbonded
    pub claims: Uint128,
}
```
----------------------------------------------

## Message
For interacting with CosmWasm smart contract, ERC20-Staking has 3 common `Message` such as:

### A. Instantiate Message
The erc20-staking `Message` first describe the `InstantiateMsg` with staking feature such as minimal withdrawal, unbounding_period and tax for the owner of the contract.
```rust
#[cw_serde]
pub struct InstantiateMsg {
    /// name of the derivative token
    pub name: String,
    /// symbol / ticker of the derivative token
    pub symbol: String,
    /// decimal places of the derivative token (for UI)
    pub decimals: u8,

    /// This is the validator that all tokens will be bonded to
    pub validator: String,
    /// This is the unbonding period of the native staking module
    /// We need this to only allow claims to be redeemed after the money has arrived
    pub unbonding_period: Duration,

    /// this is how much the owner takes as a cut when someone unbonds
    pub exit_tax: Decimal,
    /// This is the minimum amount we will pull out to reinvest, as well as a minimum
    /// that can be unbonded (to avoid needless staking tx)
    pub min_withdrawal: Uint128,
}
```

The `ExecuteMsg` has 12 kind of `Message` type to describe staking functionality on the erc20-token. A `Bond {}` message sends native staking tokens to the contract to be bonded to a validator and credits the user with the appropriate amount of derivative tokens.

A `Reinvest {}` message acts like auto compounding which user will automatically stake again the rewards that they received.
```rust
#[cw_serde]
pub enum ExecuteMsg {
    /// Bond will bond all staking tokens sent with the message and release derivative tokens
    Bond {},
    /// Unbond will "burn" the given amount of derivative tokens and send the unbonded
    /// staking tokens to the message sender (after exit tax is deducted)
    Unbond { amount: Uint128 },
    /// Claim is used to claim your native tokens that you previously "unbonded"
    /// after the chain-defined waiting period (eg. 3 weeks)
    Claim {},
    /// Reinvest will check for all accumulated rewards, withdraw them, and
    /// re-bond them to the same validator. Anyone can call this, which updates
    /// the value of the token (how much under custody).
    Reinvest {},
    /// _BondAllTokens can only be called by the contract itself, after all rewards have been
    /// withdrawn. This is an example of using "callbacks" in message flows.
    /// This can only be invoked by the contract itself as a return from Reinvest
    _BondAllTokens {},

    /// Implements CW20. Transfer is a base message to move tokens to another account without triggering actions
    Transfer { recipient: String, amount: Uint128 },
    /// Implements CW20. Burn is a base message to destroy tokens forever
    Burn { amount: Uint128 },
    /// Implements CW20.  Send is a base message to transfer tokens to a contract and trigger an action
    /// on the receiving contract.
    Send {
        contract: String,
        amount: Uint128,
        msg: Binary,
    },
    /// Implements CW20 "approval" extension. Allows spender to access an additional amount tokens
    /// from the owner's (env.sender) account. If expires is Some(), overwrites current allowance
    /// expiration with this one.
    IncreaseAllowance {
        spender: String,
        amount: Uint128,
        expires: Option<Expiration>,
    },
    /// Implements CW20 "approval" extension. Lowers the spender's access of tokens
    /// from the owner's (env.sender) account by amount. If expires is Some(), overwrites current
    /// allowance expiration with this one.
    DecreaseAllowance {
        spender: String,
        amount: Uint128,
        expires: Option<Expiration>,
    },
    /// Implements CW20 "approval" extension. Transfers amount tokens from owner -> recipient
    /// if `env.sender` has sufficient pre-approval.
    TransferFrom {
        owner: String,
        recipient: String,
        amount: Uint128,
    },
    /// Implements CW20 "approval" extension. Sends amount tokens from owner -> contract
    /// if `env.sender` has sufficient pre-approval.
    SendFrom {
        owner: String,
        contract: String,
        amount: Uint128,
        msg: Binary,
    },
    /// Implements CW20 "approval" extension. Destroys tokens forever
    BurnFrom { owner: String, amount: Uint128 },
}
```
The query message consist of `Message` that can be obtained related to erc20-staking
```rust
#[cw_serde]
#[derive(QueryResponses)]
pub enum QueryMsg {
    /// Claims shows the number of tokens this address can access when they are done unbonding
    #[returns(ClaimsResponse)]
    Claims { address: String },
    /// Investment shows metadata on the staking info of the contract
    #[returns(InvestmentResponse)]
    Investment {},

    /// Implements CW20. Returns the current balance of the given address, 0 if unset.
    #[returns(BalanceResponse)]
    Balance { address: String },
    /// Implements CW20. Returns metadata on the contract - name, decimals, supply, etc.
    #[returns(TokenInfoResponse)]
    TokenInfo {},
    /// Implements CW20 "allowance" extension.
    /// Returns how much spender can use from owner account, 0 if unset.
    #[returns(AllowanceResponse)]
    Allowance { owner: String, spender: String },
}
```

The `InvestmentResponse` has information related to staking and it's feature such as knowing minimum withdrawal, nominal value and the exit tax which is a reward to the owner when someone remove the staking.
```rust
#[cw_serde]
pub struct InvestmentResponse {
    pub token_supply: Uint128,
    pub staked_tokens: Coin,
    // ratio of staked_tokens / token_supply (or how many native tokens that one derivative token is nominally worth)
    pub nominal_value: Decimal,

    /// owner created the contract and takes a cut
    pub owner: String,
    /// this is how much the owner takes as a cut when someone unbonds
    pub exit_tax: Decimal,
    /// All tokens are bonded to this validator
    pub validator: String,
    /// This is the minimum amount we will pull out to reinvest, as well as a minimum
    /// that can be unbonded (to avoid needless staking tx)
    pub min_withdrawal: Uint128,
}
```
--------------------------------------

## Contracts
This code journal doesn't include all the code of the erc20-staking contracts since it's more than 900 line of code. The code can be found [in this repository](https://github.com/CosmWasm/cw-tokens/blob/main/contracts/cw20-staking/src/contract.rs).

The first function of the contracts is `instantiate` function. 
```rust
pub fn instantiate( deps: DepsMut, env: Env, info: MessageInfo, msg: InstantiateMsg ) -> Result<Response, ContractError>
```
This function set all relevant information needed in the network. It sets the contract version in:
```rust
    set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?;
```
Then the function gather all validators which are already registered by doing an iteration over all validators.
```rust
    // ensure the validator is registered
    let vals = deps.querier.query_all_validators()?;
    if !vals.iter().any(|v| v.address == msg.validator) { // Rizary: it goes through all validator and find if any address that is validator
        return Err(ContractError::NotInValidatorSet {
            validator: msg.validator,
        });
    }
```

The function store the token information in the `TokenInfo` struct which is based on the cw20-base format.
```rust
    // store token info using cw20-base format
    let data = TokenInfo {
        // Rizary: name of the token
        name: msg.name,
        symbol: msg.symbol,
        decimals: msg.decimals,
        // Rizary: the total supply is always set to zero
        total_supply: Uint128::zero(),
        // set self as minter, so we can properly execute mint and burn
        mint: Some(MinterData {
            minter: env.contract.address,
            cap: None,
        }),
    };
    // Rizary: save the information to the network
    TOKEN_INFO.save(deps.storage, &data)?;
```
After that, the function call the bond denominator as an initial `InvestmentInfo` state and save it to the network
```rust
    let denom = deps.querier.query_bonded_denom()?;
    let invest = InvestmentInfo {
        /// Rizary: The owner of staked token
        owner: info.sender,
        /// Rizary: The exit tax if someone exit
        exit_tax: msg.exit_tax,
        /// Rizary: the permissible time to unstake
        unbonding_period: msg.unbonding_period,
        /// Rizary: denominator of bond
        bond_denom: denom,
        validator: msg.validator,
        /// Rizary: minimum withdrawal of the token
        min_withdrawal: msg.min_withdrawal,
    };
    INVESTMENT.save(deps.storage, &invest)?;
```
Last, it sets the supply to 0 and save it to the network.
```rust
    // set supply to 0
    let supply = Supply::default();
    TOTAL_SUPPLY.save(deps.storage, &supply)?;
```
The second function is:
```rust
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
```
Which describes all possible `Message` passed in the contract and mapped it to each function associate with the message. For an example, it will call `bond` function if the executed `Message` is `ExecuteMsg::Bond {}` type.
```rust
    ExecuteMsg::Bond {} => bond(deps, env, info),
    ExecuteMsg::Unbond { amount } => unbond(deps, env, info, amount),
    ExecuteMsg::Claim {} => claim(deps, env, info),
    ExecuteMsg::Reinvest {} => reinvest(deps, env, info),
```

In this journal, I analyze "bonding" functionality in the erc20-staking. The first is the `get_bonded` function which returns all delegations made in the contract.
```rust
// get_bonded returns the total amount of delegations from contract
// it ensures they are all the same denom
fn get_bonded(querier: &QuerierWrapper, contract: &Addr) -> Result<Uint128, ContractError> {
    let bonds = querier.query_all_delegations(contract)?; // Rizary: calling unwrap to get the result
    if bonds.is_empty() {
        return Ok(Uint128::zero());
    }
    let denom = bonds[0].amount.denom.as_str();
    /// Rizary: iterate over the bonds and check if 
    /// the denominator is the same or not
    bonds.iter().fold(Ok(Uint128::zero()), |racc, d| {
        let acc = racc?;
        if d.amount.denom.as_str() != denom {
            Err(ContractError::DifferentBondDenom {
                denom1: denom.into(),
                denom2: d.amount.denom.to_string(),
            })
        } else {
            Ok(acc + d.amount.amount)
        }
    })
}
```
Next is assert bonds which check the supply of the staked token is the same or not with the bonded token provide outside the supply.
```rust
fn assert_bonds(supply: &Supply, bonded: Uint128) -> Result<(), ContractError> {
    if supply.bonded != bonded {
        Err(ContractError::BondedMismatch {
            stored: supply.bonded,
            queried: bonded,
        })
    } else {
        Ok(())
    }
}
```
The `bond()` function check the payment information, total token that is delegated by the caller address, and then minting the token after it check the bonded supply using the above `asert_bond` function.
```rust
pub fn bond(deps: DepsMut, env: Env, info: MessageInfo) -> Result<Response, ContractError> {
    // ensure we have the proper denom
    let invest = INVESTMENT.load(deps.storage)?;
    // payment finds the proper coin (or throws an error)
    let payment = info
        .funds
        .iter()
        .find(|x| x.denom == invest.bond_denom)
        .ok_or_else(|| ContractError::EmptyBalance {
            denom: invest.bond_denom.clone(),
        })?;

    // bonded is the total number of tokens we have delegated from this address
    let bonded = get_bonded(&deps.querier, &env.contract.address)?;

    // calculate to_mint and update total supply
    let mut supply = TOTAL_SUPPLY.load(deps.storage)?;
    // TODO: this is just a safety assertion - do we keep it, or remove caching?
    // in the end supply is just there to cache the (expected) results of get_bonded() so we don't
    // have expensive queries everywhere
    assert_bonds(&supply, bonded)?;
    let to_mint = if supply.issued.is_zero() || bonded.is_zero() {
        FALLBACK_RATIO * payment.amount
    } else {
        payment.amount.multiply_ratio(supply.issued, bonded)
    };
    supply.bonded = bonded + payment.amount;
    supply.issued += to_mint;
    TOTAL_SUPPLY.save(deps.storage, &supply)?;

    // call into cw20-base to mint the token, call as self as no one else is allowed
    let sub_info = MessageInfo {
        sender: env.contract.address.clone(),
        funds: vec![],
    };
    execute_mint(deps, env, sub_info, info.sender.to_string(), to_mint)?;

    // bond them to the validator
    let res = Response::new()
        .add_message(StakingMsg::Delegate {
            validator: invest.validator,
            amount: payment.clone(),
        })
        .add_attribute("action", "bond")
        .add_attribute("from", info.sender)
        .add_attribute("bonded", payment.amount) // Rizary: the amount of the staked token
        .add_attribute("minted", to_mint);
    Ok(res)
}
```
The `unbound` function is used to make the user able to unstaked the token. It loads the investment information of the user's address and check whether the withdrawal amount intended is greater than the amount of token saved in the Invesment state in the network.

It then calculates the tax and execute burn function to burn the token. If the tax is greater than zero, then the erc20-staking contract will mint some amount of the token and send it to the owner of the contract.

After that, the `unbound()` function re-calculates the bonded information as well as the total token supply. Once it's done, it save the total supply to the network and create a `Claims` so that user can have the token back.
```rust
pub fn unbond(
    mut deps: DepsMut,
    env: Env,
    info: MessageInfo,
    amount: Uint128,
) -> Result<Response, ContractError> {
    let invest = INVESTMENT.load(deps.storage)?;
    // ensure it is big enough to care
    if amount < invest.min_withdrawal {
        return Err(ContractError::UnbondTooSmall {
            min_bonded: invest.min_withdrawal,
            denom: invest.bond_denom,
        });
    }
    // calculate tax and remainer to unbond
    let tax = amount * invest.exit_tax;

    // burn from the original caller
    execute_burn(deps.branch(), env.clone(), info.clone(), amount)?;
    if tax > Uint128::zero() {
        let sub_info = MessageInfo {
            sender: env.contract.address.clone(),
            funds: vec![],
        };
        // call into cw20-base to mint tokens to owner, call as self as no one else is allowed
        execute_mint(
            deps.branch(),
            env.clone(),
            sub_info,
            invest.owner.to_string(),
            tax,
        )?;
    }

    // re-calculate bonded to ensure we have real values
    // bonded is the total number of tokens we have delegated from this address
    let bonded = get_bonded(&deps.querier, &env.contract.address)?;

    // calculate how many native tokens this is worth and update supply
    let remainder = amount.checked_sub(tax).map_err(StdError::overflow)?;
    let mut supply = TOTAL_SUPPLY.load(deps.storage)?;
    // in the end supply is just there to cache the (expected) results of get_bonded() so we don't
    // have expensive queries everywhere
    assert_bonds(&supply, bonded)?;
    let unbond = remainder.multiply_ratio(bonded, supply.issued);
    supply.bonded = bonded.checked_sub(unbond).map_err(StdError::overflow)?;
    supply.issued = supply
        .issued
        .checked_sub(remainder)
        .map_err(StdError::overflow)?;
    supply.claims += unbond;
    TOTAL_SUPPLY.save(deps.storage, &supply)?;

    CLAIMS.create_claim(
        deps.storage,
        &info.sender,
        unbond,
        invest.unbonding_period.after(&env.block),
    )?;

    // unbond them
    let res = Response::new()
        .add_message(StakingMsg::Undelegate {
            validator: invest.validator,
            amount: coin(unbond.u128(), &invest.bond_denom),
        })
        .add_attribute("action", "unbond")
        .add_attribute("to", info.sender)
        .add_attribute("unbonded", unbond)
        .add_attribute("burnt", amount);
    Ok(res)
}
```

The other function will be stored in separate journal.
