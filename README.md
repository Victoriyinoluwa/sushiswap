
## Overview

This script automates interactions with SushiSwap and MasterChef contracts on the Sepolia test network. It performs the following actions:

1. **Token Swapping**: Swaps USDC for LINK using SushiSwap's SwapRouter.
2. **Token Approval**: Approves tokens for spending by the SwapRouter and MasterChef contracts.
3. **LP Token Staking**: Stakes LINK LP tokens into SushiSwap's MasterChef contract.

### DeFi Protocols and Workflow

- **SushiSwap**:
  - **SwapRouter**: Used for token swapping.
  - **Pool**: Provides liquidity pools for swapping tokens.

- **MasterChef**:
  - Manages staking of LP tokens.

### Workflow Diagram

```plaintext
+------------------+            +----------------+            +----------------+          
|                  |            |                |            |                |
|   Approve USDC   |            |   Swap USDC    |            |   Approve LP   |
|    Token         |            |    for LINK    |            |   Tokens       |
|                  |            |                |            |                |
+------------------+            +----------------+            +----------------+          
            |                             |                          |                         
            V                             V                          V                         
+------------------+            +----------------+            +----------------+          
|                  |            |                |            |                |
|   Execute Swap   |            |  Approve LP    |            |   Stake LP     |
|    Transaction   |            |   Tokens       |            |   Tokens       |
|                  |            |                |            |                |
+------------------+            +----------------+            +----------------+
            |                             |                          |                         
            V                             V                          V                         
+------------------+            +----------------+            +----------------+          
|                  |            |                |            |                |
|   Complete       |            |   Complete     |            |   Complete     |
|   Swap           |            |   Approval     |            |   Staking      |
|                  |            |                |            |                |
+------------------+            +----------------+            +----------------+
```



## Code Explanation

This script automates the process of swapping tokens using SushiSwap, supplying tokens to Compound, and staking LP tokens into SushiSwap's MasterChef contract. It interacts with various smart contracts and executes multiple operations. Below is a detailed explanation of each part of the code:

### 1. Importing Dependencies and Configuration

```javascript
import { ethers } from "ethers";
import FACTORY_ABI from "./abis/factory.json" assert { type: "json" };
import SWAP_ROUTER_ABI from "./abis/swaprouter.json" assert { type: "json" };
import POOL_ABI from "./abis/pool.json" assert { type: "json" };
import TOKEN_IN_ABI from "./abis/token.json" assert { type: "json" };
import MASTERCHEF_ABI from "./abis/masterchef.json" assert { type: "json" };
import dotenv from "dotenv";
dotenv.config();
```

- **Dependencies**: The script uses `ethers` for Ethereum interactions and imports ABI definitions for smart contracts (Factory, SwapRouter, Pool, Token, MasterChef).
- **Environment Variables**: `dotenv` is used to load environment variables such as the RPC URL and private key.

### 2. Contract Addresses and Provider Setup

```javascript
const POOL_FACTORY_CONTRACT_ADDRESS = "0x0227628f3F023bb0B980b67D528571c95c6DaC1c";
const SWAP_ROUTER_CONTRACT_ADDRESS = "0x3bFA4769FB09eefC5a80d6E87c3B9C650f7Ae48E";
const MASTERCHEF_CONTRACT_ADDRESS = "0x1234567890abcdef1234567890abcdef12345678";

const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
const factoryContract = new ethers.Contract(POOL_FACTORY_CONTRACT_ADDRESS, FACTORY_ABI, provider);
const signer = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
```

- **Contract Addresses**: Addresses for the Factory, SwapRouter, and MasterChef contracts.
- **Provider and Signer**: Setup for interacting with the Ethereum network using the specified RPC URL and private key.

### 3. Token Configuration

```javascript
const USDC = {
  chainId: 11155111,
  address: "0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238",
  decimals: 6,
  symbol: "USDC",
  name: "USD//C",
  isToken: true,
  isNative: true,
  wrapped: false,
};

const LINK = {
  chainId: 11155111,
  address: "0x779877A7B0D9E8603169DdbD7836e478b4624789",
  decimals: 18,
  symbol: "LINK",
  name: "Chainlink",
  isToken: true,
  isNative: true,
  wrapped: false,
};
```

- **USDC and LINK**: Configuration objects containing information about the USDC and LINK tokens, including their addresses, decimals, and symbols.

### 4. Approve Token Function

```javascript
async function approveToken(tokenAddress, tokenABI, amount, wallet) {
  try {
    const tokenContract = new ethers.Contract(tokenAddress, tokenABI, wallet);
    const approveAmount = ethers.parseUnits(amount.toString(), USDC.decimals);
    const approveTransaction = await tokenContract.approve.populateTransaction(
      SWAP_ROUTER_CONTRACT_ADDRESS,
      approveAmount
    );
    const transactionResponse = await wallet.sendTransaction(approveTransaction);
    console.log(`-------------------------------`);
    console.log(`Sending Approval Transaction...`);
    console.log(`-------------------------------`);
    console.log(`Transaction Sent: ${transactionResponse.hash}`);
    console.log(`-------------------------------`);
    const receipt = await transactionResponse.wait();
    console.log(
      `Approval Transaction Confirmed! https://sepolia.etherscan.io/tx/${receipt.hash}`
    );
  } catch (error) {
    console.error("An error occurred during token approval:", error);
    throw new Error("Token approval failed");
  }
}
```

- **Purpose**: Approves the specified token for the SwapRouter contract to spend on behalf of the user.
- **Logic**: Uses `ethers.Contract` to interact with the token contract and calls the `approve` function. Sends and waits for the transaction to be confirmed.

### 5. Get Pool Info Function

```javascript
async function getPoolInfo(factoryContract, tokenIn, tokenOut) {
  const poolAddress = await factoryContract.getPool(tokenIn.address, tokenOut.address, 3000);
  if (!poolAddress) {
    throw new Error("Failed to get pool address");
  }
  const poolContract = new ethers.Contract(poolAddress, POOL_ABI, provider);
  const [token0, token1, fee] = await Promise.all([
    poolContract.token0(),
    poolContract.token1(),
    poolContract.fee(),
  ]);
  return { poolContract, token0, token1, fee };
}
```

- **Purpose**: Retrieves the pool address from the Factory contract and fetches pool details.
- **Logic**: Calls the Factory contract to get the pool address, then initializes a contract instance for the pool and retrieves token and fee information.

### 6. Prepare Swap Params Function

```javascript
async function prepareSwapParams(poolContract, signer, amountIn) {
  return {
    tokenIn: USDC.address,
    tokenOut: LINK.address,
    fee: await poolContract.fee(),
    recipient: signer.address,
    amountIn: amountIn,
    amountOutMinimum: 0,
    sqrtPriceLimitX96: 0,
  };
}
```

- **Purpose**: Prepares the parameters required for executing the token swap.
- **Logic**: Uses the pool's fee and other details to set up the parameters for the `exactInputSingle` function in the SwapRouter.

### 7. Execute Swap Function

```javascript
async function executeSwap(swapRouter, params, signer) {
  const transaction = await swapRouter.exactInputSingle.populateTransaction(params);
  const receipt = await signer.sendTransaction(transaction);
  console.log(`-------------------------------`);
  console.log(`Receipt: https://sepolia.etherscan.io/tx/${receipt.hash}`);
  console.log(`-------------------------------`);
}
```

- **Purpose**: Executes the token swap using the SwapRouter contract.
- **Logic**: Creates a transaction using `populateTransaction`, sends it, and waits for confirmation.

### 8. Approve LP Tokens for Staking

```javascript
async function approveLPToken(tokenAddress, tokenABI, amount, wallet) {
  try {
    const tokenContract = new ethers.Contract(tokenAddress, tokenABI, wallet);
    const approveAmount = ethers.parseUnits(amount.toString(), LINK.decimals); // Assuming you're staking LINK LP tokens
    const approveTransaction = await tokenContract.approve.populateTransaction(
      MASTERCHEF_CONTRACT_ADDRESS,
      approveAmount
    );
    const transactionResponse = await wallet.sendTransaction(approveTransaction);
    console.log(`Approval Transaction Sent: ${transactionResponse.hash}`);
    const receipt = await transactionResponse.wait();
    console.log(
      `LP Token Approval Confirmed! https://sepolia.etherscan.io/tx/${receipt.hash}`
    );
  } catch (error) {
    console.error("An error occurred during LP token approval:", error);
    throw new Error("LP token approval failed");
  }
}
```

- **Purpose**: Approves the LP tokens for staking by the MasterChef contract.
- **Logic**: Similar to the token approval function, but for LP tokens.

### 9. Stake LP Tokens in MasterChef

```javascript
async function stakeLPTokens(amount, poolId, signer) {
  try {
    const masterChefContract = new ethers.Contract(
      MASTERCHEF_CONTRACT_ADDRESS,
      MASTERCHEF_ABI,
      signer
    );
    const stakeTransaction = await masterChefContract.deposit(
      poolId,
      ethers.parseUnits(amount.toString(), LINK.decimals)
    );
    console.log(`Staking Transaction Sent: ${stakeTransaction.hash}`);
    const receipt = await stakeTransaction.wait();
    console.log(
      `Staking Confirmed! https://sepolia.etherscan.io/tx/${receipt.hash}`
    );
  } catch (error) {
    console.error("An error occurred during staking:", error);
    throw new Error("Staking failed");
  }
}
```

- **Purpose**: Stakes LP tokens in the MasterChef contract.
- **Logic**: Calls the `deposit` function on the MasterChef contract with the specified poolId and amount of LP tokens.

### 10. Main Function

```javascript
async function main(swapAmount, stakeAmount, poolId) {
  const inputAmount = swapAmount;
  const amountIn = ethers.parseUnits(inputAmount.toString(), USDC.decimals);

  try {
    // Approve and swap USDC for LINK
    await approveToken(USDC.address, TOKEN_IN_ABI, inputAmount, signer);
    const { poolContract } = await getPoolInfo(factoryContract, USDC, LINK);
    const params = await prepareSwapParams(poolContract, signer, amountIn);
    const swapRouter = new ethers.Contract(
      SWAP_ROUTER_CONTRACT_ADDRESS,
      SWAP_ROUTER_ABI,
      signer
    );
    await executeSwap(swapRouter, params, signer);

    // Approve and stake LINK LP tokens
    await approveLPToken(LINK.address, TOKEN_IN_ABI, stakeAmount

, signer);
    await stakeLPTokens(stakeAmount, poolId, signer);
  } catch (error) {
    console.error("An error occurred:", error.message);
  }
}

// Example: Swap 1 USDC, then stake 0.5 LINK LP tokens in pool 1
main(1, 0.5, 1);
```

- **Purpose**: Coordinates the overall workflow of approving tokens, performing the swap, and staking LP tokens.
- **Logic**: Approves USDC, performs the swap for LINK, approves LINK LP tokens, and then stakes them in the MasterChef contract. Uses `try-catch` to handle errors and ensure the process completes successfully.

---