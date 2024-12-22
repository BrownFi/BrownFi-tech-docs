# Mission  
*Renovate AMM with high capital efficiency, simple UX, better gains for average users*.  
Imagine 200X capital efficiency with simple-UniV2-like LP

# 1) BrownFi AMM: a brief intro
## 1.1) Fundamental problems
The pain-points of LPers on Uniswap V3 model (and the forks):
- Uniswap V3 has high CE but complicated UX for LPs.
- Half of LPs on Uniswap V3 lose due to high IL. On the entire Ethereum, LPs have lost $840M for arbitrageurs since the Merge, according to https://sorellalabs.xyz/.

Finding an AMM model with high CE and simple UX for average LPs with arbitrage resistance will bring huge benefits for new ecosystems with limited capital and low liquidity.

## 1.2) Solution (a novel AMM - Elastic PLOB)
BrownFi invents a novel pricing mechanism for spot AMM based on oracle, to offer high capital efficiency, flexible market making and simple UX, arbitrage resistance. The core concept of BrownFi AMM employs an elastic Parameterization of Limit Order-Book (PLOB) from a published [research papers](https://ieeexplore.ieee.org/abstract/document/10456889) on IEEE Access, a notable scientific journal. In brief, the core function of BrownFi AMM is based on an invention of a NOVEL pricing mechanism, presented in [this post](https://mirror.xyz/0x64f4Fbd29b0AE2C8e18E7940CF823df5CB639bBa/5lSUhDUCCSZTxznxfkClDvLkwE3wr_swFCH_mT9fXLI).
Mathematically, we prove that the constant-product market making (CPMM) model *xy=k* of Uniswap V2 is a special case of BrownFi's elastic PLOB model.

## 1.3) Capital efficiency
Our simulation shows that BrownFi AMM offers capital efficiency as equivalent as Uniswap V3 (bin range +/- 1%) and 200X better Uniswap V2.

## 1.4) BrownFi: market proposition & USP
BrownFi’s key advantages:  
- NO out-of-range
- High CE with flexible market marking strategies
- Simple LP management with adjustable liquidity concentration
- Resistance to arbitrage attacks
- One-sided LP (NOT zap in)

![image](https://github.com/user-attachments/assets/15b5ee91-5fe1-435a-a404-5d5b15ba7a51)


# 2) Math and Core Function

The core function of BrownFi is based on an invention of a NOVEL **pricing mechanism** which parameterizes limit order-book: 
- Given a pair of tokens X and Y, where X is base token (e.g. ETH) and Y is quote token (e.g. USDT). 
- Assume that the pool reserve has $x_0$ tokens X and $y_0$ tokens Y with the associated price is $P_0.$
- For any trade, amountOUT $dx$ of token X into the pool, we compute the average trading price $\bar{P} = P_0 * (1 + a*R),$ where $R$ denotes price impact factor, and normally $a=\frac{2}{3}, \frac{1}{2}$.  
- Price impact $R=K * f(dx)$ is proportionate to relative order size $\delta x=dx/x_0.$ Liquidity compression parameter, Kappa, $K$ is usually set LARGE for HIGH volatility (hence great slippage), and small for low volatility (hence small slippage). For $K=2$, our elastic PLOB model is mostly the same as CPMM of Uniswap V2. Note that our $K$ is **NOT** as same as in Uniswap formula $xy=k.$
- Finally, we compute amountIN $dy=dx*\bar{P}$ based on the amountOUT and the computed average trading price, then execute swaption.

Read more about the math [research papers](https://ieeexplore.ieee.org/abstract/document/10456889) on IEEE Access.

