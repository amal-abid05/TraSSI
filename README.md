# TraSSI

TraSSI


## Installation

To get started, perform installation.

### Install Zokrates

Link: https://zokrates.github.io/gettingstarted.html

##### One-line installation
``` bash
$ curl -LSfs get.zokrat.es | sh
```
##### From source
``` bash
$ git clone https://github.com/ZoKrates/ZoKrates
$ cd ZoKrates
$ export ZOKRATES_STDLIB=$PWD/zokrates_stdlib/stdlib
$ cargo +nightly build -p zokrates_cli --release
$ cd target/release
```
##### Docker
``` bash
$ docker run -ti zokrates/zokrates /bin/bash
```

### Install Zokrates Python Library: zokrates_pycrypto

``` bash
$ pip install zokrates_pycrypto
```

##### N.B.: Add "Zokrates" and "zokrates_pycrypto" to the PATH.


### Install TenSEAL
``` bash
$ pip install tenseal
```

### Install and Setup Truffle 
``` bash
$ npm install -g truffle
$ truffle init
$ truffle compile
```




## ZKP: Non-Repudiable Identity-Associated Proofs of Knowledge
First, create the file identity_zkp.zok and add the following code:

``` bash
from "ecc/babyjubjubParams" import BabyJubJubParams;
import "ecc/babyjubjubParams" as context;
import "ecc/proofOfOwnership" as proofOfOwnership;
import "hashes/sha256/512bitPacked" as sha256packed;

def proofOfKnowledge(field[4] secret, field[2] hash) -> bool {
  return (hash == sha256packed(secret));
  }
  
def main(field[2] pkA, field[2] pkB, field[2] hash, private field skA, private field[4] secret, private field skB) -> bool {
  BabyJubJubParams context = context();
  bool orgAhasKnowledge = proofOfKnowledge(secret, hash);
  bool orgAhasOwnership = proofOfOwnership(pkA, skA, context);
  bool audhasOwnership = proofOfOwnership(pkB, skB, context);
  bool isOrgAwithKnowledge = (orgAhasKnowledge && orgAhasOwnership);
  bool out = (isOrgAwithKnowledge == true || audhasOwnership == true);
  return out;
  }
```

Then run the different phases of Zokrates:
``` bash
# compile
$ zokrates compile -i identity_zkp.zok

Compiling identity.zok
Compiled code written to 'out'
Number of constraints: 55510


# perform the setup phase
$zokrates setup

Performing setup...
Verification key written to 'verification.key'
Proving key written to 'proving.key'
Setup completed


# execute the program
$ zokrates compute-witness -a 14897476871502190904409029696666322856887678969656209656241038339251270171395 16668832459046858928951622951481252834155254151733002984053501254009901876174 14897476871502190904409029696666322856887678969656209656241038339251270171395 16668832459046858928951622951481252834155254151733002984053501254009901876174 66887177014510045729599711290250768514642621566450030829962492932967577128470 2672298032000151429627794792737979639400309965406749841034588150477310325429 1997011358982923168928344992199991480689546837621580239342656433234255379025 1 2 3 4 1997011358982923168928344992199991480689546837621580239342656433234255379025

Computing witness...
Witness file written to 'witness'


# generate a proof of computation
zokrates generate-proof

Generating proof...
Proof written to 'proof.json'


# export a solidity verifier
zokrates export-verifier

Exporting verifier...
Verifier exported to 'verifier.sol'


# or verify natively
zokrates verify
Performing verification...
PASSED
```

## FHE: Computations over encrypted data

## step1.py

Add the code: 

``` bash
import tenseal as ts
import hashlib
from zokrates_pycrypto.eddsa import PrivateKey, PublicKey
from zokrates_pycrypto.field import FQ
from zokrates_pycrypto.utils import write_signature_for_zokrates_cli

if __name__ == "__main__":

    # Setup TenSEAL context
    context = ts.context(
            ts.SCHEME_TYPE.CKKS,
            poly_modulus_degree=8192,
            coeff_mod_bit_sizes=[60, 40, 40, 60]
          )
    context.generate_galois_keys()
    context.global_scale = 2**40

    # Encrypt the number of units sold
    num_units = [100]
    encrypted_num_units = ts.ckks_vector(context, num_units)

    # Encrypt the price per unit
    price_per_unit = [50]
    encrypted_price_per_unit = ts.ckks_vector(context,price_per_unit)

    # Compute the total revenue
    encrypted_revenue = encrypted_num_units * encrypted_price_per_unit

    # Setup maximum threshold
    max_threshold = "7000"

    # Write witness arguments to disk / Serialize the decrypted result and max_threshold for ZoKrates input
    path = "zokrates_inputs.txt"
    #witness = "" + encrypted_revenue.decrypt()[0] + " " + max_threshold
    witness = str(round(encrypted_revenue.decrypt()[0])) + " " + max_threshold
    with open(path, "w+") as f:
        f.write("".join(witness))

    # Compile the ZoKrates program and generate the witness
    # ...
```

Run the code:

``` bash
$ python step1.py
```


## step2.zok

Add the code: 

``` bash
def main(private field revenue, field max_threshold) -> bool {
//Check if the revenue exceeds the max threshold
	return (revenue <= max_threshold);
	}
```

Run the code:

``` bash
# compile
$ zokrates compile -i step2.zok
# perform the setup phase
$ zokrates setup
# execute the program
$ zokrates compute-witness -a $(cat zokrates_inputs.txt)
# generate a proof of computation
$ zokrates generate-proof
# export a solidity verifier
$ zokrates export-verifier
# verify natively
$ zokrates verify
```

## step3.py

Add the code: 

``` bash
from web3 import Web3
import json
import os

# Load compiled smart contract ABI
with open('build/contracts/Verifier.json', 'r') as f:
    contract_abi = json.load(f)['abi']

# Connect to Ethereum node with Web3 provider
#w3 = Web3(Web3.HTTPProvider('http://localhost:8545'))
#w3 = Web3(Web3.HTTPProvider('https://goerli.infura.io/v3/YOUR-PROJECT-ID'))
w3 = Web3(Web3.HTTPProvider('https://goerli.infura.io/v3/1fd0e66b35304942a881667481243dbb'))


# Load deployed smart contract instance
contract_address = '0x461fB743377A8D557122171D1181F431103151D5'  # Replace with actual deployed contract address
contract = w3.eth.contract(address=contract_address, abi=contract_abi)


deployed_address = contract_address
assert contract.address.lower() == deployed_address.lower()

expected_network_id = 5
assert w3.eth.chain_id == expected_network_id



# Load proof values from proof.json file generated by Zokrates
with open('proof.json', 'r') as f:
    proof_data = json.load(f)
    

proof = proof_data.
input_values =  proof_data.


# Call smart contract's verifyTx function with proof and input values
result = contract.functions.verifyTx(proof, input_values).call()

# Print the result value
print(result)
```

Run the code:

``` bash
$ python step3.py
```

##### We can replace simple arithmetic operations on encrypted data (+,-,*, >,<) by Machine Learning algorithms (e.g.Logistic Regression).
##### An example of Logistic Regression using TenSEAL is available here:
https://github.com/OpenMined/TenSEAL/blob/main/tutorials/Tutorial%201%20-%20Training%20and%20Evaluation%20of%20Logistic%20Regression%20on%20Encrypted%20Data.ipynb
##### This code can be added in our "step1.py" instead of simple multiplication.
