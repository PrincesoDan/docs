# Aqua Protocol 

The aqua protocol is a soroba-AMM: [repository](https://github.com/AquaToken/soroban-amm/tree/master)

## Equation - StableSwap Mathematics

The Aquarius protocol implements a StableSwap algorithm similar to Curve Finance for its stable liquidity pool. This document provides a detailed explanation of the fundamental mathematical models that govern the behavior of the [Smart Contract](https://github.com/AquaToken/soroban-amm/tree/master/liquidity_pool_stableswap).

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
   $$xp(i) = reserves_i \cdot precision\_mul(i)$$

2. **Compute new input balance:**
   $$x' = xp(in\_idx) + in\_amount \cdot precision\_mul(in\_idx)$$

3. **Determine the new output balance:**
   $$y = \mathrm{\_get\_y}(in\_idx, out\_idx, x', xp)$$

4. **Compute raw output amount:**
   $$dy_{raw} = xp_{out\_idx} - y - 1$$

5. **Apply fee:**
   $$dy_{fee} = dy_{raw} \cdot \frac{fee}{FEE\_DENOMINATOR}$$

6. **Final output amount:**
   $$out\_amount = \frac{dy_{raw} - dy_{fee}}{{precision\_mul}_{out\_idx}}$$

7. **Minimum check:**
   If $out\_amount < out\_min$, the transaction fails with error `OutMinNotSatisfied`

This operation ensures that the invariant $D$ remains constant after the swap, preserving pool stability.

### 2. _get_y: Output Balance Calculation

This function solves for the value of $y$ (output token balance) that keeps the pool's invariant constant after adjusting the input token balance.

**Signature:**

`fn get_y(in_idx, out_idx, x, xp) → y`

**Mathematical model:**

1. **Compute current invariant:**
   $$    D = \text{\_get\_d}(xp, A)$$

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

### 5. Auxiliary Functions

#### a. read_decimals: Retrieve Token Decimals

This function retrieves the decimals for all tokens in a pool using the Soroban token client.

**Formula**:
$$ decimals = decimal_i$$
For each token in tokens

#### b. get_precision: Calculate Target Precision

Calculates the target precision for internal calculations, based on the maximum number of decimals among the tokens.

**Formula**:
$$ precision = 10^{max(decimals)}$$

#### c. get_precision_mul: Scale Token Amounts

Scales raw token amounts to match the `Precision`, accounting for decimal differences.

**Formula**:
$$precision\_mul(i) = \frac{precision}{10^{decimals_i}}$$

#### d. xp: Reserves in Normalized Form

Calculates reserves in normalized form, scaled to the `Precision`, by multiplying each reserve by its corresponding precision multiplier.

**Formula**:
$$xp(i) = precision\_mul(i) \times reserves_i$$

$$xp(i) = \frac{precision}{10^{decimals_i}} \times reserves_i$$

These functions are essential for handling tokens in a smart contract environment where precisions can vary, and it is necessary to unify scales for accurate calculations.

### Constants

$$FEE\_DENOMINATOR = 10000$$

## Currency Implications

The StableSwap model offers several key advantages:

1. **Reduced slippage:** For exchanges between stable tokens (with similar values), slippage is significantly lower than in traditional AMMs.

2. **Adaptive flexibility:** The parameter $A$ allows dynamic adjustment of the swap curve based on market conditions.

3. **Stable price zone:** When tokens maintain parity, exchanges occur with minimal loss, similar to a 1:1 swap.

4. **Resistance to deviation:** If a token loses parity, the curve automatically adjusts to discourage excessive arbitrage.

Aqua Protocol adopts this sophisticated model to provide optimal stable token exchanges within the Stellar ecosystem.

### Stimate functions

#### get_dy: Cálculo de la Cantidad de Token Recibido

Esta función calcula la cantidad de token `j` que se recibirá al intercambiar `dx` del token `i`.

**Firma:**

`fn get_dy(e: Env, i: u32, j: u32, dx: u128) → u128`

**Modelo matemático:**

1. **Preparación de datos normalizados:**
   $$xp(i) = reserves_i \cdot precision\_mul(i)$$

2. **Calcular nuevo balance de entrada:**
   $$x' = xp(i) + dx \cdot precision\_mul(i)$$

3. **Determinar el nuevo balance de salida:**
   $$y = \mathrm{\_get\_y}(i, j, x', xp)$$

4. **Calcular cantidad bruta de salida:**
   $$dy_{raw} = xp(j) - y - 1$$

5. **Aplicar tarifa:**
   $$dy_{fee} = dy_{raw} \cdot \frac{fee}{FEE\_DENOMINATOR}$$

6. **Cantidad final de salida:**
   $$dy = \frac{dy_{raw} - dy_{fee}}{precision\_mul(j)}$$

#### get_dx: Cálculo de la Cantidad de Token Enviado

Esta función calcula la cantidad de token `i` que se enviará al intercambiar `dy` del token `j`.

**Firma:**

`fn get_dx(e: Env, i: u32, j: u32, dy: u128) → u128`

**Modelo matemático:**

1. **Preparación de datos normalizados:**
   $$xp(j) = reserves_j \cdot precision\_mul(j)$$

2. **Aplicar tarifa a `dy`:**
   $$dy_{w\_fee} = dy \cdot \frac{FEE\_DENOMINATOR}{FEE\_DENOMINATOR - fee} \cdot precision\_mul(j)$$

3. **Verificar balance suficiente:**
   Si $dy_{w\_fee} \geq xp(j)$, la transacción falla con error `InsufficientBalance`.

4. **Calcular nuevo balance de salida con tarifa:**
   $$y_{w\_fee} = xp(j) - dy_{w\_fee}$$

5. **Determinar el nuevo balance de entrada:**
   $$x = \mathrm{\_get\_y}(j, i, y_{w\_fee}, xp)$$

6. **Calcular cantidad de entrada:**
   $$dx = \frac{x - xp(i) + 1}{precision\_mul(i)}$$

### Anexos

1. [function swap()](https://github.com/AquaToken/soroban-amm/blob/a4b1b0e32fbdb632f8e37df6aa137f74e2568ff1/liquidity_pool_stableswap/src/contract.rs#L1339)

2. [function _get_y()](https://github.com/AquaToken/soroban-amm/blob/a4b1b0e32fbdb632f8e37df6aa137f74e2568ff1/liquidity_pool_stableswap/src/contract.rs#L509)

3. [function _get_d()](https://github.com/AquaToken/soroban-amm/blob/a4b1b0e32fbdb632f8e37df6aa137f74e2568ff1/liquidity_pool_stableswap/src/contract.rs#L446)

4. [function a()](https://github.com/AquaToken/soroban-amm/blob/a4b1b0e32fbdb632f8e37df6aa137f74e2568ff1/liquidity_pool_stableswap/src/contract.rs#L72)

5. [function xp()](https://github.com/AquaToken/soroban-amm/blob/a4b1b0e32fbdb632f8e37df6aa137f74e2568ff1/liquidity_pool_stableswap/src/normalize.rs#L35)

6. [function get_precision_mul()](https://github.com/AquaToken/soroban-amm/blob/a4b1b0e32fbdb632f8e37df6aa137f74e2568ff1/liquidity_pool_stableswap/src/normalize.rs#L25C8-L25C68)

7. [function get_dy()](https://github.com/AquaToken/soroban-amm/blob/a4b1b0e32fbdb632f8e37df6aa137f74e2568ff1/liquidity_pool_stableswap/src/contract.rs#L159)

8. [function get_dx()](https://github.com/AquaToken/soroban-amm/blob/a4b1b0e32fbdb632f8e37df6aa137f74e2568ff1/liquidity_pool_stableswap/src/contract.rs#L190)