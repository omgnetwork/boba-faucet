# Boba Faucet

- [Overview](#Overview)
- [Specification](#Specification)
- [Impementation](#Implementaion)
  - [Step 1: Creating API endpoints](#Step1: Creating API endpoints)
  - [Step2: Creating Boba Faucet Contract](#Step2: Creating Boba Faucet Contract)
  - [Step3: Funding Turing Helper Contract](#Step3: Funding Turing Helper Contract)

## Overview

Boba Faucet is a system for getting Boba Rinkeby ETH and Boba Rinkeby Boba Token. It's implemented using the Turing. Turing is a system for interacting with the outside world from within solidity smart contracts.

Before claiming the token, users have to answer the CAPTCHA. The answer is hashed and compared off-chain via Turing. Once the answer is verified, the smart contract releases the funds.

## Specification

This procedure takes place in five steps:

1. User queries the CAPTCHA image and UUID
2. User sends the transaction to the contract with UUID and CAPTCHA answer
3. Geth sends the request to backend and retrieves the result
4. Geth atomically revises the calldata
5. User gets the result

## Implementation

### Step1: Creating API endpoints

Two simple API endpoints are created.

#### get.captcha

```python
image = ImageCaptcha(width=280, height=90)

# For image
uuid1 = uuid.uuid4()
imageStr = str(uuid1).split('-')[0]
imageStrBytes = Web3.solidityKeccak(['string'], [str(imageStr)]).hex()

# For key
uuid2 = uuid.uuid4()
keyBytes = Web3.solidityKeccak(['string'], [str(uuid2)]).hex()

# Use the first random string
imageData = image.generate(imageStr)

# Get base64
imageBase64 = base64.b64encode(imageData.getvalue())

# Read YML
with open("env.yml", 'r') as ymlfile:
config = yaml.load(ymlfile)

# Connect to Redis Elasticache
db = redis.StrictRedis(host=config.get('REDIS_URL'),
port=config.get('REDIS_PORT'))

# Store and set the expire time as 10 minutes
print('keyBytes: ', keyBytes, 'imageStrBytes: ', imageStrBytes, 'imageStr: ', imageStr)
db.set(keyBytes, imageStrBytes, ex=600)

payload = {'uuid': keyBytes, 'imageBase64': imageBase64.decode('utf-8') }
```

#### verify.captcha

```python
paramsHexString = body['params'][0]

uuid = '0x' + paramsHexString[66: 130]
key = '0x' + paramsHexString[130:]

# Read YML
with open("env.yml", 'r') as ymlfile:
  config = yaml.load(ymlfile)

  # Connect to Redis Elasticache
  db = redis.StrictRedis(host=config.get('REDIS_URL'),
                         port=config.get('REDIS_PORT'))

  # Store and set the expire time as 10 minutes
  keyInRedis = db.get(uuid)

  if keyInRedis:
    print('keyInRedis: ', keyInRedis.decode('utf-8'), 'key: ', key)
    return returnPayload(keyInRedis.decode('utf-8') == key)
  else:
    return returnPayload(False)
```

### Step2: Creating Boba Faucet Contract

The smart contract imports [Turing Helper Contract](https://github.com/omgnetwork/optimism-v2/blob/develop/packages/boba/turing/contracts/TuringHelper.sol), so it can interact with outside API endpoints.

```solidity
import './TuringHelper.sol';

contract BobaFaucet is Ownable {
    // Turing
    address public turingHelperAddress;
    string public turingUrl;
    TuringHelper public turing;
    
    constructor(
        address _turingHelperAddress,
        string memory _turingUrl
    ) {
        turingHelperAddress = _turingHelperAddress;
        turing = TuringHelper(_turingHelperAddress);
        turingUrl = _turingUrl;
    }
    
    function getBobaFaucet(
        bytes32 _uuid,
        string memory _key
    ) external {
        require(BobaClaimRecords[msg.sender] + waitingPeriod < block.timestamp, 'Invalid request');
				
				// The key is hashed
        bytes32 hashedKey = keccak256(abi.encodePacked(_key));
        uint256 result = _verifyKey(_uuid, hashedKey);

        require(result == 1, 'Invalid key and UUID');

        BobaClaimRecords[msg.sender] = block.timestamp;
        IERC20(BobaAddress).safeTransfer(msg.sender, BobaFaucetAmount);

        emit GetBobaFaucet(_uuid, hashedKey, msg.sender, BobaFaucetAmount, block.timestamp);
    }
    
    // Call Turing contract to get the result from the outside API endpoint
    function _verifyKey(bytes32 _uuid, bytes32 _key) private returns (uint256) {
        bytes memory encRequest = abi.encodePacked(_uuid, _key);
        bytes memory encResponse = turing.TuringTx(turingUrl, encRequest);

        uint256 result = abi.decode(encResponse,(uint256));

        emit VerifyKey(turingUrl, _uuid, _key, result);

        return result;
    }
}
```

### Step3: Funding Turing Helper Contract

We charge 0.01 BOBA for each Turing request and it's based on the Turing Helper Contract.

```js
// Deploy Turing Helper Contract
const Factory__TuringHelperr = new ContractFactory(
  TuringHelperJson.abi,
  TuringHelperJson.bytecode,
  (hre as any).deployConfig.deployer_l2
)

const TuringHelper = await Factory__TuringHelperr.deploy()
await TuringHelper.deployTransaction.wait()

console.log(`TuringHelper was deployed at: ${TuringHelper.address}`)

// Approve Boba Token to add funds to Turing Credit Contract
const BobaTuringCredit = new Contract(
  (hre as any).deployConfig.BobaTuringCreditAddress,
  BobaTuringCreditJson.abi,
  (hre as any).deployConfig.deployer_l2
)
const approveTx = await L2BobaToken.approve(
  BobaTuringCredit.address,
  utils.parseEther('100')
)
await approveTx.wait()

const addCreditTx = await BobaTuringCredit.addBalanceTo(
  utils.parseEther('100'),
  TuringHelper.address
)
await addCreditTx.wait()

// Add permission for Boba Faucet contract
const addPermissionTx = await TuringHelper.addPermittedCaller(
  BobaFaucet.address
)
await addPermissionTx.wait()
```

