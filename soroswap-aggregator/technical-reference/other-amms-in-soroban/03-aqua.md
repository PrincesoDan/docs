# Aqua Protocol 

The aqua protocol is a soroba-AMM: [repository](https://github.com/AquaToken/soroban-amm/tree/master)

## Equation - StableSwap Mathematics

The Aquarius protocol implements a StableSwap algorithm similar to Curve Finance for its stable liquidity pool. This document provides a detailed explanation of the fundamental mathematical models that govern the behavior of the [Smart Cntract](https://github.com/AquaToken/soroban-amm/tree/master/liquidity_pool_stableswap).

### The StableSwap Invariant 

The core of the StableSwap model is its invariant equation. Unlike Uniswap's constant product model ($x \cdot y = k$), StableSwap uses a more complex invariant function:

$$A \cdot n^n \cdot \sum_{i=1}^{n} x_i + D = A \cdot D \cdot n^n + \frac{D^{n+1}}{n^n \cdot \prod_{i=1}^{n} x_i}$$

Where:
- $x_i$ are the normalized balances of each token in the pool
- $n$ is the number of tokens in the pool
- $A$ is the amplification coefficient (explained below)
- $D$ is the invariant that remains constant during swaps

This equation produces a hybrid behavior:
- When $A \to 0$: it approaches a constant sum AMM ($\sum x_i = k$)
- When $A \to \infty$: it approaches a constant product AMM ($\prod x_i = k$)
- For intermediate values of $A$: optimal balance for stable tokens

## Core Mathematical Functions

### 1. Swap: The Token Exchange

The `swap` function is the pool's main operation that allows exchanging one token for another.

**Signature:**

`fn swap(user, in_idx, out_idx, in_amount, out_min) → out_amount`

**Mathematical model:**

1. **Normalized data preparation:**
   $$xp_i = \text{reserves}_i \cdot \text{precision_mul}_i$$
   $$
   xp_i = \mathrm{reserves}_i \cdot \mathrm{precision\_mul}_i
   $$

2. **Compute new input balance:**
   $$x' = xp_{in\_idx} + in\_amount \cdot \text{precision_mul}_{in\_idx}$$

3. **Determine the new output balance:**
   $$y = \text{\_get\_y}(in\_idx, out\_idx, x', xp)$$

4. **Compute raw output amount:**
   $$dy_{raw} = xp_{out\_idx} - y - 1$$

5. **Apply fee:**
   $$dy_{fee} = dy_{raw} \cdot \frac{fee}{FEE\_DENOMINATOR}$$

6. **Final output amount:**
   $$out\_amount = \frac{dy_{raw} - dy_{fee}}{\text{precision_mul}_{out\_idx}}$$

7. **Minimum check:**
   If $out\_amount < out\_min$, the transaction fails with error `OutMinNotSatisfied`

This operation ensures that the invariant $D$ remains constant after the swap, preserving pool stability.

### 2. _get_y: Output Balance Calculation

This function solves for the value of $y$ (output token balance) that keeps the pool's invariant constant after adjusting the input token balance.

**Signature:**

`fn get_y(in_idx, out_idx, x, xp) → y`

**Mathematical model:**

1. **Compute current invariant:**
   $$D = \text{\_get\_d}(xp, A)$$

2. **Prepare coefficients:**
   $$c = \frac{D^{n+1}}{(A \cdot n^n) \cdot \prod_{i \neq out\_idx} x_i}$$

   $$s = \sum_{i \neq out\_idx} x_i$$
   
   $$b = s + \frac{D}{A \cdot n}$$

3. **Iterative solution (Newton's method):**
   $$y_{i+1} = \frac{y_i^2 + c}{2y_i + b - D}$$

4. **Convergence criterion:**
   Iteration stops when $|y_{i+1} - y_i| \leq 1$

This algorithm precisely finds the new output balance that maintains the invariant $D$.

### 3. a: The Amplification Coefficient

The coefficient $A$ determines the curvature of the StableSwap function and can be adjusted over time.

**Signature:**

`fn a() → amp`

**Mathematical model:**

The value of $A$ may follow a linear ramp over time:

$$A(t) = 
\begin{cases}
A_0 + (A_1 - A_0) \cdot \frac{t - t_0}{t_1 - t_0}, & \text{if } A_1 > A_0 \text{ and } t_0 \leq t < t_1 \\
A_0 - (A_0 - A_1) \cdot \frac{t - t_0}{t_1 - t_0}, & \text{if } A_1 < A_0 \text{ and } t_0 \leq t < t_1 \\
A_1, & \text{if } t \geq t_1
\end{cases}$$

Where:
- $A_0$ is the initial value (`initial_a`)
- $A_1$ is the target value (`future_a`)
- $t_0$ is the ramp start time (`initial_a_time`)
- $t_1$ is the ramp end time (`future_a_time`)
- $t$ is the current time (`now`)

This ramp mechanism allows the pool's curvature to be gradually adjusted without causing price discontinuities.

### 4. _get_d: Invariant Calculation

This function computes the value of the invariant $D$ for a given set of balances and amplification coefficient.

**Signature:**

`fn _get_d(xp, amp) → D`

**Mathematical model:**

1. **Sum of balances:**
   $$S = \sum_{i=1}^{n} x_i$$

2. **Initialization:**
   If $S = 0$, then $D = 0$ and we exit.
   Otherwise, set $D_0 = S$

3. **Iterative process:**
   For each iteration $j$:
   
   $$D_{p,j} = D_j \cdot \prod_{i=1}^{n} \frac{D_j}{n \cdot x_i}$$
   
   $$ann = A \cdot n^n$$
   
   $$D_{j+1} = \frac{(ann \cdot S + D_{p,j} \cdot n) \cdot D_j}{(ann - 1) \cdot D_j + (n + 1) \cdot D_{p,j}}$$

4. **Convergence criterion:**
   Iteration stops when $|D_{j+1} - D_j| \leq 1$

This algorithm converges quickly (typically in fewer than 10 iterations) to the correct value of $D$.

## Currency Implications

The StableSwap model offers several key advantages:

1. **Reduced slippage:** For exchanges between stable tokens (with similar values), slippage is significantly lower than in traditional AMMs.

2. **Adaptive flexibility:** The parameter $A$ allows dynamic adjustment of the swap curve based on market conditions.

3. **Stable price zone:** When tokens maintain parity, exchanges occur with minimal loss, similar to a 1:1 swap.

4. **Resistance to deviation:** If a token loses parity, the curve automatically adjusts to discourage excessive arbitrage.

Aqua Protocol adopts this sophisticated model to provide optimal stable token exchanges within the Stellar ecosystem.


### Anexos

1. [function a()](https://github.com/AquaToken/soroban-amm/blob/a4b1b0e32fbdb632f8e37df6aa137f74e2568ff1/liquidity_pool_stableswap/src/contract.rs#L72):
```rust
    fn a(e: Env) -> u128 {
        // Handle ramping A up or down
        let t1 = get_future_a_time(&e) as u128;
        let a1 = get_future_a(&e);
        let now = e.ledger().timestamp() as u128;

        if now < t1 {
            let a0 = get_initial_a(&e);
            let t0 = get_initial_a_time(&e) as u128;
            // Expressions in u128 cannot have negative numbers, thus "if"
            if a1 > a0 {
                a0 + (a1 - a0).fixed_mul_floor(&e, &(now - t0), &(t1 - t0))
            } else {
                a0 - (a0 - a1).fixed_mul_floor(&e, &(now - t0), &(t1 - t0))
            }
        } else {
            // when t1 == 0 or block.timestamp >= t1
            a1
        }
    }
```
2. function swap():
```rust
    // Swaps tokens in the pool.
    //
    // # Arguments
    //
    // * `user` - The address of the user swapping the tokens.
    // * `in_idx` - The index of the input token to be swapped.
    // * `out_idx` - The index of the output token to be received.
    // * `in_amount` - The amount of the input token to be swapped.
    // * `out_min` - The minimum amount of the output token to be received.
    //
    // # Returns
    //
    // The amount of the output token received.
    fn swap(
        e: Env,
        user: Address,
        in_idx: u32,
        out_idx: u32,
        in_amount: u128,
        out_min: u128,
    ) -> u128 {
        user.require_auth();
        if get_is_killed_swap(&e) {
            panic_with_error!(e, LiquidityPoolError::PoolSwapKilled);
        }

        if in_amount == 0 {
            panic_with_error!(e, LiquidityPoolValidationError::ZeroAmount);
        }

        let precision_mul = get_precision_mul(&e);
        let old_balances = get_reserves(&e);
        let xp = Self::_xp(&e, &old_balances);

        let coins = get_tokens(&e);
        let input_coin = coins.get(in_idx).unwrap();

        let token_client = SorobanTokenClient::new(&e, &input_coin);
        token_client.transfer(&user, &e.current_contract_address(), &(in_amount as i128));

        let reserve_sell = old_balances.get(in_idx).unwrap();
        let reserve_buy = old_balances.get(out_idx).unwrap();
        if reserve_sell == 0 || reserve_buy == 0 {
            panic_with_error!(e, LiquidityPoolValidationError::EmptyPool);
        }

        let x = xp.get(in_idx).unwrap() + in_amount * precision_mul.get(in_idx).unwrap();
        let y = Self::_get_y(&e, in_idx, out_idx, x, &xp);

        let dy = xp.get(out_idx).unwrap() - y - 1; // -1 just in case there were some rounding errors
        let dy_fee = dy.fixed_mul_ceil(&e, &(get_fee(&e) as u128), &(FEE_DENOMINATOR as u128));

        // Convert all to real units
        let dy = (dy - dy_fee) / precision_mul.get(out_idx).unwrap();
        if dy < out_min {
            panic_with_error!(e, LiquidityPoolValidationError::OutMinNotSatisfied);
        }

        // Change balances exactly in same way as we change actual ERC20 coin amounts
        let mut reserves = get_reserves(&e);
        reserves.set(in_idx, old_balances.get(in_idx).unwrap() + in_amount);
        reserves.set(out_idx, old_balances.get(out_idx).unwrap() - dy);
        put_reserves(&e, &reserves);

        let token_out = coins.get(out_idx).unwrap();
        let token_client = SorobanTokenClient::new(&e, &token_out);
        token_client.transfer(&e.current_contract_address(), &user, &(dy as i128));

        // update plane data for every pool update
        update_plane(&e);

        // since we need fee in amount sent to the pool, calculate it here
        let dx_fee =
            in_amount.fixed_mul_ceil(&e, &(get_fee(&e) as u128), &(FEE_DENOMINATOR as u128));
        PoolEvents::new(&e).trade(user, input_coin, token_out, in_amount, dy, dx_fee);

        dy
    }
    ```

