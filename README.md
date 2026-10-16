# COMP7810A Assignment 3: RWA Tokenization

## 1. Student information

- Name: LUI Chi Shing
- Student ID: 3036326931
- Course: COMP7810A
- Assignment: 3
- Repository: https://github.com/HaroldLui/COMP7810A-Assignment3.git

## 2. Tests and submitted work

### `make dotor`

![make test screenshot](docs%20%28SUBMISSION%20MATERIALS%29/%22make%20dotor%22_screenshot.png)

### `make exercise`

![make exercise screenshot](docs%20%28SUBMISSION%20MATERIALS%29/%22make%20exercise%22_screenshot.png)

### `make test`

![make test screenshot](docs%20%28SUBMISSION%20MATERIALS%29/%22make%20test%22_screenshot.png)

## 3. Architecture and explanation

The main idea is that TBill represents a fund share, because its value changes with NAV. The number of shares can stay the same while its USDC value changes. On-chain accounting cannot prove that the real TBills exist.

![RWA architecture](<docs (SUBMISSION MATERIALS)/TBillVault_Architecture_Diagram.png>)

**Custody:** subscribing puts USDC in the vault and creates shares. The operator can invest the buffer. The mock custodian only records a number; the SPV and real TBills shown here are the intended off chain arrangement, not assets created.

**Attestation:** NAV decides how many shares a subscription buys and how much a redemption requests. The mock feed's public `setPrice()` is a bypass of the reporter check, so this deployment is a teaching demo. The discussion answers explain that risk.

**Redemption:** enqueue transfers shares into escrow and fixes the payout. Settlement pays whole tickets in FIFO order when the budget and vault cash are sufficient, burns their shares, and makes USDC claimable. The investor then claims. T+1 is the intended off-chain delay, this contract does not enforce a one day timer.

**Admission:** both nonzero endpoints need to be whitelisted when TBILL balances change. This applies to transfers, minting and burning. Removing a holder therefore blocks both transfer and burn.

---

## 4. Deployment (Sepolia)

All seven contracts are deployed on Sepolia and the source code is verified on Etherscan.
The deployment also sets up the roles and whitelist, because the vault and queue need these permissions to work together.

- Network: Sepolia (chain ID `11155111`)
- Deployment date: 16 October 2026
- Test wallet / admin: `0x7BF7242a18C218A604196B78c85aA89bBeb2a8F2`
- Compiler: Solidity `0.8.24`, optimizer enabled with `200` runs
- Result: all 13 deployment and setup transactions succeeded; all seven contracts verified.

| Contract | Sepolia address | Verified source |
| --- | --- | --- |
| MockUSDC | `0xe92062ac635b2f3b3d44826f424265daec25a521` | [Etherscan](https://sepolia.etherscan.io/address/0xe92062ac635b2f3b3d44826f424265daec25a521#code) |
| ComplianceRegistry | `0xc088b5d487fc3930c859ff0720797553bbbd4e9a` | [Etherscan](https://sepolia.etherscan.io/address/0xc088b5d487fc3930c859ff0720797553bbbd4e9a#code) |
| MockPriceFeed | `0xce178372c082910962266cfbe0911a0380f3f44d` | [Etherscan](https://sepolia.etherscan.io/address/0xce178372c082910962266cfbe0911a0380f3f44d#code) |
| TBillToken | `0x620330b79f37b29653e11bd0e67022261392afeb` | [Etherscan](https://sepolia.etherscan.io/address/0x620330b79f37b29653e11bd0e67022261392afeb#code) |
| MockTBillCustodian | `0x63d9f975c1b9d9a57ccc0c017baeb4eade61910e` | [Etherscan](https://sepolia.etherscan.io/address/0x63d9f975c1b9d9a57ccc0c017baeb4eade61910e#code) |
| TBillVault | `0x88ff770a050b479f64ba8db74909196fc0f04066` | [Etherscan](https://sepolia.etherscan.io/address/0x88ff770a050b479f64ba8db74909196fc0f04066#code) |
| RedemptionQueue | `0xce199a81ed679b1d95b7563051b1c640178c8113` | [Etherscan](https://sepolia.etherscan.io/address/0xce199a81ed679b1d95b7563051b1c640178c8113#code) |
