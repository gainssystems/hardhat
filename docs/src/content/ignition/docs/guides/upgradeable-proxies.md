# Using Hardhat Ignition with upgradeable proxies

When developing smart contracts, you may decide to use an upgradeable proxy pattern to allow for future upgrades to your contracts. This guide will explain how to create Ignition modules to deploy and interact with your upgradeable proxy contracts.

While there are several different proxy patterns, each with their own tradeoffs, this guide will focus on the [TransparentUpgradeableProxy](https://docs.openzeppelin.com/contracts/5.x/api/proxy#TransparentUpgradeableProxy) pattern. You can read more about upgradeable proxy patterns [on OpenZeppelin's blog](https://blog.openzeppelin.com/proxy-patterns).

:::tip

The finished code for this guide can be found in the [Hardhat Ignition monorepo](https://github.com/NomicFoundation/hardhat-ignition/tree/development/examples/upgradeable)

:::

## Getting to know our contracts

Before we start writing our Ignition modules, let's take a look at the contracts we'll be deploying and interacting with.

First, inside our `contracts` directory, we'll create a file called `Demo.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

// A contrived example of a contract that can be upgraded
contract Demo {
  function version() public pure returns (string memory) {
    return "1.0.0";
  }
}
```

This is the contract that we'll be upgrading. It's a simple contract that returns a version string.

Let's go ahead and create our upgraded version of this contract in a new file called `DemoV2.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

// A contrived example of a contract that can be upgraded
contract DemoV2 {
  function version() public pure returns (string memory) {
    return "2.0.0";
  }
}
```

This contract is identical to the first one, except that it returns an updated version string.

Finally, we'll create a file called `Proxies.sol` to import our proxy contracts. This file will look a little different than the others:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

import "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";
import "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
```

Because we're using the OpenZeppelin proxy contracts, we need to import them here to make sure Hardhat knows to compile them. This will ensure that their artifacts are available for Hardhat Ignition to use later when we're writing our Ignition modules.

## Writing our Ignition modules

Inside our `ignition` directory, we'll create a directory called `modules`, if one doesn't already exist. Inside this directory, we'll create a file called `ProxyModule.js` (or `ProxyModule.ts` if you're using TypeScript). Inside this file, we'll break up our Ignition module into three parts.

### Part 1: Deploying our proxies

As always, we'll begin by importing `buildModule` from `@nomicfoundation/hardhat-ignition/modules`, then we'll define our first module, which we'll call `ProxyModule`:

::::tabsgroup{options="TypeScript,JavaScript"}

:::tab{value="TypeScript"}

```typescript
import { buildModule } from "@nomicfoundation/hardhat-ignition/modules";

const proxyModule = buildModule("ProxyModule", (m) => {
  const proxyAdminOwner = m.getAccount(0);

  const demo = m.contract("Demo");

  const proxy = m.contract("TransparentUpgradeableProxy", [
    demo,
    proxyAdminOwner,
    "0x",
  ]);

  const proxyAdminAddress = m.readEventArgument(
    proxy,
    "AdminChanged",
    "newAdmin"
  );

  const proxyAdmin = m.contractAt("ProxyAdmin", proxyAdminAddress);

  return { proxyAdmin, proxy };
});
```

:::

:::tab{value="JavaScript"}

```javascript
const { buildModule } = require("@nomicfoundation/hardhat-ignition/modules");

const proxyModule = buildModule("ProxyModule", (m) => {
  const proxyAdminOwner = m.getAccount(0);

  const demo = m.contract("Demo");

  const proxy = m.contract("TransparentUpgradeableProxy", [
    demo,
    proxyAdminOwner,
    "0x",
  ]);

  const proxyAdminAddress = m.readEventArgument(
    proxy,
    "AdminChanged",
    "newAdmin"
  );

  const proxyAdmin = m.contractAt("ProxyAdmin", proxyAdminAddress);

  return { proxyAdmin, proxy };
});
```

:::

::::

Let's break down what's happening here.

First, we're getting our account that will own the ProxyAdmin contract. This account will not be able to interact with the proxy, but it will be able to upgrade it. In this case, we'll use the first account in our Hardhat accounts array.

Next, we deploy our `Demo` contract. This will be the contract that we'll upgrade.

Then, we deploy our `TransparentUpgradeableProxy` contract. This contract will be deployed with the `Demo` contract as its implementation, and the `proxyAdminOwner` account as its owner. The third argument is the initialization code, which we'll leave blank for now by setting to an empty hex string (`"0x"`).

When we deploy the proxy, it will automatically create a new `ProxyAdmin` contract within its constructor. We'll need to get the address of this contract so that we can interact with it later. To do this, we'll use the `m.readEventArgument(...)` method to read the `newAdmin` argument from the `AdminChanged` event that is emitted when the proxy is deployed.

Finally, we'll use the `m.contractAt(...)` method to get the `ProxyAdmin` contract at the address we retrieved from the event and return it along with the proxy.

### Part 2: Upgrading our proxy

Now that we have our proxy deployed, we want to upgrade it. To do this, within the same file, we'll create a new module called `UpgradeModule`:

```js
const upgradeModule = buildModule("UpgradeModule", (m) => {
  const proxyAdminOwner = m.getAccount(0);

  const { proxyAdmin, proxy } = m.useModule(proxyModule);

  const demoV2 = m.contract("DemoV2");

  m.call(proxyAdmin, "upgradeAndCall", [proxy, demoV2, "0x"], {
    from: proxyAdminOwner,
  });

  return { proxyAdmin, proxy };
});
```

This module begins the same way as the previous one, by getting the account that owns the `ProxyAdmin` contract. We'll use this in a moment to upgrade the proxy.

Next, we use the `m.useModule(...)` method to get the `ProxyAdmin` and proxy contracts from the previous module. This will ensure that the proxy is deployed before we try to upgrade it.

Then, we deploy our `DemoV2` contract. This will be the contract that we'll upgrade our proxy to.

Finally, we call the `upgradeAndCall` method on the `ProxyAdmin` contract. This method takes three arguments: the proxy contract, the new implementation contract, and a data parameter that can be used to call an additional function. We don't need it right now, so we'll leave it blank by setting it to an empty hex string (`"0x"`). We also provide the `from` option to ensure that the upgrade is called from the owner of the `ProxyAdmin` contract.

Lastly, we again return the `ProxyAdmin` and proxy contracts so that we can use them in our next module.

### Part 3: Interacting with our proxy

Now that we have our proxy deployed and upgraded, we're ready to interact with it. To do this, we'll create a new module called `InteractableModule`:

```js
const interactableModule = buildModule("InteractableModule", (m) => {
  const { proxy } = m.useModule(upgradeModule);

  const demo = m.contractAt("DemoV2", proxy);

  return { demo };
});
```

First, as before, we use the `m.useModule(...)` method to get the proxy contract from the previous module.

Then, we use `m.contractAt("DemoV2", proxy)` to get the `DemoV2` contract at the address of the proxy. This will allow us to interact with the proxy as if it were the `DemoV2` contract itself.

Finally, we return the `DemoV2` contract instance so that we can use it in other modules or tests.

As a final step, we'll export `interactableModule` from our file so that we can deploy it and use it in our tests:

::::tabsgroup{options="TypeScript,JavaScript"}

:::tab{value="TypeScript"}

```typescript
export default interactableModule;
```

:::

:::tab{value="JavaScript"}

```javascript
module.exports = interactableModule;
```

:::

::::

## Testing our Ignition modules

Now that we've written our Ignition modules for deploying and interacting with our proxy, let's write a simple test to make sure everything works as expected.

Inside our `test` directory, we'll create a file called `ProxyDemo.js` (or `ProxyDemo.ts` if you're using TypeScript):

::::tabsgroup{options="TypeScript,JavaScript"}

:::tab{value="TypeScript"}

```typescript
import { expect } from "chai";

import ProxyModule from "../ignition/modules/ProxyModule";

describe("Demo Proxy", function () {
  describe("Upgrading", function () {
    it("Should have upgraded the proxy to DemoV2", async function () {
      const [owner, otherAccount] = await ethers.getSigners();

      const { demo } = await ignition.deploy(ProxyModule);

      expect(await demo.connect(otherAccount).version()).to.equal("2.0.0");
    });
  });
});
```

:::

:::tab{value="JavaScript"}

```javascript
const { expect } = require("chai");

const ProxyModule = require("../ignition/modules/ProxyModule");

describe("Demo Proxy", function () {
  describe("Upgrading", function () {
    it("Should have upgraded the proxy to DemoV2", async function () {
      const [owner, otherAccount] = await ethers.getSigners();

      const { demo } = await ignition.deploy(ProxyModule);

      expect(await demo.connect(otherAccount).version()).to.equal("2.0.0");
    });
  });
});
```

:::

::::

Here we use Hardhat Ignition to deploy our imported module. Then, we use the `demo` contract instance that is returned to call the `version` method and make sure that it returns the correct version string.

## Further reading

In this guide we learned how to use Hardhat Ignition to deploy and interact with an upgradeable proxy contract. While this specific example may not be useful in a production environment, it can be used as a starting point for more complex upgradeable proxy patterns.

Here are some additional resources to learn more about topics discussed in this guide:

- [OpenZeppelin's blog post on proxy patterns](https://blog.openzeppelin.com/proxy-patterns)
- [OpenZeppelin's documentation on the TransparentUpgradeableProxy used in this guide](https://docs.openzeppelin.com/contracts/5.x/api/proxy#TransparentUpgradeableProxy)
- [Hardhat Ignition's documentation on creating Ignition modules](/ignition/docs/guides/creating-modules)
- [Hardhat Ignition's documentation on testing Ignition modules](/ignition/docs/guides/tests)