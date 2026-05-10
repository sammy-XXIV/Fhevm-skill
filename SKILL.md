# FHEVM Smart Contract Development — AI Agent Skill

## How to Use This Skill

**For AI agents (Claude Code, Cursor, Windsurf, Aider):**
Load this file as context before writing any FHEVM contract. Read all sections before generating code. Every anti-pattern here caused a real production failure — do not skip them.

**Mandatory pre-code checklist:**
- Read Section 4 (encrypted zero in constructor) before any contract
- Read Section 6 (ACL) before any function that stores encrypted values
- Read Section 9 (decryption) before any frontend decrypt flow
- Read Section 13 (anti-patterns) before finalizing any contract
- Read Section 39 (local bundle approach) before any browser encryption work
- Run `node fhevm-lint.js contracts/` after writing — fix all errors before deploying

**For developers:**
Give this file to your AI agent as a system prompt or project context. The agent will have full FHEVM proficiency for the session. Pair with `fhevm-lint.js` to audit agent output before Sepolia deployment.

**What this skill covers:**
Encrypted types · FHE operations · Access control · Input proofs · Decryption patterns · Frontend integration · Testing · Common anti-patterns · Production debugging · Known limitations · Local WASM bundle approach

---

## 1. Core Concepts

### What FHEVM Is

Zama FHEVM allows Solidity contracts to compute directly on encrypted data. Users encrypt values client-side. The contract processes ciphertexts without ever seeing plaintext. Results stay encrypted until the authorized user decrypts them via wallet signature.

### The Mental Model

```
User Browser → encrypt(amount) → ciphertext handle
                                        ↓
                              Confidential Contract
                                        ↓
                         FHE coprocessor computes on ciphertext
                                        ↓
                         Encrypted result stored onchain
                                        ↓
                    Only authorized user can decrypt via EIP-712
```

### What You Can and Cannot Do

**Can do:**
- Compute on encrypted integers: add, subtract, multiply, compare
- Store encrypted values in contract storage
- Grant/revoke access to encrypted handles
- Return encrypted handles to authorized users
- Use `FHE.select` for conditional logic on encrypted booleans

**Cannot do:**
- `require()` on an encrypted boolean — contract cannot branch on encrypted results
- Read encrypted values in plaintext inside the contract
- Use `FHE.div` — it does not exist in the library
- Subtract encrypted values without risk of underflow — use `FHE.min` first

---

## 2. Setup

### Installation

```bash
mkdir my-fhevm-project && cd my-fhevm-project
git clone https://github.com/zama-ai/fhevm-hardhat-template .
npm install
```

### Environment Variables

```bash
# .env
PRIVATE_KEY=your_deployer_private_key
SEPOLIA_RPC_URL=https://ethereum-sepolia-rpc.publicnode.com
```

### Hardhat Config

```typescript
import "@fhevm/hardhat-plugin";
import "@nomicfoundation/hardhat-ethers";
import type { HardhatUserConfig } from "hardhat/config";
import * as dotenv from "dotenv";
dotenv.config();

const config: HardhatUserConfig = {
  defaultNetwork: "hardhat",
  networks: {
    sepolia: {
      accounts: process.env.PRIVATE_KEY ? [process.env.PRIVATE_KEY] : [],
      chainId: 11155111,
      url: process.env.SEPOLIA_RPC_URL || "https://ethereum-sepolia-rpc.publicnode.com",
    },
  },
  solidity: {
    version: "0.8.24",
    settings: {
      optimizer: { enabled: true, runs: 800 },
      evmVersion: "cancun",
    },
  },
};

export default config;
```

---

## 3. Encrypted Types

### Available Types

```solidity
euint8    // encrypted 8-bit integer
euint16   // encrypted 16-bit integer
euint32   // encrypted 32-bit integer
euint64   // encrypted 64-bit integer — most common for token amounts
euint128  // encrypted 128-bit integer
euint256  // encrypted 256-bit integer
ebool     // encrypted boolean
eaddress  // encrypted address
ebytes64  // encrypted bytes

// External types — used for function parameters from users
externalEuint64  // user-supplied encrypted input
externalEbool    // user-supplied encrypted boolean
```

### Choosing the Right Type

- Token amounts with 8 decimals (cWETH) — use `euint64`
- Token amounts with 18 decimals — use `euint128` or `euint256`
- Boolean flags — use `ebool`
- Always use the smallest type that fits — lower gas cost

---

## 4. Contract Structure

### Required Imports

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, euint64, externalEuint64, ebool } from "@fhevm/solidity/lib/FHE.sol";
import { ZamaEthereumConfig } from "@fhevm/solidity/config/ZamaConfig.sol";
import { IERC7984 } from "@openzeppelin/confidential-contracts/interfaces/IERC7984.sol";
```

### Base Contract

Always inherit `ZamaEthereumConfig`:

```solidity
contract MyContract is ZamaEthereumConfig {
    // your code
}
```

### Storing Encrypted Zero — CRITICAL

Never use `FHE.asEuint64(0)` inline for comparisons. Inline calls create new handles without ACL permissions, causing unreliable comparison results.

Always store encrypted zero in the constructor:

```solidity
euint64 private _encryptedZero;

constructor() {
    _encryptedZero = FHE.asEuint64(0);
    FHE.allowThis(_encryptedZero);
}
```

Use `_encryptedZero` everywhere instead of `FHE.asEuint64(0)`.

---

## 5. FHE Operations

### Arithmetic

```solidity
euint64 sum  = FHE.add(a, b);
euint64 diff = FHE.sub(a, b);   // WARNING: underflow risk — use FHE.min first
euint64 prod = FHE.mul(a, b);
// FHE.div does NOT exist — rewrite as cross-multiplication
```

### Safe Subtraction Pattern

```solidity
// WRONG — underflow risk
euint64 result = FHE.sub(debt, repayAmount);

// CORRECT — cap repay at actual debt
euint64 actualRepay = FHE.min(repayAmount, debt);
euint64 result = FHE.sub(debt, actualRepay);
```

### Division Workaround — CRITICAL

`FHE.div` does not exist. Rewrite all division as cross-multiplication:

```solidity
// WRONG — will not compile
euint64 maxBorrow = FHE.div(FHE.mul(collateral, 66), 100);

// CORRECT — multiply both sides
ebool withinLTV = FHE.le(
    FHE.mul(debt, FHE.asEuint64(100)),
    FHE.mul(collateral, FHE.asEuint64(66))
);
```

### Comparisons

```solidity
ebool eq  = FHE.eq(a, b);
ebool ne  = FHE.ne(a, b);
ebool lt  = FHE.lt(a, b);
ebool le  = FHE.le(a, b);
ebool gt  = FHE.gt(a, b);
ebool ge  = FHE.ge(a, b);
```

### Boolean Operations

```solidity
ebool and = FHE.and(a, b);
ebool or  = FHE.or(a, b);
ebool not = FHE.not(a);
```

### Conditional Selection — replaces if/else on encrypted values

```solidity
// FHE.select(condition, valueIfTrue, valueIfFalse)
euint64 result = FHE.select(condition, trueValue, falseValue);

// Example: only transfer if condition is true, else transfer 0
euint64 toSend = FHE.select(isEligible, amount, _encryptedZero);
```

### Min/Max

```solidity
euint64 minimum = FHE.min(a, b);
euint64 maximum = FHE.max(a, b);
```

### Bit Operations

```solidity
euint64 shifted  = FHE.shl(value, FHE.asEuint64(3));
euint64 shifted  = FHE.shr(value, FHE.asEuint64(3));
euint64 rotated  = FHE.rotl(value, FHE.asEuint64(3));
euint64 rotated  = FHE.rotr(value, FHE.asEuint64(3));
```

### Random Numbers

```solidity
euint64 rand = FHE.randEuint64();
euint64 rand = FHE.randEuint64Bounded(uint64(100));
```

### Coming Soon — Do Not Use Yet

```solidity
FHE.div()       // Division — not available
FHE.rem()       // Remainder — not available
FHE.safeAdd()   // Safe add — not available
FHE.safeSub()   // Safe sub — not available
FHE.safeMul()   // Safe mul — not available
eint8…256       // Signed integers — not available
```

---

## 6. Access Control — ACL

Every encrypted handle must be explicitly granted permission before it can be used.

### Required Permissions

```solidity
FHE.allowThis(handle);                      // contract uses handle in future txs
FHE.allow(handle, userAddress);             // user can decrypt handle
FHE.allow(handle, address(contract));       // another contract can use handle
FHE.allowTransient(handle, address(token)); // token can use handle in same tx
```

### Full Pattern for Storing Encrypted Position

```solidity
function openPosition(externalEuint64 encryptedAmount, bytes calldata inputProof) external {
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);

    FHE.allowTransient(amount, address(collateralToken));
    euint64 received = collateralToken.confidentialTransferFrom(
        msg.sender, address(this), amount
    );

    _positions[msg.sender].collateral = received;

    FHE.allowThis(_positions[msg.sender].collateral);
    FHE.allow(_positions[msg.sender].collateral, msg.sender);
}
```

### Common ACL Mistakes

```solidity
// WRONG — forgot allowThis
_positions[msg.sender].collateral = received;

// WRONG — forgot allowTransient before token transfer
collateralToken.confidentialTransferFrom(msg.sender, address(this), amount);

// WRONG — forgot allow(user)
FHE.allowThis(handle);
// Missing: FHE.allow(handle, msg.sender);
```

---

## 7. Input Proofs

```solidity
function deposit(
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) external {
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
    // use amount in FHE operations
}
```

- Always validate via `FHE.fromExternal` — never skip
- `externalEuint64` is for user-supplied inputs only
- Internal `euint64` values cannot be passed as `externalEuint64`

---

## 8. Confidential Token (ERC-7984)

### Interface

```solidity
interface IERC7984 {
    function confidentialTransferFrom(address from, address to, euint64 amount) external returns (euint64);
    function confidentialTransfer(address to, euint64 amount) external returns (euint64);
    function confidentialBalanceOf(address account) external returns (euint64);
    function setOperator(address operator, uint48 until) external;
    function isOperator(address account, address operator) external view returns (bool);
}
```

### Critical: Capture Return Value

```solidity
// WRONG — trusts user-supplied amount
euint64 amount = FHE.fromExternal(encryptedAmount, proof);
collateralToken.confidentialTransferFrom(msg.sender, address(this), amount);
_positions[msg.sender].collateral = amount;

// CORRECT — cryptographically verified
euint64 amount = FHE.fromExternal(encryptedAmount, proof);
FHE.allowTransient(amount, address(collateralToken));
euint64 received = collateralToken.confidentialTransferFrom(
    msg.sender, address(this), amount
);
_positions[msg.sender].collateral = received;
```

### ERC-7984 Does NOT Support ERC20 approve()

```javascript
// WRONG — reverts silently
await cweth.approve(CONTRACT_ADDRESS, ethers.MaxUint256);

// CORRECT
const until = Math.floor(Date.now()/1000) + 365*24*60*60;
await cweth.setOperator(CONTRACT_ADDRESS, until);
```

### Setting Operator in Frontend

```javascript
const isApproved = await cweth.isOperator(userAddress, CONTRACT_ADDRESS);
if (!isApproved) {
    const until = Math.floor(Date.now()/1000) + 365*24*60*60;
    await (await cweth.setOperator(CONTRACT_ADDRESS, until)).wait();
}
```

---

## 9. Decryption — Backend-Mediated Pattern

### Why Frontend Decryption Fails

- CORS blocks requests to Zama's KMS/Gateway
- BigInt values cannot be JSON serialized
- `chainId` must be `Number` not `BigInt` — causes `InvalidTypeError createEIP712`
- Signature must have `0x` prefix stripped before passing to SDK

### Solution: Backend-Mediated Decryption

```
1. Frontend → POST /decrypt-prepare { handle, contractAddress, userAddress }
2. Backend generates keypair + EIP-712 message
3. Backend → Frontend: { keypair, eip712, startTimestamp, durationDays }
4. Frontend: user signs EIP-712 with wallet → signature
5. Frontend → POST /decrypt-balance { handle, contractAddress, userAddress, signature, keypair, startTimestamp, durationDays }
6. Backend calls instance.userDecrypt()
7. Backend → Frontend: { balance }
```

### Backend Decrypt Endpoints

```javascript
app.post('/decrypt-prepare', async (req, res) => {
  const { handle, contractAddress, userAddress } = req.body;
  try {
    const instance = await getInstance();
    const keypair = instance.generateKeypair();
    const startTimestamp = Math.floor(Date.now() / 1000);
    const durationDays = 10;

    const eip712 = instance.createEIP712(
      keypair.publicKey,
      [contractAddress],
      startTimestamp,
      durationDays,
    );

    // CRITICAL: serialize BigInt
    const serialize = (obj) => JSON.parse(
      JSON.stringify(obj, (_, v) => typeof v === 'bigint' ? v.toString() : v)
    );

    res.json({
      success: true,
      keypair: {
        publicKey: keypair.publicKey.toString(),
        privateKey: keypair.privateKey.toString(),
      },
      eip712: serialize(eip712),
      startTimestamp,
      durationDays,
    });
  } catch(err) {
    res.status(500).json({ error: err.message, success: false });
  }
});

app.post('/decrypt-balance', async (req, res) => {
  const { handle, contractAddress, userAddress, signature, keypair, startTimestamp, durationDays } = req.body;
  try {
    const instance = await getInstance();
    const result = await instance.userDecrypt(
      [{ handle, contractAddress }],
      keypair.privateKey,
      keypair.publicKey,
      signature.replace('0x', ''),  // CRITICAL: strip 0x prefix
      [contractAddress],
      userAddress,
      Number(startTimestamp),       // CRITICAL: must be Number not BigInt
      Number(durationDays),
    );

    const balance = result[handle];
    res.json({ success: true, balance: balance.toString() });
  } catch(err) {
    res.status(500).json({ error: err.message, success: false });
  }
});
```

### Frontend Signing Flow

```javascript
async function decryptBalance(handle, contractAddress, userAddress) {
  const prepRes = await fetch(`${BACKEND_URL}/decrypt-prepare`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ handle, contractAddress, userAddress }),
  });
  const { keypair, eip712, startTimestamp, durationDays } = await prepRes.json();

  const { domain, types: allTypes, message } = eip712;
  const { EIP712Domain: _, ...signTypes } = allTypes;

  // CRITICAL: chainId must be Number not BigInt
  const signature = await signer.signTypedData(
    { ...domain, chainId: Number(domain.chainId) },
    signTypes,
    message
  );

  const decRes = await fetch(`${BACKEND_URL}/decrypt-balance`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ handle, contractAddress, userAddress, signature, keypair, startTimestamp, durationDays }),
  });
  const { balance } = await decRes.json();
  return balance;
}
```

### Common Decrypt Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `InvalidTypeError createEIP712` | chainId is BigInt | `Number(domain.chainId)` |
| `signature must not include 0x` | ethers adds 0x prefix | `signature.replace('0x', '')` |
| `Cannot serialize BigInt` | keypair values are BigInt | serialize with custom replacer |
| `handle not found in result` | handle format mismatch | ensure same hex string in both calls |
| `ACL permission denied` | contract never called `FHE.allow` | add `FHE.allow(handle, userAddress)` in getter |

### Getting Handle from Contract

```solidity
// Cannot be view — FHE.allow modifies state
function getCollateralHandle(address user) external returns (euint64) {
    require(_positions[user].exists, "No position");
    FHE.allow(_positions[user].collateral, msg.sender);
    return _positions[user].collateral;
}
```

---

## 10. Deployment

### Deploy Script

```typescript
import { ethers } from "hardhat";

async function main() {
    const [deployer] = await ethers.getSigners();
    const MyContract = await ethers.getContractFactory("MyContract");
    const contract = await MyContract.deploy(CWETH_ADDRESS, CWETH_ADDRESS);
    await contract.waitForDeployment();
    console.log("Deployed at:", await contract.getAddress());
}

main().catch(console.error);
```

```bash
npx hardhat run deploy/script.ts --network sepolia
```

### CRITICAL: estimateGas Blocked on Sepolia

Use plain Node.js instead of Hardhat tasks post-deployment:

```javascript
const provider = new ethers.JsonRpcProvider("https://ethereum-sepolia-rpc.publicnode.com");
const signer = new ethers.Wallet(process.env.PRIVATE_KEY, provider);
const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer);

// Always set explicit gasLimit
const tx = await contract.someFunction(args, { gasLimit: 1_000_000 });
await tx.wait();
```

### CRITICAL: Clean Before Sepolia Deploy

```bash
npx hardhat clean
npx hardhat compile --network sepolia
npx hardhat run deploy/script.ts --network sepolia
```

Skipping `clean` causes internal exceptions on Sepolia.

---

## 11. Frontend Integration

### Install SDK

```bash
npm install @zama-fhe/relayer-sdk
```

**Prefer the local bundle approach (Section 39) for browser encryption.**

### Gas Limits

```javascript
{ gasLimit: 1_000_000n }   // deposits, simple ops
{ gasLimit: 2_000_000n }   // borrow, repay with FHE math
{ gasLimit: 3_000_000n }   // liquidation, multiple FHE ops
```

---

## 12. Testing Patterns

| Mode | Encryption | Speed | Usage |
|------|------------|-------|-------|
| Hardhat in-memory | Mock | Very fast | Unit tests, CI |
| Hardhat Node | Mock | Fast | Frontend integration |
| Sepolia | Real | Slow | Production validation |

```bash
# Local
npx hardhat test --network hardhat

# Sepolia
npx hardhat clean
npx hardhat compile --network sepolia
npx hardhat deploy --network sepolia
npx hardhat fhevm check-fhevm-compatibility --network sepolia --address <addr>
```

---

## 13. Common Anti-Patterns

### FHE.div does not exist

```solidity
// WILL NOT COMPILE
euint64 result = FHE.div(FHE.mul(a, 66), 100);
// Fix: cross-multiply
ebool check = FHE.le(FHE.mul(a, 100), FHE.mul(b, 66));
```

### Inline FHE.asEuint64(0) for comparisons

```solidity
// UNRELIABLE
ebool isZero = FHE.eq(debt, FHE.asEuint64(0));
// Fix: use _encryptedZero from constructor
```

### Forgetting allowTransient before token transfer

```solidity
// REVERTS
collateralToken.confidentialTransferFrom(msg.sender, address(this), amount);
// Fix: FHE.allowTransient(amount, address(collateralToken)) first
```

### Not capturing confidentialTransferFrom return value

```solidity
// ATTACK VECTOR
collateralToken.confidentialTransferFrom(msg.sender, address(this), amount);
_balance = amount; // user-supplied, not verified
// Fix: _balance = confidentialTransferFrom(...) return value
```

### require() on encrypted boolean

```solidity
// WILL NOT COMPILE
require(FHE.lt(debt, maxBorrow), "Exceeds LTV");
// Fix: FHE.select to return 0 silently
```

### view function with FHE.allow

```solidity
// COMPILE ERROR
function getHandle() external view returns (euint64) {
    FHE.allow(handle, msg.sender); // modifies state
}
// Fix: remove view modifier
```

### ERC20 approve on confidential token

```solidity
// REVERTS — ERC-7984 does not support approve()
cweth.approve(CONTRACT_ADDRESS, ethers.MaxUint256);
// Fix: use setOperator instead
```

---

## 14. Full Example Contract

```solidity
// SPDX-License-Identifier: BSD-3-Clause-Clear
pragma solidity ^0.8.24;

import { FHE, euint64, externalEuint64, ebool } from "@fhevm/solidity/lib/FHE.sol";
import { ZamaEthereumConfig } from "@fhevm/solidity/config/ZamaConfig.sol";
import { IERC7984 } from "@openzeppelin/confidential-contracts/interfaces/IERC7984.sol";

contract ConfidentialVault is ZamaEthereumConfig {

    IERC7984 public immutable token;
    euint64 private _encryptedZero;

    mapping(address => euint64) private _balances;
    mapping(address => bool)    public  hasDeposit;

    event Deposited(address indexed user);
    event Withdrawn(address indexed user);

    constructor(address _token) {
        token = IERC7984(_token);
        _encryptedZero = FHE.asEuint64(0);
        FHE.allowThis(_encryptedZero);
    }

    function deposit(externalEuint64 encryptedAmount, bytes calldata inputProof) external {
        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);

        FHE.allowTransient(amount, address(token));
        euint64 received = token.confidentialTransferFrom(msg.sender, address(this), amount);

        _balances[msg.sender] = FHE.add(_balances[msg.sender], received);
        hasDeposit[msg.sender] = true;

        FHE.allowThis(_balances[msg.sender]);
        FHE.allow(_balances[msg.sender], msg.sender);

        emit Deposited(msg.sender);
    }

    function withdraw(externalEuint64 encryptedAmount, bytes calldata inputProof) external {
        require(hasDeposit[msg.sender], "No deposit");

        euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
        euint64 actualWithdraw = FHE.min(amount, _balances[msg.sender]);
        _balances[msg.sender] = FHE.sub(_balances[msg.sender], actualWithdraw);

        FHE.allowThis(_balances[msg.sender]);
        FHE.allow(_balances[msg.sender], msg.sender);
        FHE.allow(actualWithdraw, address(token));

        token.confidentialTransfer(msg.sender, actualWithdraw);
        emit Withdrawn(msg.sender);
    }

    function getBalanceHandle(address user) external returns (euint64) {
        FHE.allow(_balances[user], msg.sender);
        return _balances[user];
    }
}
```

---

## 15. Useful Addresses (Sepolia)

| Contract | Address |
|----------|---------|
| cWETH (ERC-7984) | `0x46208622DA27d91db4f0393733C8BA082ed83158` |
| Underlying WETH | `0xff54739b16576FA5402F211D0b938469Ab9A5f3F` |
| ACL Contract | `0xf0Ffdc93b7E186bC2f8CB3dAA75D86d1930A433D` |
| KMS Contract | `0xbE0E383937d564D7FF0BC3b46c51f0bF8d5C311A` |
| Input Verifier | `0xBBC1fFCdc7C316aAAd72E807D9b0272BE8F84DA0` |
| Verifying Contract (Decryption) | `0x5D8BD78e2ea6bbE41f26dFe9fdaEAa349e077478` |
| Verifying Contract (Input Verification) | `0x483b9dE06E4E4C7D35CCf5837A1668487406D955` |

cWETH uses **8 decimals** not 18.

---

## 16. ABI Fragment Format for ethers.js

Use `bytes32` not `euint64` in ethers.js ABI strings:

```javascript
// WRONG
'function deposit(euint64 encryptedAmount, bytes inputProof) external'

// CORRECT
'function deposit(bytes32 encryptedAmount, bytes inputProof) external'
'function getBalanceHandle(address user) external returns (bytes32)'
```

---

## 17. cWETH Decimal Handling

cWETH uses 8 decimals. Using 18-decimal functions causes 10x display errors.

```javascript
const CWETH_DECIMALS = 8;

// Parsing input
const amountWei = ethers.parseUnits(amount.toString(), CWETH_DECIMALS);

// Displaying
const display = parseFloat(ethers.formatUnits(balance, CWETH_DECIMALS)).toFixed(4);
```

---

## 18. cWETH Faucet Pattern

```javascript
// Step 1: Mint underlying WETH
await weth.mint(signer.address, ethers.parseUnits('0.5', 18));

// Step 2: Approve WETH for cWETH contract
await weth.approve(CWETH_ADDRESS, ethers.parseUnits('0.5', 18));

// Step 3: Wrap to cWETH
await cweth.wrap(recipientAddress, ethers.parseUnits('0.5', 18));
```

---

## 19. SDK and Backend Integration

### SDK Import Paths

```javascript
// Node.js backend
const { createInstance, SepoliaConfig } = require('@zama-fhe/relayer-sdk/node');

// Browser — prefer local bundle approach (see Section 39)
import { createInstance, SepoliaConfig } from '@zama-fhe/relayer-sdk';

// Cloudflare Workers — DOES NOT WORK
// Use Node.js on Render instead
```

### Working RPC URL

```
https://ethereum-sepolia-rpc.publicnode.com  ✅
https://eth-sepolia.public.blastapi.io       ❌ 403 Forbidden
```

### SDK Version

Use `0.4.1`:
```bash
npm view @zama-fhe/relayer-sdk versions
```

### Full Working server.js

```javascript
const express = require('express');
const cors = require('cors');
const { createInstance, SepoliaConfig } = require('@zama-fhe/relayer-sdk/node');
const { ethers } = require('ethers');

const app = express();
app.use(cors());
app.use(express.json());

let _instance = null;
async function getInstance() {
  if (_instance) return _instance;
  _instance = await createInstance({
    ...SepoliaConfig,
    network: 'https://ethereum-sepolia-rpc.publicnode.com',
  });
  return _instance;
}

app.get('/health', (req, res) => res.json({ status: 'ok' }));

app.post('/encrypt', async (req, res) => {
  const { amount, contractAddress, userAddress } = req.body;
  if (!amount || !contractAddress || !userAddress)
    return res.status(400).json({ error: 'Missing fields' });
  try {
    const checksumContract = ethers.getAddress(contractAddress);
    const checksumUser = ethers.getAddress(userAddress);
    const instance = await getInstance();
    const encrypted = await instance
      .createEncryptedInput(checksumContract, checksumUser)
      .add64(BigInt(amount))
      .encrypt();
    const toHex = (val) => '0x' + Buffer.from(val).toString('hex');
    res.json({ handle: toHex(encrypted.handles[0]), inputProof: toHex(encrypted.inputProof), success: true });
  } catch(err) {
    res.status(500).json({ error: err.message, success: false });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Backend on port ${PORT}`));
```

---

## 20. Stale Handle Anti-Pattern — CRITICAL

Every FHE operation creates a NEW handle. Old ACL permissions do not transfer.

```solidity
// WRONG — new handle has no permissions
_positions[msg.sender].debt = FHE.add(_positions[msg.sender].debt, amount);

// CORRECT — re-grant after every FHE storage update
_positions[msg.sender].debt = FHE.add(_positions[msg.sender].debt, amount);
FHE.allowThis(_positions[msg.sender].debt);
FHE.allow(_positions[msg.sender].debt, msg.sender);
```

Rule: after every line that updates encrypted storage — immediately re-grant all permissions.

---

## 21. Known Limitations and Workarounds

### Cannot branch on encrypted results

```solidity
// IMPOSSIBLE
require(FHE.ge(collateral, debt));

// WORKAROUND
euint64 actualBorrow = FHE.select(withinLTV, amount, _encryptedZero);
```

### No oracle in pure FHE lending

```solidity
ebool isLiquidatable = FHE.lt(
    FHE.mul(collateral, FHE.asEuint64(100)),
    FHE.mul(debt, FHE.asEuint64(150))
);
```

---

## 22. Remix-Specific Notes

```solidity
// Remix only
import "https://raw.githubusercontent.com/zama-ai/fhevm/refs/heads/main/lib/FHE.sol";
import "https://raw.githubusercontent.com/zama-ai/fhevm/refs/heads/main/config/ZamaConfig.sol";
```

---

## 23. Render Free Tier Cold Start

```javascript
async function wakeBackend() {
  try { await fetch(`${BACKEND_URL}/health`); } catch(e) {}
}
wakeBackend(); // call immediately on page load
```

---

## 24. missing revert data Error

Most common causes:
1. `setOperator` not called
2. `confidentialTransferFrom` failed — no cWETH balance
3. Position already exists
4. Pool empty on borrow

```javascript
const isOp = await cweth.isOperator(userAddress, CONTRACT_ADDRESS);
console.log('isOperator:', isOp); // false = root cause
```

---

## 25. Complete Operations Reference

```solidity
euint8, euint16, euint32, euint64, euint128, euint256
ebool, eaddress
ebytes1, ebytes4, ebytes8, ebytes16, ebytes32, ebytes64, ebytes128, ebytes256
```

---

## 26. Quick Reference

| Task | Code |
|------|------|
| Import FHE | `import { FHE, euint64, externalEuint64, ebool } from "@fhevm/solidity/lib/FHE.sol"` |
| Inherit config | `contract MyContract is ZamaEthereumConfig` |
| Decode user input | `euint64 amt = FHE.fromExternal(encAmt, proof)` |
| Allow contract | `FHE.allowThis(handle)` |
| Allow user | `FHE.allow(handle, userAddress)` |
| Allow token transfer | `FHE.allowTransient(handle, address(token))` |
| Safe subtraction | `FHE.sub(a, FHE.min(b, a))` |
| Division workaround | `FHE.le(FHE.mul(a,100), FHE.mul(b,66))` |
| Conditional transfer | `FHE.select(condition, amount, _encryptedZero)` |
| Parse cWETH amount | `ethers.parseUnits(amount, 8)` |
| Format cWETH amount | `ethers.formatUnits(balance, 8)` |
| Browser encryption | Use local bundle (Section 39) not npm package |

---

*Built from real production bugs on Zama FHEVM Sepolia testnet.*
*Every anti-pattern here caused a real failure in production.*

---

## 27. Testing Checklist — FHEVM Contracts on Sepolia

### Pre-Deploy

```bash
npx hardhat clean
npx hardhat compile --network sepolia
npx hardhat fhevm check-fhevm-compatibility --network sepolia --address <addr>
```

### Frontend Test Flow

```
1. Connect wallet on Sepolia
2. Get cWETH — Mint WETH → Approve → Wrap
3. setOperator — check isOperator returns true
4. Deposit cWETH — check hasPosition returns true
5. Decrypt collateral — sign EIP-712, verify value
6. Borrow within 66% LTV — verify cWETH in wallet
7. Decrypt debt — verify matches borrow amount
8. Repay — verify debt shows 0 after decrypt
9. Close position — verify collateral returned
```

### Debugging Silent Failures

```javascript
const isOp = await cweth.isOperator(userAddress, CONTRACT);
if (!isOp) console.error("ROOT CAUSE: setOperator not called");

try {
  await contract.openPosition.staticCall(handle, proof, { from: userAddress });
} catch(e) {
  console.error("staticCall failed:", e.message);
}
```

---

## 28. Complete Error Lookup Table

| Error Message | Cause | Fix |
|---------------|-------|-----|
| `FHE.div is not a function` | `FHE.div` doesn't exist | Cross-multiply instead |
| `Function cannot be declared as view` | `FHE.allow` called inside `view` | Remove `view` modifier |
| `missing revert data (data=null, reason=null)` | Contract reverted silently | Check `isOperator`, `hasPosition`, token balance |
| `InvalidTypeError createEIP712` | `chainId` passed as BigInt | `Number(domain.chainId)` |
| `signature must not include 0x prefix` | Ethers adds `0x` to signatures | `signature.replace('0x', '')` |
| `Cannot serialize BigInt` | Keypair has BigInt values | Serialize with custom JSON replacer |
| `handle not found in result` | Handle format mismatch | Use same hex string in both calls |
| `ACL permission denied` | Contract never called `FHE.allow(handle, user)` | Add `FHE.allow` in getter function |
| `no matching fragment` | Wrong ABI — using `euint64` not `bytes32` | Use `bytes32` in ethers.js ABI strings |
| `Contract address is not a valid address` | Lowercase address passed to SDK | `ethers.getAddress(contractAddress)` |
| `estimateGas failed` | FHEVM plugin blocks gas estimation on Sepolia | Use plain Node.js with explicit `gasLimit` |
| `ENOTFOUND relayer.testnet.zama.cloud` | Dead relayer URL | Use `https://relayer.testnet.zama.org` |
| `Impossible to fetch public key` | Wrong relayer URL or contract addresses | Use local bundle (Section 39) |
| `Cannot read properties of undefined (_wbindgen_malloc)` | WASM not loaded — relayer URL wrong | Use local bundle (Section 39) |
| `403 Forbidden` | Blocked RPC (`blastapi.io`) | Use `publicnode.com` RPC |
| `No such module @zama-fhe/relayer-sdk` | Cloudflare Workers can't import SDK | Use Node.js backend on Render |
| `approve() reverts on cWETH` | ERC-7984 doesn't support `approve()` | Use `setOperator()` instead |
| `balance shows 0 after deposit` | Stale handle — ACL permissions not re-granted | `FHE.allowThis` + `FHE.allow` after every FHE op |
| `FHE.eq returns wrong result` | Inline `FHE.asEuint64(0)` has no ACL | Use `_encryptedZero` from constructor |
| `wrap() fails` | WETH not approved before wrapping | `weth.approve(CWETH_ADDRESS, amount)` first |
| `amount display is 10x wrong` | Using 18 decimals for 8-decimal cWETH | `ethers.parseUnits(amount, 8)` not `parseEther` |
| `Etherscan verification fails` | FHEVM plugin transforms bytecode | Cannot verify — link GitHub source instead |

---

## 29-38. [Sections unchanged — deployment scripts, TypeScript config, React template, OpenZeppelin contracts, public decryption, token registry, project bootstrap, test templates, frontend template, relayer URL history]

---

## 39. Local WASM Bundle Approach — RECOMMENDED FOR BROWSER

**This is the most reliable way to do browser encryption.** The npm package approach fails when the relayer URL changes or WASM files can't be fetched. The local bundle serves everything from your own domain.

### Why the npm Approach Fails in the Browser

When you import `@zama-fhe/relayer-sdk/web` in a React/Vite app:
- The SDK tries to fetch WASM files from the relayer URL
- If the relayer URL is wrong or dead — WASM never loads
- CORS blocks requests to Zama's KMS from the browser
- Result: `Cannot read properties of undefined (reading '_wbindgen_malloc')` or silent failure

### Why the Local Bundle Works

The local `sdk-bundle.js` is a pre-built single file that:
- Loads `tfhe_bg.wasm` and `kms_lib_bg.wasm` from your own domain via `fetch()`
- No relayer URL needed for WASM initialization
- Still uses `SepoliaConfig` for contract addresses and KMS operations
- No npm install, no build-time WASM issues, no CORS

### The 5 Files You Need

Place all 5 in `frontend/public/` so they're served as static assets:

```
frontend/public/
├── sdk-bundle.js      # pre-built SDK — all JS bundled
├── sdk.js             # SDK entry point
├── tfhe_bg.wasm       # FHE crypto engine
├── kms_lib_bg.wasm    # KMS library
└── workerHelpers.js   # web worker for heavy WASM computation
```

Confirmed working versions (extracted from `sdk-bundle.js`):
- fhevmjs: `0.4.1`
- ethers: `6.16.0`

### React/Vite Integration — App.jsx Pattern

```javascript
// REMOVE this static import:
// import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/web";

// ADD this dynamic loader:
let createInstance = null;
let SepoliaConfig  = null;

async function loadSDK() {
  if (createInstance) return; // already loaded
  const m = await import("./sdk-bundle.js");
  createInstance = m.createInstance;
  SepoliaConfig  = m.SepoliaConfig;
  if (m.initSDK) await m.initSDK(); // CRITICAL: call initSDK if present
}
```

Update `getFhevmInstance()`:

```javascript
async function getFhevmInstance() {
  if (fhevmRef.current) return fhevmRef.current;
  await loadSDK();
  fhevmRef.current = await createInstance({
    ...SepoliaConfig,
    network: "https://ethereum-sepolia-rpc.publicnode.com",
  });
  return fhevmRef.current;
}
```

### Plain HTML/JS Integration

```javascript
let createInstance, SepoliaConfig, instance;

async function initFHEVM() {
  const m = await import('./sdk-bundle.js');
  createInstance = m.createInstance;
  SepoliaConfig  = m.SepoliaConfig;
  if (m.initSDK) await m.initSDK();

  instance = await createInstance({
    ...SepoliaConfig,
    network: "https://ethereum-sepolia-rpc.publicnode.com",
  });
}

await initFHEVM();
```

### Encrypting After SDK Load

```javascript
const addr = ethers.getAddress(CONTRACT_ADDRESS);
const user = ethers.getAddress(userAddress);

const encrypted = await instance
  .createEncryptedInput(addr, user)
  .add64(BigInt(amount))
  .encrypt();

function toHex(val) {
  if (typeof val === "string" && val.startsWith("0x")) return val;
  if (val instanceof Uint8Array || Array.isArray(val))
    return "0x" + Array.from(val).map(b => b.toString(16).padStart(2, "0")).join("");
  return String(val);
}

const handle     = toHex(encrypted.handles[0]);
const inputProof = toHex(encrypted.inputProof);

await contract.deposit(handle, inputProof, { gasLimit: 1_000_000n });
```

### Confirmed Working SepoliaConfig (from sdk-bundle.js v0.4.1)

```javascript
{
  aclContractAddress:                        "0xf0Ffdc93b7E186bC2f8CB3dAA75D86d1930A433D",
  kmsContractAddress:                        "0xbE0E383937d564D7FF0BC3b46c51f0bF8d5C311A",
  inputVerifierContractAddress:              "0xBBC1fFCdc7C316aAAd72E807D9b0272BE8F84DA0",
  verifyingContractAddressDecryption:        "0x5D8BD78e2ea6bbE41f26dFe9fdaEAa349e077478",
  verifyingContractAddressInputVerification: "0x483b9dE06E4E4C7D35CCf5837A1668487406D955",
  chainId: 11155111,
  gatewayChainId: 10901,
  relayerUrl: "https://relayer.testnet.zama.org"
}
```

These match the currently deployed Sepolia FHEVM contracts. If browser encryption produces wrong handles or ACL errors — contract addresses are likely mismatched.

### Anti-Patterns — Local Bundle

```javascript
// WRONG — npm import tries to fetch WASM from relayer
import { createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/web";

// WRONG — skipping initSDK
const m = await import('./sdk-bundle.js');
const instance = await m.createInstance({ ...m.SepoliaConfig, network: RPC });
// Missing: if (m.initSDK) await m.initSDK();

// WRONG — using window.ethereum as network (use RPC URL instead)
fhevmRef.current = await createInstance({ ...SepoliaConfig, network: window.ethereum });

// CORRECT
await loadSDK();
fhevmRef.current = await createInstance({
  ...SepoliaConfig,
  network: "https://ethereum-sepolia-rpc.publicnode.com",
});
```

### Deployment Checklist

```
✓ All 5 files in frontend/public/
  - sdk-bundle.js
  - sdk.js
  - tfhe_bg.wasm
  - kms_lib_bg.wasm
  - workerHelpers.js
✓ Static import removed — using dynamic import('./sdk-bundle.js')
✓ loadSDK() called before createInstance()
✓ initSDK() called if present in the bundle
✓ network set to publicnode RPC URL, not window.ethereum
✓ ethers.getAddress() used on contract and user addresses
✓ toHex() used on handles[0] and inputProof before sending to contract
```

