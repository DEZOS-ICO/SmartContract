# Dezos Smart Contract

The Dezos smart contract is located [here](https://github.com/DEZOS-ICO/SmartContract/blob/master/src/test/resources/DOS.sol).
The contract implements ERC 20 and ERC 865. It also implements transferAndCall, 
which is similar to ERC 677. The main difference to ERC 677 is that from this contract any other 
contract can be called and does not need to implement the *tokenFallback()* function.

The use-case for a user to register an item will be used to showcase the token flow within 
the contract. One important feature is [ERC 865](https://github.com/ethereum/EIPs/issues/865), 
which allows to call functions without Ethers. The gas can be e.g., payed in tokens. As this 
feature is required now, waiting for [EIP 101](https://github.com/ethereum/EIPs/issues/28) 
is not an option. More information about account abstraction can be found 
[here](https://blog.ethereum.org/2015/12/24/understanding-serenity-part-i-abstraction/).

## Contract Details
tbd.

total supply: 9mio
3 admins -> disable token
6month cliff set during minting
name "DOS Token";
symbol = "DOS";
decimals = 18;

Example Token Contract: Inventory.sol

## Technical Details

With ERC 865, we have 2 additional functions *transferPreSigned* and *transferAndCallPreSigned*, 
which will be called by the DOS service. In both functions, the user needs to pre sign the 
transaction and agree to a fee that has to be paid in DOS tokens. The DOS service receives 
the signed transaction and calls *transferPreSigned* or *transferAndCallPreSigned*. This 
function checks if the signature from the user is valid, deducts the tokens from the user 
account, and adds to to the recipient address. The DOS service pays the fees in Ethers, 
but gets the equivalent value in DOS tokens. Thus, the user does not need to have Ether, 
only DOS tokens to use the system.

As the signature is the transaction signature from the user, the function calls 
*transferPreSigned* and *transferAndCallPreSigned* only succeed if the signature matches. 
The DOS service layer needs to communicated the fee amount beforehand. That means that the 
user that registers its item only needs the DOS token. So, a private key to access his 
tokens is enough. This allows for a much better UX as tokens can be given to users as a 
promotion and tokens can be bought from the DOS website. No Ethers necessary.

In order to not implement all the future functionality into the token now, ERC 677 is 
used where tokens can be used to call other methods: 
*transferAndCall(address _to, uint256 _value, bytes4 _methodName, bytes _args)*. The address *_to* 
needs to be aware of the contract. However, it does not need to have the following interface: 
*function tokenFallback(address _from, uint _value, bytes _data)*. It can be any function with 
the first two parameters are *address* and *uint256* with the from address respectively the 
value that was transferred. The function on the other contract that is being called should 
be aware of the DOS tokens, as otherwise the tokens could be locked up forever. The 
disadvantage of not having a tokenFallback is that tokens could be locked up if a wrong 
contract is called. However, the advantages are easier parameter encoding/decoding and 
calling any contracts function. Since this functionality will be used through the DOS service, 
allowing only whitelisted functions on contracts, locking up tokens should not happen.

With such a flexibility, the encoding of the function needs special attention. An example 
of such an encoding is here, which calls a function in contract *Inventory.sol*:

```
String invParam = io.iconator.testonator.Utils.encodeParameters(2,
                new Utf8String("serial"),
                new Utf8String("description"));
                
byte[] b4 = Numeric.hexStringToByteArray("0x5c28b451");
byte[] args = Numeric.hexStringToByteArray(invParam);
```

These parameters that will be used for the function *itemize(address,uint256,string,string)*, with
the signature 5c28b451. The encodeParameters starts at 2, as the first two parameters are set by
the contract: *address*, which is the address of the one who signed (CREDENTIAL_1 in this example), 
and *uint256*, which is the value of tokens that was transferred.

Then the parameters for the DOS contract need to be encoded, including the arguments for the 
*itemize* function call. The following snippet follows this function: 

```
transferAndCallPreSigned(bytes memory _signature, address _to, uint256 _value, 
    uint256 _fee, uint256 _nonce, bytes4 _methodName, bytes memory _args)
```

The encodeParameters starts at 0, as no additional parameters are prepended.
```
String encoded = io.iconator.testonator.Utils.encodeParameters(0,
                new Bytes4(Numeric.hexStringToByteArray("0x38980f82")),
                new Address(deployed.contractAddress()),
                new Address(deployedInventory.contractAddress()),
                new Uint256(new BigInteger("1")),
                new Uint256(new BigInteger("1")),
                new Uint256(new BigInteger("2")),
                new Bytes4(b4),
                new DynamicBytes(args));

```
Once the encoded string is created according to Soliditys *abi.encode()*, it can be hashed and 
signed:
```
String keccak = Hash.sha3(encoded);

Sign.SignatureData sigData = Sign.signMessage(
                Numeric.hexStringToByteArray(keccak),
                CREDENTIAL_1.getEcKeyPair(),
                false);
```
Here, CREDENTIAL_1 will sign the data, and the signature is assembled as in Solidity:
```
byte[] sig = new byte[65];
System.arraycopy(sigData.getR(),0,sig,0,32);
System.arraycopy(sigData.getS(),0,sig,32,32);
sig[64] = sigData.getV();
```
The signature can then be sent to the DOS service. The DOS service checks if everything matches
together with the input data, and if yes, it will send the parameters with the signature to the
Ethereum network:
```
List<Event> event = blockchain.call(CREDENTIAL_3, deployed,
                "transferAndCallPreSigned",
                sig,
                deployedInventory.contractAddress(),
                new BigInteger("1"),
                new BigInteger("1"),
                new BigInteger("2"),
                b4,
                args);
```
This is a regular Ethereum transaction, coming from the DOS service (in this example CREDENTIAL_3),
however, the transaction comes originally from the user (in this example CREDENTIAL_1), who
spent DOS tokens to pay for the Ethereum transaction fee.

Unfortunately, the recover() method from OpenZeppelin allows signature malleability, thus, the
recover() function was adapted accordingly. The discussion is 
[ongoing](https://github.com/OpenZeppelin/openzeppelin-solidity/pull/1622).

## Installation Details

Testing is done with JUnit and [Testonator](https://github.com/ICOnator/Testonator). 
The repository can be checked out with: 

```
git clone https://github.com/DEZOS-ICO/SmartContract.git
```

and run the testcases. It uses Solidity 0.5.x.

```
cd SmartContract
./gradlew clean build
```
