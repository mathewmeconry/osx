---
title: Upgrade the Plugin
---

## How to upgrade an Upgradeable Plugin

Updating an Upgradeable plugin means we want to direct the implementation of our functionality to a new build, rather than the existing one.

In this tutorial, we will go through how to update the version of an Upgradeable plugin and each component needed.

### 1. Create the new build implementation contract

Firstly, you want to create the new build implementation contract the plugin should use. You can read more about how to do this in the ["How to create a subsequent build implementation to an Upgradeable Plugin" guide](./04-subsequent-builds.md).

```solidity
// SPDX-License-Identifier: AGPL-3.0-or-later
pragma solidity 0.8.17;

import {IDAO, PluginUUPSUpgradeable} from '@aragon/osx/core/plugin/PluginUUPSUpgradeable.sol';

/// @title SimpleStorage build 2
contract SimpleStorageBuild2 is PluginUUPSUpgradeable {
  bytes32 public constant STORE_PERMISSION_ID = keccak256('STORE_PERMISSION');

  uint256 public number; // added in build 1
  address public account; // added in build 2

  /// @notice Initializes the plugin when build 2 is installed.
  function initializeBuild2(
    IDAO _dao,
    uint256 _number,
    address _account
  ) external reinitializer(2) {
    __PluginUUPSUpgradeable_init(_dao);
    number = _number;
    account = _account;
  }

  /// @notice Initializes the plugin when the update from build 1 to build 2 is applied.
  /// @dev The initialization of `SimpleStorageBuild1` has already happened.
  function initializeFromBuild1(address _account) external reinitializer(2) {
    account = _account;
  }

  function storeNumber(uint256 _number) external auth(STORE_PERMISSION_ID) {
    number = _number;
  }

  function storeAccount(address _account) external auth(STORE_PERMISSION_ID) {
    account = _account;
  }
}
```

### 2. Write a new Plugin Setup contract

In order to do update a plugin, we need a `prepareUpdate()` function in our Plugin Setup contract which points the functionality to a new build, as we described in the ["How to create a subsequent build implementation to an Upgradeable Plugin" guide](./04-subsequent-builds.md). The `prepareUpdate()` function must transition the plugin from the old build state into the new one so that it ends up having the same permissions (and helpers) as if it had been freshly installed.

In contrast to the original build 1, build 2 requires two input arguments: `uint256 _number` and `address _account` that we decode from the bytes-encoded input `_data`.

```solidity
// SPDX-License-Identifier: AGPL-3.0-or-later

pragma solidity 0.8.17;

import {PermissionLib} from '@aragon/osx/core/permission/PermissionLib.sol';
import {PluginSetup, IPluginSetup} from '@aragon/osx/framework/plugin/setup/PluginSetup.sol';
import {SimpleStorageBuild2} from './SimpleStorageBuild2.sol';

/// @title SimpleStorageSetup build 2
contract SimpleStorageBuild2Setup is PluginSetup {
  address private immutable simpleStorageImplementation;

  constructor() {
    simpleStorageImplementation = address(new SimpleStorageBuild2());
  }

  /// @inheritdoc IPluginSetup
  function prepareInstallation(
    address _dao,
    bytes memory _data
  ) external returns (address plugin, PreparedSetupData memory preparedSetupData) {
    (uint256 _number, address _account) = abi.decode(_data, (uint256, address));

    plugin = createERC1967Proxy(
      simpleStorageImplementation,
      abi.encodeWithSelector(SimpleStorageBuild2.initializeBuild2.selector, _dao, _number, _account)
    );

    PermissionLib.MultiTargetPermission[]
      memory permissions = new PermissionLib.MultiTargetPermission[](1);

    permissions[0] = PermissionLib.MultiTargetPermission({
      operation: PermissionLib.Operation.Grant,
      where: plugin,
      who: _dao,
      condition: PermissionLib.NO_CONDITION,
      permissionId: SimpleStorageBuild2(this.implementation()).STORE_PERMISSION_ID()
    });

    preparedSetupData.permissions = permissions;
  }

  /// @inheritdoc IPluginSetup
  function prepareUpdate(
    address _dao,
    uint16 _currentBuild,
    SetupPayload calldata _payload
  )
    external
    pure
    override
    returns (bytes memory initData, PreparedSetupData memory preparedSetupData)
  {
    (_dao, preparedSetupData);

    if (_currentBuild == 0) {
      address _account = abi.decode(_payload.data, (address));
      initData = abi.encodeWithSelector(
        SimpleStorageBuild2.initializeFromBuild1.selector,
        _account
      );
    }
  }

  /// @inheritdoc IPluginSetup
  function prepareUninstallation(
    address _dao,
    SetupPayload calldata _payload
  ) external view returns (PermissionLib.MultiTargetPermission[] memory permissions) {
    permissions = new PermissionLib.MultiTargetPermission[](1);

    permissions[0] = PermissionLib.MultiTargetPermission({
      operation: PermissionLib.Operation.Revoke,
      where: _payload.plugin,
      who: _dao,
      condition: PermissionLib.NO_CONDITION,
      permissionId: SimpleStorageBuild2(this.implementation()).STORE_PERMISSION_ID()
    });
  }

  /// @inheritdoc IPluginSetup
  function implementation() external view returns (address) {
    return simpleStorageImplementation;
  }
}
```

The key thing to review in this new Plugin Setup contract is its `prepareUpdate()` function. The function only contains a condition checking from which build number the update is transitioning to build `2`. Here, it is the build number `1` as this is the only update path we support. Inside, we decode the `address _account` input argument provided with `bytes _date` and pass it to the `initializeFromBuild1` function taking care of intializing the storage that was added in this build.

### 3. Future builds

For each build we add, we will need to add a `prepareUpdate()` function with any parameters needed to update to that implementation.

In this third build, for example, we are modifying the bytecode of the plugin.

<details>
<summary>Third plugin build example, modifying the plugin's bytecode</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0-or-later
pragma solidity 0.8.17;

import {IDAO, PluginUUPSUpgradeable} from '@aragon/osx/core/plugin/PluginUUPSUpgradeable.sol';

/// @title SimpleStorage build 3
contract SimpleStorageBuild3 is PluginUUPSUpgradeable {
  bytes32 public constant STORE_NUMBER_PERMISSION_ID = keccak256('STORE_NUMBER_PERMISSION'); // changed in build 3
  bytes32 public constant STORE_ACCOUNT_PERMISSION_ID = keccak256('STORE_ACCOUNT_PERMISSION'); // added in build 3

  uint256 public number; // added in build 1
  address public account; // added in build 2

  // added in build 3
  event NumberStored(uint256 number);
  event AccountStored(address account);
  error AlreadyStored();

  /// @notice Initializes the plugin when build 3 is installed.
  function initializeBuild3(
    IDAO _dao,
    uint256 _number,
    address _account
  ) external reinitializer(3) {
    __PluginUUPSUpgradeable_init(_dao);
    number = _number;
    account = _account;

    emit NumberStored({number: _number});
    emit AccountStored({account: _account});
  }

  /// @notice Initializes the plugin when the update from build 2 to build 3 is applied.
  /// @dev The initialization of `SimpleStorageBuild2` has already happened.
  function initializeFromBuild2() external reinitializer(3) {
    emit NumberStored({number: number});
    emit AccountStored({account: account});
  }

  /// @notice Initializes the plugin when the update from build 1 to build 3 is applied.
  /// @dev The initialization of `SimpleStorageBuild1` has already happened.
  function initializeFromBuild1(address _account) external reinitializer(3) {
    account = _account;

    emit NumberStored({number: number});
    emit AccountStored({account: _account});
  }

  function storeNumber(uint256 _number) external auth(STORE_NUMBER_PERMISSION_ID) {
    if (_number == number) revert AlreadyStored();

    number = _number;

    emit NumberStored({number: _number});
  }

  function storeAccount(address _account) external auth(STORE_ACCOUNT_PERMISSION_ID) {
    if (_account == account) revert AlreadyStored();

    account = _account;

    emit AccountStored({account: _account});
  }
}
```

</details>

With each new build implementation, we will need to udate the Plugin Setup contract to be able to update to that new version. We do this through updating the `prepareUpdate()` function to support any new features that need to be set up.

<details>
<summary>Third plugin setup example, modifying <code>prepareUpdate</code> function</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0-or-later

pragma solidity 0.8.17;

import {PermissionLib} from '@aragon/osx/core/permission/PermissionLib.sol';
import {PluginSetup, IPluginSetup} from '@aragon/osx/framework/plugin/setup/PluginSetup.sol';
import {SimpleStorageBuild2} from '../build2/SimpleStorageBuild2.sol';
import {SimpleStorageBuild3} from './SimpleStorageBuild3.sol';

/// @title SimpleStorageSetup build 3
contract SimpleStorageBuild3Setup is PluginSetup {
  address private immutable simpleStorageImplementation;

  constructor() {
    simpleStorageImplementation = address(new SimpleStorageBuild3());
  }

  /// @inheritdoc IPluginSetup
  function prepareInstallation(
    address _dao,
    bytes memory _data
  ) external returns (address plugin, PreparedSetupData memory preparedSetupData) {
    (uint256 _number, address _account) = abi.decode(_data, (uint256, address));

    plugin = createERC1967Proxy(
      simpleStorageImplementation,
      abi.encodeWithSelector(SimpleStorageBuild3.initializeBuild3.selector, _dao, _number, _account)
    );

    PermissionLib.MultiTargetPermission[]
      memory permissions = new PermissionLib.MultiTargetPermission[](2);

    permissions[0] = PermissionLib.MultiTargetPermission({
      operation: PermissionLib.Operation.Grant,
      where: plugin,
      who: _dao,
      condition: PermissionLib.NO_CONDITION,
      permissionId: SimpleStorageBuild3(this.implementation()).STORE_NUMBER_PERMISSION_ID()
    });

    permissions[1] = permissions[0];
    permissions[1].permissionId = SimpleStorageBuild3(this.implementation())
      .STORE_ACCOUNT_PERMISSION_ID();

    preparedSetupData.permissions = permissions;
  }

  /// @inheritdoc IPluginSetup
  function prepareUpdate(
    address _dao,
    uint16 _currentBuild,
    SetupPayload calldata _payload
  )
    external
    view
    override
    returns (bytes memory initData, PreparedSetupData memory preparedSetupData)
  {
    if (_currentBuild == 0) {
      address _account = abi.decode(_payload.data, (address));
      initData = abi.encodeWithSelector(
        SimpleStorageBuild3.initializeFromBuild1.selector,
        _account
      );
    } else if (_currentBuild == 1) {
      initData = abi.encodeWithSelector(SimpleStorageBuild3.initializeFromBuild2.selector);
    }

    PermissionLib.MultiTargetPermission[]
      memory permissions = new PermissionLib.MultiTargetPermission[](3);
    permissions[0] = PermissionLib.MultiTargetPermission({
      operation: PermissionLib.Operation.Revoke,
      where: _dao,
      who: _payload.plugin,
      condition: PermissionLib.NO_CONDITION,
      permissionId: keccak256('STORE_PERMISSION')
    });

    permissions[1] = permissions[0];
    permissions[1].operation = PermissionLib.Operation.Grant;
    permissions[1].permissionId = SimpleStorageBuild3(this.implementation())
      .STORE_NUMBER_PERMISSION_ID();

    permissions[2] = permissions[1];
    permissions[2].permissionId = SimpleStorageBuild3(this.implementation())
      .STORE_ACCOUNT_PERMISSION_ID();

    preparedSetupData.permissions = permissions;
  }

  /// @inheritdoc IPluginSetup
  function prepareUninstallation(
    address _dao,
    SetupPayload calldata _payload
  ) external view returns (PermissionLib.MultiTargetPermission[] memory permissions) {
    permissions = new PermissionLib.MultiTargetPermission[](2);

    permissions[0] = PermissionLib.MultiTargetPermission({
      operation: PermissionLib.Operation.Revoke,
      where: _payload.plugin,
      who: _dao,
      condition: PermissionLib.NO_CONDITION,
      permissionId: SimpleStorageBuild3(this.implementation()).STORE_NUMBER_PERMISSION_ID()
    });

    permissions[1] = permissions[1];
    permissions[1].permissionId = SimpleStorageBuild3(this.implementation())
      .STORE_ACCOUNT_PERMISSION_ID();
  }

  /// @inheritdoc IPluginSetup
  function implementation() external view returns (address) {
    return simpleStorageImplementation;
  }
}
```

</details>

In this case, the `prepareUpdate()` function only contains a condition checking from which build number the update is transitioning to build 2. Here, we can update from build 0 or build 1 and different operations must happen for each case to transition to `SimpleAdminBuild3`.

In the first case, `initializeFromBuild1` is called taking care of intializing `address _account` that was added in build 1 and emitting the events added in build 2.

In the second case, `initializeFromBuild2` is called taking care of intializing the build. Here, only the two events will be emitted.

Lastly, the `prepareUpdate()` function takes care of modifying the permissions by revoking the `STORE_PERMISSION_ID` and granting the more specific `STORE_NUMBER_PERMISSION_ID` and `STORE_ACCOUNT)PERMISSION_ID` permissions, that are also granted if build 2 is freshly installed. This must happen for both update paths so this code is outside the `if` statements.
