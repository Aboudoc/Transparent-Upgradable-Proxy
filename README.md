<a name="readme-top"></a>

[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![MIT License][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly.git">
    <img src="images/logo.png" alt="Logo" width="100" height="80">
  </a>

<h3 align="center">Transparent Upgradable Proxy</h3>

  <p align="center">
    A Transparent Upgradable Proxy using assembly to return from fallback and to write to any storage slot
    <br />
    <a href="https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly"><strong>Explore the docs »</strong></a>
    <br />
    <br />
    <a href="https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly">View Demo</a>
    ·
    <a href="https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly/issues">Report Bug</a>
    ·
    <a href="https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly/issues">Request Feature</a>
  </p>
</div>

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#prerequisites">Prerequisites</a></li>
        <li><a href="#installation">Installation</a></li>
      </ul>
    </li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#roadmap">Roadmap</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>

<!-- ABOUT THE PROJECT -->

## About The Project

<p align="right">(<a href="#readme-top">back to top</a>)</p>

### Built With

-   [![Hardhat][Hardhat]][Hardhat-url]
-   [![Ethers][Ethers.js]][Ethers-url]

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- GETTING STARTED -->

## Getting Started

To get a local copy up and running follow these simple example steps.

## Getting Started

To get a local copy up and running follow these simple example steps.

### Prerequisites

-   npm

    ```sh
    npm install npm@latest -g
    ```

-   hardhat

    ```sh
    npm install --save-dev hardhat
    ```

    ```sh
    npm install @nomiclabs/hardhat-ethers @nomiclabs/hardhat-waffle
    ```

    run:

    ```sh
    npx hardhat
    ```

### Installation

1. Clone the repo
    ```sh
    git clone https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly.git
    ```
2. Install NPM packages
    ```sh
    npm install
    ```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- USAGE EXAMPLES -->

## Transparent Uppgradable Proxy

1. Intro: the wrong way to implement proxy
2. Return data from fallback
3. Storage for implementation and admin
4. Separate user / admin interfaces
5. Proxy admin contract

## Intro: the wrong way to implement proxy

Let's consider the following upgradable contract with two separate implementations: CounterV1 and CounterV2:

```js

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract BuggyProxy {
    address public  implementation;
    address public admin;

    constructor() {
        admin = msg.sender;
    }

    function _delegate() private  {
        (bool ok, bytes memory res) = implementation.delegatecall(msg.data);
        require(ok);
    }

    receive() external payable{
        _delegate();
    }

    fallback() external payable {
        _delegate();
    }

    function upgradeTo(address _implementation) external {
        require(msg.sender == admin);
        implementation = _implementation;
    }
}

contract CounterV1 {
    address public implementation;
    address public admin;
    uint public count;

    function inc() external {
        count += 1;
    }
}

contract CounterV2 {
    address public implementation;
    address public admin;
    uint public count;

    function inc() external {
        count += 1;
    }

    function dec() external {
        count -= 1;
    }
}

```

**Two problems to fix:**

1.  All of the `implementation` contract must have the same storage layout as the `proxy` contract
2.  The `fallback`can not return any data, so we can not get the count of the counter.

## Return data from `fallback`

We can increment the count using `inc` function, but if you try to get the count, it will return 0

`fallback` function and `receive`function both calls an internal function called `_delegate`

We'll be modifying this function to call `delegatecall` but after it will return the data even though the function signature does say that it return any data.

This will allow us to get the count from the implementation

**To return data, we'll need to use `assembly`**

```js
  /**
   * @dev Delegates execution to an implementation contract.
   * This is a low level function that doesn't return to its internal call site.
   * It will return to the external caller whatever the implementation returns.
   * @param implementation Address to delegate.
   */
  function _delegate(address implementation) internal {
    assembly {
      // Copy msg.data. We take full control of memory in this inline assembly
      // block because it will not return to Solidity code. We overwrite the
      // Solidity scratch pad at memory position 0.
      calldatacopy(0, 0, calldatasize())

      // Call the implementation.
      // out and outsize are 0 because we don't know the size yet.
      let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)

      // Copy the returned data.
      returndatacopy(0, 0, returndatasize())

      switch result
      // delegatecall returns 0 on error.
      case 0 { revert(0, returndatasize()) }
      default { return(0, returndatasize()) }
    }
  }
```

This code comes from the following [repo from Open Zeppelin](https://github.com/OpenZeppelin/upgrades-safe-app/blob/master/contracts/Proxy.sol)

Notes that if you compile you'll get the following error : **only local variables are supported**

so we pass in the storage variable `_implementation` as argument:

```js

function _delegate(address _implementation) internal {}

```

Then, we change the state variable to a local:

```js

let result := delegatecall(gas(), _implementation, 0, calldatasize(), 0, 0)

```

Finally, pass in the state variable to `fallback` and `receive`:

```js
    fallback() external payable {
        _delegate(_implementation);
    }

    receive() external payable {
        _delegate(_implementation);
    }
```

The basic idea for the code inside `assembly` is that we're gonna be copying the `data`, then manually `delegatecall`, and once we call `delegatecall`, we have our data stored in returned data. We'll copy this returned data into memory, then manually return it.

Let's see in details what the code inside `assembly` does

```js
calldatacopy(t, f, s)
```

`calldatacopy` copy the calldata at memory 0 (`t`) starting from the calldata from 0 (`f`) to `calldatasize`

```js
calldatacopy(0, 0, calldatasize())
```

Basically we're copying all of the calldata ounto memory at 0th position

```js

delegatecall(g, a, in, insize, out, outsize)

```

`g` means we are forwarding all of the gas
`a` is the address of the implementation to execut `delegatecall` on
`in` and `insize` means the data is stored inside the memory from 0 to datasize.
`out`and `outside` says to store the result of `delegatecall` to memory from out, to output size. But since we don't know the size of the output before delegating, we are simply ignoring the output, saying 0 and 0.

```js

let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)

```

We were ignoring the output, next we will handle it by copying the output (data returned)

```js
returndatacopy(t, f, s)
```

`returndatacopy` copies `s` bytes from returndata at position `f` to memory at position `t`

```js
returndatacopy(0, 0, returndatasize())
```

Basically we're copying the data that was returned to memory 0 (`t`) starting at the 0th position (`f`) of the memory, and the size of the data to copy is stored in the `returndatasize` (returndatasize() returns the size of the last returndata)

The last step is to handle wether the `delegatecall` was successfull or not

```js

switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }

```

If the `result` is 0, this mean there was an error, so we will revert and return the all of the output that was returned from `delegatecall`, from 0 to `returndatasize`

If the result is not equal to 0, we return the data stored in memory (that was copied using `returndatacopy`) from 0 to `returndatasize`

## Storage for implementation and admin

Our goal is to store the address of the implementation and admin somewhere else, besides the 0th and the 1st slot

First of all, remove the address of the implementation and the admin from `CounterV1` and `CounterV2``

```js
contract CounterV1 {
    uint public count;

    function inc() external {
        count += 1;
    }
}
contract CounterV2 {
    uint public count;

    function inc() external {
        count += 1;
    }

    function dec() external {
        count -= 1;
    }
}
```

Now we need to write to any storage slot. We'll do it by creating a library `StorageSlot`:

```js

library StorageSlot {
    struct AddressSlot {
        address value;
    }

    function getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r.slot := slot
        }
    }
}

```

Keep in mind that the storage of a solidity smart contract is an array of length `2^256`, and to each slot we can store up to 32 bytes

To use this trick we can not simply pass in an address, we'll need to wrap our address in a `struct`

Next, we write a `function` to get the pointer of the storage, taking a single input that will specify the pointer that we want to get (in `bytes32`). This function will return the pointer to the address slot

Basically, this function will return the pointer to the storage r, located at slot from the input

To do this, we will use `assembly`

```js

assembly {
    r.slot := slot
}

```

We can use `StorageSlot` library to get an address stored at any slot

```js
StorageSlot.getAddressSlot(SLOT).value
```

This will return a pointer that is storing `SLOT` address struct

We can use `StorageSlot` library to store an address to any slot

```js
StorageSlot.getAddressSlot(SLOT).value = _addr
```

Basically it says to get the storage pointer at the slot from the input

By using `StorageSlot` library, our proxy is not "Buggy" anymore. We'll be following the Open Zeppelin Transparent Upgradable Proxy contract, so we store implementation and admin in a special place

```js
bytes32 private constant IMPLEMENTATION_SLOT =
        bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);

bytes32 private constant ADMIN_SLOT = bytes32(uint256(keccak256("eip1967.proxy.admin")) - 1);
```

We're substracting 1 by casting the hash into `uint` to make some king of **Hash Collision Attack** difficult to pull off because we don't know the pre-image of the hash

We'll need a `getter` and a `setter` for these! (check code from line 34 to 50) and use these in `constructor()`, `fallback()`, `receive()` and `upgradeTo()`

Then, write public functions to get the admin, and get the implementation: `admin()` using `_getAdmin()` and `implementation()` using `_getImplementation`

## Separate user / admin interfaces

If the caller is an admin, then we'll let him execute some function on the proxy contract, otherwise, if the caller is a user, we'll forward all of the calls to the implementation

Notice that we have some `public` functions in our `Proxy` contract (`IMPLEMENTATION_SLOT`, `ADMIN_SLOT`, `upgradeTo` and `admin` and `implementation`)

The problem is if we were to had these function also inside the implementation contract, then even though we might want to call the function inside the implementation contract, these functions inside the Proxy contract will always be executed (even if it's not the same function, as long as the function selectors are the same)

Inside the Proxy contract, we'll make all the public functions private, but we'll keep `upgradeTo` public since since we need, so we keep it `external` and create `isAdmin` modifier using `_getAdmin` internal function. Remember, admin is no longer a state variable, it's stored in the admin slot

In the modifier, if `msg.sender` is not admin, we'll forward the request to `fallback`

```js

modifier ifAdmin() {
    if (msg.sender == _getAdmin()) {
        _;
    } else {
        _fallback();
    }
}

```

Notice that `fallback` function is `external`, so we can not call it directly, instead we use an intenal function named `_fallback`

```js
function _fallback() private {
    _delegate(_getImplementation());
}

```

We also use this internal function inside `fallback` and `receive`

Then, we expose `upgradeTo()` only if msg.sender is admin

```js
function upgradeTo(address _implementation) external ifAdmin {
    _setImplementation(_implementation);
}
```

Add modifier also for `admin()` function so we can keep it public

## Proxy admin contract

Finally, we'll write a `ProxyAdmin` contract, and this contract will be the admin of the `Proxy` contract

The problem is that since the `admin()` and the `implementation()` also exist inside the `Proxy` contract, if the admin wants to call one of these function inside the `implementation` contract, we wouldn't be able to call it

So 1st thing that we need into the `Proxy` contract is to set the admin of the contract

```js


function changeAdmin(address _admin) external ifAdmin {
    _setAdmin(_admin);
}

```

Now we'll write the `ProxyAdmin` contract, and when we deploy both, the `Proxy` contract and the `ProxyAdmin` contract, we'll call `changeAdmin()` passing-in the address of the `ProxyAdmin` contract

**Note1** Inside of `changeProxyAdmin`, proxy address passed in as argument is marqued as `payable` because the `Proxy` contract has a `fallback` and a `receive`

```js
function changeProxyAdmin(address payable proxy, address _admin) external onlyOwner {
    TransparentUpgradeableProxy(proxy).changeAdmin(_admin);
}
```

**Note2** We wrote read-only function to read the address of the implementation and the address of the admin because in `Proxy` contract `admin()` and `implementation()` are not read-only functions. We had to remore the `view` declaration since we have a `ifAdmin` modifier which have the potential to call the `fallback`

The way we'll call a function that is not read-only: `staticcall` to make it read-only function

`staticcall` is like `call` except that it does not write anything into the blockchain. Note that we're passing-in empty parenthesis inside `abi.encodeCall` because there is no input to pass-in inside the function `admin()`

Finally, we know that when call the `admin()` it will return an address, so we `abi.decode` the response into address

```js
    function getProxyAdmin(address proxy) external view returns (address) {

        (bool ok, bytes memory res) = proxy.staticcall(
            abi.encodeCall(TransparentUpgradeableProxy.admin, ())
        );
        require(ok, "call failed");

        return abi.decode(res, (address));
    }
```

We did something similar to get the address of the implementation inside `getProxyImplementation()`

### Further reading

(...soon)

### Sources

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- ROADMAP -->

## Roadmap

-   [ ] Further reading
-   [ ] Deploy script
-   [ ] Unit test

See the [open issues](https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly.git/issues) for a full list of proposed features (and known issues).

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CONTRIBUTING -->

## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

If you have a suggestion that would make this better, please fork the repo and create a pull request. You can also simply open an issue with the tag "enhancement".
Don't forget to give the project a star! Thanks again!

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- LICENSE -->

## License

Distributed under the MIT License. See `LICENSE.txt` for more information.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- CONTACT -->

## Contact

Reda Aboutika - [@twitter](https://twitter.com/AboutikaR) - reda.aboutika@gmail.com

Project Link: [https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly.git](https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly.git)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- ACKNOWLEDGMENTS -->

## Acknowledgments

-   [Smart Contract Engineer](https://www.smartcontract.engineer/)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->

[contributors-shield]: https://img.shields.io/github/contributors/Aboudoc/Transparent-Upgradable-Proxy-assembly.svg?style=for-the-badge
[contributors-url]: https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/Aboudoc/Transparent-Upgradable-Proxy-assembly.svg?style=for-the-badge
[forks-url]: https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly/network/members
[stars-shield]: https://img.shields.io/github/stars/Aboudoc/Transparent-Upgradable-Proxy-assembly.svg?style=for-the-badge
[stars-url]: https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly/stargazers
[issues-shield]: https://img.shields.io/github/issues/Aboudoc/Transparent-Upgradable-Proxy-assembly.svg?style=for-the-badge
[issues-url]: https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly/issues
[license-shield]: https://img.shields.io/github/license/Aboudoc/Transparent-Upgradable-Proxy-assembly.svg?style=for-the-badge
[license-url]: https://github.com/Aboudoc/Transparent-Upgradable-Proxy-assembly/blob/master/LICENSE.txt
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://www.linkedin.com/in/r%C3%A9da-aboutika-34305453/?originalSubdomain=fr
[product-screenshot]: https://ethereum.org/static/28214bb68eb5445dcb063a72535bc90c/9019e/hero.webp
[Hardhat]: https://img.shields.io/badge/Hardhat-20232A?style=for-the-badge&logo=hardhat&logoColor=61DAFB
[Hardhat-url]: https://hardhat.org/
[Ethers.js]: https://img.shields.io/badge/ethers.js-000000?style=for-the-badge&logo=ethersdotjs&logoColor=white
[Ethers-url]: https://docs.ethers.org/v5/
[Vue.js]: https://img.shields.io/badge/Vue.js-35495E?style=for-the-badge&logo=vuedotjs&logoColor=4FC08D
[Vue-url]: https://vuejs.org/
[Angular.io]: https://img.shields.io/badge/Angular-DD0031?style=for-the-badge&logo=angular&logoColor=white
[Angular-url]: https://angular.io/
[Svelte.dev]: https://img.shields.io/badge/Svelte-4A4A55?style=for-the-badge&logo=svelte&logoColor=FF3E00
[Svelte-url]: https://svelte.dev/
[Laravel.com]: https://img.shields.io/badge/Laravel-FF2D20?style=for-the-badge&logo=laravel&logoColor=white
[Laravel-url]: https://laravel.com
[Bootstrap.com]: https://img.shields.io/badge/Bootstrap-563D7C?style=for-the-badge&logo=bootstrap&logoColor=white
[Bootstrap-url]: https://getbootstrap.com
[JQuery.com]: https://img.shields.io/badge/jQuery-0769AD?style=for-the-badge&logo=jquery&logoColor=white
[JQuery-url]: https://jquery.com
