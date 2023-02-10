# ZK Email Verify

**WIP: This tech is extremely tricky to use and very much a work in progress, and we do not recommend use in any production application right now. This is both due to unaudited code, and several theoretical issues such as nullifiers, bcc’s, non-nested signatures, and hash sizings. We are working on addressing those, and if you have a possible usecase, please run it by us so we can ensure that your guarantees are in fact correct!**

Join the conversation [on discord](https://discord.gg/34EPvjuPZj) or via [dm](https://twitter.com/yush_g/)!

## MVP App

The application is located at https://zkemail.xyz. It only works on Chrome/Brave/Arc (or other Chromium-based browsers) due to download limits on other browsers.

The documentation for the app is located at https://zkemail.xyz/docs (WIP). Made by [@yush_g](https://twitter.com/yush_g) and [@sampriti0](https://twitter.com/sampriti0) at [@0xparc](https://twitter.com/0xparc) and [@personae_labs](https://twitter.com/personae_labs), dm if interested in usage or building next generation primitives like this. This is very much a work in progress, and we invite folks to contribute, or contact us for interesting projects that can be built on top of the tech! We are especially prioritizing optimizing circuits, making our end-to-end demo more efficient and on-chain, and an SDK/CLI.

## Local website

To run the frontend with existing circuits (there is no backend or server), enable Node 16 (with nvm) and run:

```
yarn start
```

If the frontend shows an error on fullProve line, run this and rerun

```
yarn add snarkjs@https://github.com/sampritipanda/snarkjs.git#fef81fc51d17a734637555c6edbd585ecda02d9e
```

## Getting email headers

In Outlook, turn on plain text mode. Copy paste the 'full email details' into the textbox on the (only client side!) webpage.

In gmail, download original message then copy paste the contents into the textbox.

# Development Instructions

This will let you build new zkeys from source.

## Filetree Description

```bash
circuits/ # groth16 zk circuits
    contracts/ # Auto-gen verifier
    example/ # Example proofs, publics, and private witnesses
    inputs/ # Test inputs for example witness generation for compilation
        input_email_domain.json # Standard input for from/to mit.edu domain matching, for use with circuit without body checks
        input_email_packed.json # Same as above but has useless packed input -- is private so irrelevant, this file could be deleted.
    main/ # Legacy RSA code
    scripts/ # Run snarkjs ceremony to generate zkey with yarn compile
dizkus-scripts/
    *.sh # Scripts to compile the chunked keys on a remote server
    *.circom # Final circom file that imports from the circuits
    sample_input.json # Generated by running generate_input.ts on an email file, or by asking Aayush for one
docs/
src/
    circuits/  # Has vkey
    contracts/ # Run foundry commands from this folder
        src/   # Note that these are untested WIPs. Need to decrease calldata to be able to work on chain.
            emailHandlerBase.sol # Build new verifiers by forking this
            twitterEmailHandler.sol # Verifies Twitter usernames and issues a badge
            domainEmailHandler.sol # Verifies email domain and issues a badge
        lib/ # Foundry libraries
    helpers/ # Shared JS/TS helpers for both input generation + frontend
    pages/   # Frontend
    scripts/
        fast-sha256.ts # SHA 256 helper that we use for partial SHA
        generate_input.ts # Helper to convert email into circuit input
public/ # Should contain vkey/wasm, but we end up fetching those from AWS server instead
    docs/
    logos
    vkey
    wasm
```

## Regex to Circom

Modify the `let regex = ` in lexical.js and then run `python3 gen.py`

## Email Circuit Build Steps

Install rust/circom2 via the following steps, according to: https://docs.circom.io/getting-started/installation/

```bash
curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh # Install rust if don't already have
source "$HOME/.cargo/env" # Also rust installation step

git clone https://github.com/iden3/circom.git
cd circom
cargo build --release
cargo install --path circom
sudo apt-get install nlohmann-json3-dev libgmp-dev nasm # Ubuntu packages needed for C-based witness generator
brew install nlohmann-json gmp nasm # OSX
```

Inside `zk-email-verify` folder, do

```
sudo npm i -g yarn # If don't have yarn
yarn install # If this fails, delete yarn.lock and try again
```

To get the ptau, do (note that you only need the 22 file right now)

```bash
wget https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_22.ptau
mv powersOfTau28_hez_final_22.ptau powersoftau/powersOfTau28_hez_final_22.ptau

wget https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_21.ptau
# shasum pot21_final.ptau: e0ef07ede5c01b1f7ddabb14b60c0b740b357f70
mv powersOfTau28_hez_final_21.ptau powersoftau/powersOfTau28_hez_final_21.ptau
```

<!-- Previously snarkjs@git+https://github.com/vb7401/snarkjs.git#fae4fe381bdad2da13eee71010dfe477fc694ac1 -->
<!-- Now -> yarn add https://github.com/vb7401/snarkjs/commits/chunk_zkey_gen -->

Put the email into ...\*.eml. Edit the constant filename at the top of generate_input.ts to import that file, then use the output of running that file as the input file (you may need to rename it). You'll need this for both zkey and verifier generation.

To create a chunked zkey for in-browser proving, run the following (likely on a high CPU computer):

```bash
yarn add snarkjs@git+https://github.com/vb7401/snarkjs.git#24981febe8826b6ab76ae4d76cf7f9142919d2b8 # Swap to chunked generation version
cd dizkus-scripts/
./1_compile.sh && ./2_gen_wtns.sh && ./3_gen_chunk_zkey.sh && ./4_gen_vkey.sh && ./5_gen_proof.sh
# optional: ./6_gen_proof_rapidsnark.sh
aws configure # Only needs to be run once
pip3 install boto3
python3 upload_to_s3.py
yarn add snarkjs@https://github.com/sampritipanda/snarkjs.git#fef81fc51d17a734637555c6edbd585ecda02d9e # Revert to frontend version
```

Note that there's no .zkeya file, only .zkeyb ... .zkeyk. The script will automatically zip into .tar.gz files and load into s3 bucket.

We use a fork of [zkp.ts](https://github.com/personaelabs/heyanon/blob/main/lib/zkp.ts) to load these keys into localforage. In the browser, to read off of localforage, you have to use this fork when running the frontend locally/in prod:

```
yarn install snarkjs@git+https://github.com/vb7401/snarkjs.git#53e86631b5e409e5bd30300611b495ca469503bc
```

Manually copy paste the modulus in the resulting generated file into solidity verified mailserver keys.

Change s3 address in the frontend to your bucket.

To do a non-chunked zkey for non-browser running,

```
yarn compile-all
```

## Compiling Subcircuits

If you want to compile subcircuits instead of the whole thing, you can use the following:

If you want to generate a new email/set of inputs, edit the src/constants.ts file with your constants.
In generate_input.ts, change the circuitType variable inside to match what circom file you are running, then run

```bash
npm install typescript ts-node -g
# uncomment do_generate function call at end of file
# go to tsconfig.json and change esnext to CommonJS
# if weird things dont work with this and yarn start, go go node_modules/react-scripts/config/webpack.config.ts and add/cut `target: 'node',` after like 793 after `node:`.
npx tsc --moduleResolution node --target esnext circuits/scripts/generate_input.ts
```

which will autowrite input\_<circuitName>.json to the inputs folder.

To do the steps in https://github.com/iden3/snarkjs#7-prepare-phase-2 automatically, do

```
yarn compile email true
```

and you can swap `email` for `sha` or `rsa` or any other circuit name that matches your generate_input type.

and when the circuit doesn't change,

```
yarn compile email true skip-r1cswasm
```

and when the zkey also doesn't change,

```
yarn compile email true skip-r1cswasm skip-zkey
```

## Production

For production, make sure to set a beacon in .env.

Note that this leaks the number of characters in the username of someone who sent you an email, iff the first field in the email serialization format is from (effectively irrelevant).

## Testing

To constraint count, do

```
cd circuits
node --max-old-space-size=614400 ./../node_modules/.bin/snarkjs r1cs info email.r1cs
```

To test solidity,

```
cp node_modules/forge-std src/contracts/lib/forge-std
cd src/contracts
forge test
```

To deploy contract to forked mainnet, do:

```
anvil --fork-url https://eth-mainnet.alchemyapi.io/v2/***REMOVED*** --port 8547 # Run in tmux
export ETH_RPC_URL=http://localhost:8547
forge create --rpc-url $ETH_RPC_URL src/contracts/src/emailVerifier.sol:Verifier --private-key  0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 # Public anvil sk
```

## Constraint count

|          Operation          | Constraint # |
| :-------------------------: | :----------: |
|     SHA of email header     |   506,670    |
|        RSA signature        |   149,251    |
|      DKIM header regex      |   736,553    |
|       Body hash regex       |   760,142    |
|        SHA body hash        |   506,670    |
|    Twitter handle regex     |   328,044    |
| Packing output for solidity |    16,800    |
|      Total constraints      |  3,115,057   |

## General guidelines

Just RSA + SHA (without masking or regex proofs) for arbitrary message length <= 512 bytes is 402,802 constraints, and the zkey took 42 minutes to generate on an intel mac.

RSA + SHA + Regex + Masking with up to 1024 byte message lengths is 1,392,219 constraints, and the chunked zkey took 9 + 15 + 15 + 2 minutes to generate on a machine with 32 cores.

The full email header circuit above with the 7-byte packing into signals is 1,408,571 constraints, with 163 public signals, and the verifier script fits in the 24kb contract limit.

The full email header and body check circuit, with 7-byte packing and final public output compression, is **3,115,057 constraints**, with 21 public signals. zkey size was originally 1.75GB, and with tar.gz compression it is now 982 MB.

In the browser, on a 2019 Intel Mac on Chrome, proving uses 7.3/8 cores. zk-gen takes 384 s, groth16 prove takes 375 s, and witness calculation takes 9 s.

For baremetal, proof generation time on 16 CPUs took 97 seconds. Generating zkey 0 took 17 minutes. Unclear about zkey 1. Zkey 2 took 5 minutes. r1cs + wasm generation took 5 minutes. Witness generation took 16 seconds. cpp witness gen file generation (from script 6) took 210 minutes.

### Scrubbing Sensitive Files

```
brew install git-filter-repo
git filter-repo --replace-text <(echo "0x000000000000000000000000000000000000000000000000000000000abcdef")
git filter-repo --path mit_msg.eml --invert-paths
git remote add origin https://github.com/zk-email-verify/zk-email-verify
ls
git push --set-upstream origin main --force
```

## FAQ/Errors

### I'm having issues with the intricacies of the SHA hashing. How do I understand the function better?

Use https://sha256algorithm.com/ as an explainer! It's a great visualization of what is going on, and our code should match what is going on there.

### What are the differences between generating proofs (snarkjs.groth16.fullprove) on the client vs. on a server?

If the server is generating the proof, it has to have the private input. We want people to own their own data, so client side proving is the most secure both privacy and anonymity wise. There are fancier solutions (MPC, FHE, recursive proofs etc), but those are still in the research stage.

### “Cannot resolve module ‘fs’”

Fixed by downgrading react-scripts version.

### TypeError: Cannot read properties of undefined (reading 'toString')

This is the full error:

```
zk-email-verify/src/scripts/generateinput.ts:182
const = result.results[0].publicKey.toString();
                                    ^
TypeError: Cannot read properties of undefined (reading 'toString')
```

You need to have internet connection while running dkim verification locally, in order to fetch the public key.

### No available storage method found.

If when using snarkjs, you see this:

```
[ERROR] snarkJS: Error: No available storage method found. [full path]
/node_modules/localforage/dist/localforage.js:2762:25
```

Rerun with this:
`yarn add snarkjs@git+https://github.com/vb7401/snarkjs.git#24981febe8826b6ab76ae4d76cf7f9142919d2b8`

### I'm trying to edit the circuits, and running into the error 'Non-quadratic constraints are not allowed!'

The line number of this error is usually arbitrary. Make sure you are not mixing signals and variables anywhere: signals can only be assigned once, and assigned to other signals (not variables), and cannot be used as parameters in control flow like for, if, array indexing, etc. You can get versions of these by using components like isEqual, lessThan, and quinSelector, respectively.

### Where do I get the public key for the signature?

Usually, this will be hosted on some URL under the parent organization. You can try to get it from a .pem file, but that is usually a fraught effort since the encoding of such files varies a lot, is idiosyncratic, and hard to parse. The easiest way is to just extract it from the RSA signature itself, and just verify that it matches the parent organization.

### How can I trust that you verify the correct public key?

You can see the decomposed public key in our Solidity verifier, and you can auto-check this against the mailserver URL. This prevents the code from falling victim to DNS spoofing attacks. We don't have mailserver key rotations figured out right now, but we expect that can be done trustlessly via DNSSEC (though not widely enabled) or via deploying another contract.

### How do I get a Verifier.sol file that matches my chunked zkeys?

You should be able to put in identical randomness on both the chunked zkey fork and the regular zkey generation fork in the beacon and Powers of Tau phase 2, to be able to get the same zkey in both a chunked and non-chunked form. You can then run compile.js, or if you prefer the individual line, just `node --max-old-space-size=614400 ${snarkJSPath} zkey export solidityverifier ${cwd}/circuits/${circuitNamePrimary}/keys/circuit_final.zkey ${cwd}/circuits/contracts/verifier.sol`, where you edit the path variables to be your preferred ones.

The chunked file utils will automatically search for circuit_final.zkeyb from this command line call if you are using the chunked zkey fork (you'll know you have that fork, if you have a file called chunkFileUtils in snarkJS).

### How do I deal with all of these snarkJS forks?

Apologies, this part is some messy legacy code from previous projects. You use vivekab's fork for keygeneration, sampritipanda's fork for chunked zkey checking on the frontend, and the original snarkjs@latest to get rid of chunking entirely (but you'll need to edit frontend code to not do that). You can do something like `./node_modules/bin/snarkjs' inside your repo, and it'll run the snarkjs command built from the fork you're using instead of the global one.

### How do I build my own frontend but plug in your ZK parsing?

zkp.ts is the key file that calls the important proving functions. You should be able to just call the exported functions from there, along with setting up your own s3 bucket and setting the constants at the top.

### Why did you choose GPL over MIT licensing?

Since circom is GPL, we are forced to use the GPL license, which is still a highly permissive license. You can dm us if you'd like to treat non-circom parts of the repo as MIT licensed, but broadly we are pro permissive open source usage with attribution! We hope that those who derive profit from this primitive contribute that money altruistically back to this technology.

## To-Do

- Make the frontend Solidity calls work
- Make a general method to get formatted signatures and bodies from all email clients
- Make versions for different size RSA keys
- Add ENS DNSSEC code (possibly SNARKed), so anyone can add a website's RSA key via DNS record
- Design the NFT/POAP to have the user's domain/verified identity on it
- Make a testnet faucet as a PoC for Sybil resistance and to get developers interested
- Dynamically tradeoff between gzip (2x faster decompression) and xz (30% smaller file size): https://www.rootusers.com/gzip-vs-bzip2-vs-xz-performance-comparison/ based on internet speed (i.e. minimize download time + unzip time)
