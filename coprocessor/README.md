<p align="center">
<!-- product name logo -->
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../assets/Zama-KMS-fhEVM-dark.png">
  <source media="(prefers-color-scheme: light)" srcset="../assets/Zama-KMS-fhEVM-light.png">
  <img width=600 alt="Zama fhEVM & KMS">
</picture>
</p>

---

<p align="center">
  <a href="./fhevm-whitepaper.pdf"> 📃 Read white paper</a> |<a href="https://docs.zama.ai/fhevm"> 📒 Documentation</a> | <a href="https://zama.ai/community"> 💛 Community support</a> | <a href="https://github.com/zama-ai/awesome-zama"> 📚 FHE resources by Zama</a>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-BSD--3--Clause--Clear-%23ffb243?style=flat-square"></a>
  <a href="https://github.com/zama-ai/bounty-program"><img src="https://img.shields.io/badge/Contribute-Zama%20Bounty%20Program-%23ffd208?style=flat-square"></a>
</p>

# What does the setup do ?

It provides a way to run a co-processor, gateway and kms (in centralized/threshold mode)
using docker compose.

The pre-computed contracts and account addresses/private keys are configured
across all the components for this test setup.

> ⚠️ **Warning**
> This repository is intended for demonstration purposes only.
> **Do not use any secrets** included here, such as those for the coprocessor API or wallets, in any production or serious deployment.
> Secrets in this repository are hard-coded solely to simplify the setup process for demonstration.
> In any real-world deployment, secrets must be managed securely—**never hard-coded in configuration files!**

## How the repository is organized ?

- The docker compose folder contains all the files for KMS and coprocessor.

- fhEVM solidity repository is cloned to work_dir and used to run tests

- All the components are dockerized, a tutorial explains also how to run it from source (if necessary)

### Install Binaries

Make sure you have these binaries installed in your environment:

- [docker](https://docs.docker.com/engine/install)
- [nodejs v20](https://nodejs.org/en/download/package-manager)

> **NB:** We will wrap all components in docker images to remove these prerequisites in the near future.

## Fast start

If you only want to see a running setup and trigger some tests, follow the instructions.

### Centralized KMS

```bash
# Ensure in .env that CENTRALIZED_KMS=true
make clean
./scripts/run_everything.sh
# To stop
make stop
```

### Threshold KMS

```bash
# Ensure in .env that CENTRALIZED_KMS=false
make clean
./scripts/run_everything.sh
# To stop
make stop
```

- The different steps are:
  - Running the coprocessor + db + geth
  - Running the KMS blockchain with KMS core(s)
  - Deploying the different KMS blockchain smart contracts via `zama-setup-kms-blockchain-contracts` docker image
  - Running KMS connector(s)
  - Triggering a key-gen and crs-gen via `zama-setup-kms-key-crs` docker image
  - Updating the CRS and key ids in gateway config + running the gateway (which need a working geth node)
  - Retrieving the fhe keys + crs and give them to coprocessor via `zama-setup-fhevm-coprocessor-db` docker image
  - Deploying fhEVM smart contracts via `zama-setup-fhevm-contracts` docker image
  - Run a a trivial decryption test

## Steps setting up the environment (detailed)

### 0. Set the KMS mode

Verify the configuration in .env file, most important variable is CENTRALIZED_KMS, set it to `false` for threshold KMS.

| CENTRALIZED_KMS | Purpose                                                                                                               |
| --------------- | --------------------------------------------------------------------------------------------------------------------- |
| true (default)  | KMS is running in centralized mode, keys are generated by one KMS party and signer is automatically updated  |
| false           | KMS is running in threshold mode with 4 MPC nodes, keys are freshly generated and signers are automatically updated   |

Note: **Based on the mode the command will trigger different configurations transparently!**

### 1. Launch almost everything

```bash
make run-kms
```

The different steps are:

- Running the coprocessor + db + geth
- Running the KMS blockchain with KMS core(s)
- Deploying the different KMS blockchain smart contracts via `zama-setup-kms-blockchain-contracts` docker image
- Running KMS connector(s)
- Triggering a key-gen and crs-gen via `zama-setup-kms-key-crs` docker image
- Updating the CRS and key ids in gateway config + running the gateway (which need a working geth node)

📝 At this step keys are not loaded in coprocessor DB.

<details><summary> 💡 Why are we starting the coprocessor if keys are not available ? </summary>

We have to do it to satisfy the gateway, gateway needs to conenct to the host BC node (geth here) in order to listen events.

We need the gateway (1) to be able to call `\keyurl` endpoint in order to retrieve the identifiers associated to each keys (publicKey, serverKey, CRS ...).

Then (2) we download keys (with identifiers) from minio (S3 bucket like storage), see [point 2](#2-retrieve-public-key-material)

</details>

<details>
<summary> 💡 Follow KMS blockchain smart contracts deployment  (__first step__)~ 2 mn </summary>

```bash
  $docker logs zama-setup-kms-blockchain-contracts-1 -f

  Summary of all the addresses:
  IPSC_ETHERMINT_ADDRESS : wasm1wug8sewp6cedgkmrmvhl3lf3tulagm9hnvy8p0rppz9yjw0g4wtqhs9hr8
  IPSC_ETHEREUM_ADDRESS : wasm1qg5ega6dykkxc307y25pecuufrjkxkaggkkxh7nad0vhyhtuhw3sq29c3m
  ASC_DEBUG_ADDRESS : wasm1yyca08xqdgvjz0psg56z67ejh9xms6l436u8y58m82npdqqhmmtqas0cl7
  ASC_ETHERMINT_ADDRESS : wasm1yw4xvtc43me9scqfr2jr2gzvcxd3a9y4eq7gaukreugw2yd2f8tsu3v7ad
  ASC_ETHEREUM_ADDRESS : wasm1cnuw3f076wgdyahssdkd0g3nr96ckq8cwa2mh029fn5mgf2fmcms9ax00l

  +++++++++++++++++++++++++++
  Contracts setups successful
  +++++++++++++++++++++++++++
```

</details>

<details><summary> 💡 Follow key and crs genereation ~ 2 mn </summary>

This docker container is starting only after the deployment step (above)

```bash
$docker logs zama-setup-kms-key-crs -f
Launching insecure key-gen
[
  {
    "keygen_response": {
      "request_id": "53108da2fd6fe8947f4df0eacd083fa759617567",
      "public_key_digest": "4082a86a0e0d3db82a7f767d5eed54f20c9d6cfa",
      "public_key_signature": "40000000000000005a7815b61aa3bd034b2835b0305e4d9c21a9ef45521ebb547e3f3013c627ccf346dd929be3ad466e60816ec163cc8395597560bb6e9547d69f39b15e713cdb56",
      "server_key_digest": "7c210349453c4d29b9d852dbdf6e622973e03804",
      "server_key_signature": "400000000000000024fec74ee2cb706924552f9404531f5696a426c6cdcabd1f7b47fa966cc0b6a9167f9f0dffdc9cead38943b2b04030677ca472c18583543c28e8dc2f8f2f16dd",
      "param": "default"
    }
  }
]
Launching crs-gen
[
  {
    "crs_gen_response": {
      "request_id": "ea9ce765d560786c8bf0824e2126cb5692f36cd7",
      "digest": "ddc523e087d364a0bc69f72c51b4f622fd1ce872",
      "signature": "400000000000000099e84dafd499bd3cfa0ff06e10e5d07c58e4b80f1f4073b3114843efb87d8c5666c56bab26eb93b2eac84ea6298c29c038d5c20355b3a3dce70649a8d7f74568",
      "max_num_bits": 2048,
      "param": "default"
    }
  }
]
Success
```

</details>

<details><summary> 💡 Curious to see the tx going through the connector ? </summary>

```bash
# Threshold KMS
docker logs zama-kms-connector-1 > log_connector.txt 2>&1  &&  grep crsgen log_connector.txt -i
# Centralized KMS
docker logs zama-kms-connector-1 > log_connector.txt 2>&1  &&  grep keygen log_connector.txt -i
```

Typical output, highlights on:

- `Running KMS operation with value: InsecureKeyGen`
- `Sending response to the blockchain: KeyGenResponse`

```bash
2024-11-15T10:18:39.083655Z  INFO kms_blockchain_connector: Starting kms-asc-connector in mode 'KmsCore' - config ConnectorConfig { tick_interval_secs: 1, storage_path: "./temp/events.toml", tracing: Some(Tracing { service_name: "kms-asc-connector", endpoint: Some("http://localhost:4317"), batch: None, json_logs: None }), blockchain: BlockchainConfig { addresses: ["http://dev-kms-blockchain-validator:9090"], contract: "wasm1cnuw3f076wgdyahssdkd0g3nr96ckq8cwa2mh029fn5mgf2fmcms9ax00l", fee: ContractFee { amount: 3000000, denom: "ucosm" }, signkey: SignKeyConfig { mnemonic: "<REDACTED>", bip32: "<REDACTED>" }, kv_store_address: None }, core: CoreConfig { addresses: ["http://dev-kms-core:50051"], timeout_config: TimeoutConfig { channel_timeout: 60, crs: TimeoutTriple { initial_wait_time: 10, retry_interval: 10, max_poll_count: 200 }, keygen: TimeoutTriple { initial_wait_time: 18000, retry_interval: 15000, max_poll_count: 1150 }, insecure_key_gen: TimeoutTriple { initial_wait_time: 9, retry_interval: 3, max_poll_count: 36 }, preproc: TimeoutTriple { initial_wait_time: 18000, retry_interval: 15000, max_poll_count: 1150 }, decryption: TimeoutTriple { initial_wait_time: 1, retry_interval: 1, max_poll_count: 24 }, reencryption: TimeoutTriple { initial_wait_time: 1, retry_interval: 1, max_poll_count: 24 }, verify_proven_ct: TimeoutTriple { initial_wait_time: 1, retry_interval: 1, max_poll_count: 24 } } }, oracle: OracleConfig { addresses: ["http://localhost:26657"] }, store: StoreConfig { url: "http://dev-kv-store:8088" } }
2024-11-15T10:21:08.170061Z  INFO kms_blockchain_connector::application::kms_core_sync: Received message: TransactionEvent { tx_hash: "50EC66BCF0206AC893BC0577C055FA0078BD3E7D6345E9AE201AA4C8B0F9029C", event: KmsEvent { operation: InsecureKeyGen, txn_id: TransactionId(HexVector([83, 16, 141, 162, 253, 111, 232, 148, 127, 77, 240, 234, 205, 8, 63, 167, 89, 97, 117, 103])) } }
2024-11-15T10:21:08.170224Z  INFO get_operation_value{event=KmsEvent { operation: InsecureKeyGen, txn_id: TransactionId(HexVector([83, 16, 141, 162, 253, 111, 232, 148, 127, 77, 240, 234, 205, 8, 63, 167, 89, 97, 117, 103])) }}:query_contract: kms_blockchain_client::query_client: contract address: wasm1cnuw3f076wgdyahssdkd0g3nr96ckq8cwa2mh029fn5mgf2fmcms9ax00l
2024-11-15T10:21:08.174239Z  INFO get_operation_value{event=KmsEvent { operation: InsecureKeyGen, txn_id: TransactionId(HexVector([83, 16, 141, 162, 253, 111, 232, 148, 127, 77, 240, 234, 205, 8, 63, 167, 89, 97, 117, 103])) }}:query_contract: kms_blockchain_client::query_client: Query executed successfully 316
2024-11-15T10:21:08.174312Z  INFO kms_blockchain_connector::application::kms_core_sync: Running KMS operation with value: InsecureKeyGen(InsecureKeyGenValues { eip712_name: "eip712_name", eip712_version: "1.0.4", eip712_chain_id: HexVector([42, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]), eip712_verifying_contract: "0x00dA6BF26964af9D7EED9e03E53415d37aa960EE", eip712_salt: Some(HexVector([0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31])) })
2024-11-15T10:21:20.184034Z  INFO kms_blockchain_connector::application::kms_core_sync: Sending response to the blockchain: KeyGenResponse
2024-11-15T10:21:20.184105Z  INFO send_result{tx_id=53108da2fd6fe8947f4df0eacd083fa759617567}: kms_blockchain_connector::infrastructure::blockchain: Sending result to contract: ExecuteContractRequest { message: KmsMessage { txn_id: Some(TransactionId(HexVector([83, 16, 141, 162, 253, 111, 232, 148, 127, 77, 240, 234, 205, 8, 63, 167, 89, 97, 117, 103]))), value: KeyGenResponse(KeyGenResponseValues { request_id: HexVector([83, 16, 141, 162, 253, 111, 232, 148, 127, 77, 240, 234, 205, 8, 63, 167, 89, 97, 117, 103]), public_key_digest: "4082a86a0e0d3db82a7f767d5eed54f20c9d6cfa", public_key_signature: HexVector([64, 0, ... 86]), server_key_digest: "7c210349453c4d29b9d852dbdf6e622973e03804", server_key_signature: HexVector([64, 0, 0, 0, 0, 0,...21]), param: Default }) }, gas_limit: 3000000, funds: None }
```

</details>

### 2. Retrieve public key material

```bash
make init-db
```

<details><summary> 💡 Details </summary>

This command performs the following actions:

- Retrieves from [http://localhost:9001/browser/kms](http://localhost:9001/browser/kms):
  - FHE Public Key
  - FHE Server Key (Evaluation Key)
  - CRS for input proofs
  - MPC nodes signers (used to sign decryption results, for example)
- Inserts Keys into the coprocessor database.
- Updates Signers in the ./env/.env.example.deployment file.

</details>

### 3. Deploy fhevm smart contracts

```bash
make prepare-e2e-test
```

<details><summary> 💡 Details </summary>

This command will deploy on the host blockchain

- GatewayContract.sol: used to trigger decryption events catched by the `gateway`
- ACL.sol: used to manage rights to decrypt or reencrypt
- TFHEExecutor.sol: used to trigger FHE operation on coprocessor

```bash
atewayContractAddress written to gateway/.env.gateway successfully!
gateway/lib/GatewayContractAddress.sol file has been generated successfully.
ACL address 0x339EcE85B9E11a3A3AA557582784a15d7F82AAf2 written successfully!
./lib/ACLAddress.sol file generated successfully!
TFHEExecutor address 0x596E6682c72946AF006B27C131793F2b62527A4b written successfully!
./lib/TFHEExecutorAddress.sol file generated successfully!
KMSVerifier address 0x208De73316E44722e16f6dDFF40881A3e4F86104 written successfully!
./lib/KMSVerifierAddress.sol file generated successfully!
InputVerifier address 0x69dE3158643e738a0724418b21a35FAA20CBb1c5 written successfully!
./lib/InputVerifierAddress.sol file generated successfully!
Coprocessor address 0xc9990FEfE0c27D31D0C2aa36196b085c0c4d456c written successfully!
./lib/CoprocessorAddress.sol file generated successfully!
FHEPayment address 0x6d5A11aC509C707c00bc3A0a113ACcC26c532547 written successfully!
./lib/FHEPaymentAddress.sol file generated successfully!
Generating typings for: 30 artifacts in dir: types for target: ethers-v6
Successfully generated 92 typings!
Compiled 32 Solidity files successfully (evm target: cancun).
Generating typings for: 5 artifacts in dir: types for target: ethers-v6
Successfully generated 58 typings!
Compiled 5 Solidity files successfully (evm target: cancun).
ACL was deployed at address: 0x339EcE85B9E11a3A3AA557582784a15d7F82AAf2
TFHEExecutor was deployed at address: 0x596E6682c72946AF006B27C131793F2b62527A4b
KMSVerifier was deployed at address: 0x208De73316E44722e16f6dDFF40881A3e4F86104
InputVerifier was deployed at address: 0x69dE3158643e738a0724418b21a35FAA20CBb1c5
FHEPayment was deployed at address: 0x6d5A11aC509C707c00bc3A0a113ACcC26c532547
KMS signer no0 (0xA488340e90CaF82c8DD1bF4060A779595b8f13Cc) was added to KMSVerifier contract
GatewayContract was deployed at address:  0x096b4679d45fB675d4e2c1E4565009Cec99A12B1
Account 0x97F272ccfef4026A1F3f0e0E879d514627B84E69 was succesfully added as an gateway relayer

```

</details>

### 4. Run a simple test

```bash
make run-async-test
```

To run more test, check this [section](#testing-part).

## Running gateway from source

First run as usual with the dockerized version, stop the gateway and switch to running from source.

In a separate terminal, checkout the branch set in the .env (for instance `v0.9.0-rc38`) in kms-core repository.

### Centralized KMS configuration

Ensure to have in `<PATH_TO_KMS_CORE>/blockchain/gateway/config/gateway.toml`

```toml
# configure to match the KMS mode: centralized or threshold
mode = "centralized"

# List of prefixed of URL endpoints for the public storage of the KMS.
# The key in the map is the MPC ID of the party. I.e. [1; amount of parties]
[kms.public_storage]
1 = "http://localhost:9000/kms/PUB"
```

### Threshold KMS configuration

```toml
# configure to match the KMS mode: centralized or threshold
mode = "threshold"

# List of prefixed of URL endpoints for the public storage of the KMS.
# The key in the map is the MPC ID of the party. I.e. [1; amount of parties]
[kms.public_storage]
1 = "http://localhost:9000/kms/PUB-p1"
2 = "http://localhost:9000/kms/PUB-p2"
3 = "http://localhost:9000/kms/PUB-p3"
4 = "http://localhost:9000/kms/PUB-p4"
```

### Start the gateway

```bash
cd $path-to-kms-core
cd blockchain/gateway
cargo run --bin gateway
```

Wait for the gateway to start listening for blocks and print block numbers.

```bash
...
2024-10-16T15:35:22.876765Z  INFO gateway::events::manager: 🧱 block number: 10
2024-10-16T15:35:27.787809Z  INFO gateway::events::manager: 🧱 block number: 11
...
```

<details><summary> 💡 Check the keys are ready by calling `\keyurl` endpoint </summary>

```bash
curl  http://localhost:7077/keyurl
```

</details>

## Testing part

Note: If one of the non-trivial or re-encrypt tests fails, it should succeed
after a retry.

> [!Warning]
> One must set localCoprocessor in `./work_dir/fhevm/hardhat.config.ts` to avoid adding --network localCoprocessor for each following tests

Tests should be run from `work_dir/fhevm`

```bash
    cd work_dir/fhevm && npx hardhat test --grep 'test aasync decrypt uint32$'
```

1. PASSING TESTS - All tests for trivial decrypt should now pass.

   ```bash
   npx hardhat test --grep 'test async decrypt bool$'
   npx hardhat test --grep 'test async decrypt uint4$'
   npx hardhat test --grep 'test async decrypt uint8$'
   npx hardhat test --grep 'test async decrypt uint16$'
   npx hardhat test --grep 'test async decrypt uint32$'
   npx hardhat test --grep 'test async decrypt uint64$'
   npx hardhat test --grep 'test async decrypt uint128$'
   npx hardhat test --grep 'test async decrypt uint256$'
   npx hardhat test --grep 'test async decrypt address$'
   npx hardhat test --grep 'test async decrypt mixed$'
   ```

2. PASSING TESTS - All tests for trivial reencrypt should now pass.

   ```bash
   npx hardhat test --grep 'test reencrypt bool$'
   npx hardhat test --grep 'test reencrypt uint4$'
   npx hardhat test --grep 'test reencrypt uint8$'
   npx hardhat test --grep 'test reencrypt uint16$'
   npx hardhat test --grep 'test reencrypt uint32$'
   npx hardhat test --grep 'test reencrypt uint64$'
   npx hardhat test --grep 'test reencrypt uint128$'
   npx hardhat test --grep 'test reencrypt uint256$'
   npx hardhat test --grep 'test reencrypt address$'
   ```

3. PASSING TEST - Non trivial decrypt with input mechanism

   ```bash
   npx hardhat test --grep 'test async decrypt uint64 non-trivial'
   ```

4. FAILING TEST - on centralized

- decryption ebytes
- reencryption ebytes

5. FAILING TEST - on threshold

- decryption ebytes, euint256
- reencryption ebytes, euint256

## Some issues you can encounter

<details>
<summary> 💡 Unable to delete a docker network, example at `make clean` step  </summary>

```bash
failed to remove network zama_default: Error response from daemon: error while removing network: network zama_default id 7f9cb8a4b1107a6c53663c4f5e513f6008bc227122ff64693576c8a686aaeae8 has active endpoints
```

First try to docker prune the networks

```bash
docker network prune

# Check networks
docker network ls | grep zama
4f43b8c0143b   zama_default   bridge    local
```

If the network is still here, just restart the docker deamon:

```bash
# Linux-based OS
sudo service docker restart
docker network prune
```

</details>
