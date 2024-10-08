---
eip: 6900
title: Modular Smart Contract Accounts and Plugins
description: Interfaces for composable contract accounts optionally supporting upgradability and introspection
author: Adam Egyed (@adamegyed), Fangting Liu (@trinity-0111), Jay Paik (@jaypaik), Yoav Weiss (@yoavw), Huawei Gu (@huaweigu), Daniel Lim (@dlim-circle), Zhiyu Zhang (@ZhiyuCircle), Ruben Koch (@0xrubes), David Philipson (@dphilipson), Howy Ho (@howydev)
discussions-to: https://ethereum-magicians.org/t/eip-modular-smart-contract-accounts-and-plugins/13885
status: Draft
type: Standards Track
category: ERC
created: 2023-04-18
requires: 165, 4337
---

## Abstract

This proposal standardizes smart contract accounts and account plugins, which are smart contract interfaces that allow for composable logic within smart contract accounts. This proposal is compliant with [ERC-4337](../04337.md), and takes inspiration from [ERC-2535](../02535.md) when defining interfaces for updating and querying modular function implementations.

This modular approach splits account functionality into three categories, implements them in external contracts, and defines an expected execution flow from accounts.

## Motivation

One of the goals that ERC-4337 accomplishes is abstracting the logic for execution and validation to each smart contract account.

Many new features of accounts can be built by customizing the logic that goes into the validation and execution steps. Examples of such features include session keys, subscriptions, spending limits, and role-based access control. Currently, some of these features are implemented natively by specific smart contract accounts, and others are able to be implemented by plugin systems. Examples of proprietary plugin systems include Safe modules and ZeroDev plugins.

However, managing multiple account instances provides a worse user experience, fragmenting accounts across supported features and security configurations. Additionally, it requires plugin developers to choose which platforms to support, causing either platform lock-in or duplicated development effort.

We propose a standard that coordinates the implementation work between plugin developers and wallet developers. This standard defines a modular smart contract account capable of supporting all standard-conformant plugins. This allows users to have greater portability of their data, and for plugin developers to not have to choose specific account implementations to support.

![diagram showing relationship between accounts and plugins with modular functions](./assets/MSCA_Shared_Components_Diagram.svg)

We take inspiration from ERC-2535's diamond pattern for routing execution based on function selectors, and create a similarly composable account. However, the standard does not require the multi-facet proxy pattern.

These plugins can contain execution logic, validation schemes, and hooks. Validation schemes define the circumstances under which the smart contract account will approve actions taken on its behalf, while hooks allow for pre- and post-execution controls.

Accounts adopting this standard will support modular, upgradable execution and validation logic. Defining this as a standard for smart contract accounts will make plugins easier to develop securely and will allow for greater interoperability.

Goals:

- Provide standards for how validation, execution, and hook functions for smart contract accounts should be written.
- Provide standards for how compliant accounts should add, update, remove, and inspect plugins.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Terms

- An **account** (or **smart contract account, SCA**) is a smart contract that can be used to send transactions and hold digital assets. It implements the `IAccount` interface from ERC-4337.
- A **modular account** (or **modular smart contract account, MSCA**) is an account that supports modular functions. There are three types of modular functions:
  - **Validation functions** validate the caller's authenticity and authority to the account.
  - **Execution functions** execute any custom logic allowed by the account.
  - **Hooks** execute custom logic and checks before and/or after an execution function or validation function.
- A **validation function** is a function that validates authentication and authorization of a caller to the account. There are two types of validation functions:
  - **User Operation Validation** functions handle calls to `validateUserOp` and check the validity of an ERC-4337 user operation.
  - **Runtime Validation** functions run before an execution function when not called via a user operation, and enforce checks. Common checks include allowing execution only by an owner.
- An **execution function** is a smart contract function that defines the main execution step of a function for a modular account.
- The **standard execute** functions are two specific execute functions that are implemented natively by the modular account, and not on a plugin. These allow for open-ended execution.
- A **hook** is a smart contract function executed before or after another function, with the ability to modify state or cause the entire call to revert. There are four types of hooks:
  - **Pre User Operation Validation Hook** functions run before user operation validation functions. These can enforce permissions on what actions a validation function may perform via user operations.
  - **Pre Runtime Validation Hook** functions run before runtime validation functions. These can enforce permissions on what actions a validation function may perform via direct calls.
  - **Pre Execution Hook** functions run before an execution function. They may optionally return data to be consumed by their related post execution hook functions.
  - **Post Execution Hook** functions run after an execution function. They may optionally take returned data from their related pre execution hook functions.
- An **associated function** refers to either a validation function or a hook.
- A **native function** refers to a function implemented natively by the modular account, as opposed to a function added by a plugin.
- A **plugin** is a deployed smart contract that hosts any amount of the above three kinds of modular functions: execution functions, validation functions, or hooks.
- A plugin **manifest** is responsible for describing the execution functions, validation functions, and hooks that will be configured on the MSCA during installation, as well as the plugin’s metadata, dependency requirements, and permissions.

### Overview

A modular account handles two kinds of calls: either from the `Entrypoint` through ERC-4337, or through direct calls from externally owned accounts (EOAs) and other smart contracts. This standard supports both use cases.

A call to the smart contract account can be broken down into the steps as shown in the diagram below. The validation steps validate if the caller is allowed to perform the call. The pre execution hook step can be used to do any pre execution checks or updates. It can also be used along with the post execution hook step to perform additional actions or verification. The execution step performs a defined task or collection of tasks.

![diagram showing call flow within an modular account](./assets/Modular_Account_Call_Flow.svg)

The following diagram shows permitted plugin execution flows. During a plugin's execution step from the above diagram, the plugin may perform a "Plugin Execution Function", using either `executeFromPlugin` or `executeFromPluginExternal`. These can be used by plugins to execute using the account's context.

- `executeFromPlugin` handles calls to other installed plugin's execution function on the modular account.
- `executeFromPluginExternal` handles calls to external addresses.

![diagram showing a plugin execution flow](./assets/Plugin_Execution_Flow.svg)

Each step is modular, supporting different implementations for each execution function, and composable, supporting multiple steps through hooks. Combined, these allow for open-ended programmable accounts.

### Interfaces

**Modular Smart Contract Accounts** **MUST** implement

- `IAccount.sol` from [ERC-4337](../04337.md).
- `IPluginManager.sol` to support installing and uninstalling plugins.
- `IStandardExecutor.sol` to support open-ended execution. **Calls to plugins through this SHOULD revert.**
- `IPluginExecutor.sol` to support execution from plugins. **Calls to plugins through `executeFromPluginExternal` SHOULD revert.**

**Modular Smart Contract Accounts** **MAY** implement

- `IAccountLoupe.sol` to support visibility in plugin configuration on-chain.

**Plugins** **MUST** implement

- `IPlugin.sol` described below and implement [ERC-165](../00165.md) for `IPlugin`.

#### `IPluginManager.sol`

Plugin manager interface. Modular Smart Contract Accounts **MUST** implement this interface to support installing and uninstalling plugins.

```solidity
// Treats the first 20 bytes as an address, and the last byte as a function identifier.
type FunctionReference is bytes21;

interface IPluginManager {
    event PluginInstalled(address indexed plugin, bytes32 manifestHash, FunctionReference[] dependencies);

    event PluginUninstalled(address indexed plugin, bool indexed onUninstallSucceeded);

    /// @notice Install a plugin to the modular account.
    /// @param plugin The plugin to install.
    /// @param manifestHash The hash of the plugin manifest.
    /// @param pluginInstallData Optional data to be decoded and used by the plugin to setup initial plugin data
    /// for the modular account.
    /// @param dependencies The dependencies of the plugin, as described in the manifest. Each FunctionReference
    /// MUST be composed of an installed plugin's address and a function ID of its validation function.
    function installPlugin(
        address plugin,
        bytes32 manifestHash,
        bytes calldata pluginInstallData,
        FunctionReference[] calldata dependencies
    ) external;

    /// @notice Uninstall a plugin from the modular account.
    /// @param plugin The plugin to uninstall.
    /// @param config An optional, implementation-specific field that accounts may use to ensure consistency
    /// guarantees.
    /// @param pluginUninstallData Optional data to be decoded and used by the plugin to clear plugin data for the
    /// modular account.
    function uninstallPlugin(address plugin, bytes calldata config, bytes calldata pluginUninstallData) external;
}

```

#### `IStandardExecutor.sol`

Standard execute interface. Modular Smart Contract Accounts **MUST** implement this interface to support open-ended execution.

Standard execute functions SHOULD check whether the call's target implements the `IPlugin` interface via ERC-165.

**If the target is a plugin, the call SHOULD revert.** This prevents accidental misconfiguration or misuse of plugins (both installed and uninstalled).

```solidity
struct Call {
    // The target address for the account to call.
    address target;
    // The value to send with the call.
    uint256 value;
    // The calldata for the call.
    bytes data;
}

interface IStandardExecutor {
    /// @notice Standard execute method.
    /// @dev If the target is a plugin, the call SHOULD revert.
    /// @param target The target address for account to call.
    /// @param value The value to send with the call.
    /// @param data The calldata for the call.
    /// @return The return data from the call.
    function execute(address target, uint256 value, bytes calldata data) external payable returns (bytes memory);

    /// @notice Standard executeBatch method.
    /// @dev If the target is a plugin, the call SHOULD revert. If any of the calls revert, the entire batch MUST
    /// revert.
    /// @param calls The array of calls.
    /// @return An array containing the return data from the calls.
    function executeBatch(Call[] calldata calls) external payable returns (bytes[] memory);
}
```

#### `IPluginExecutor.sol`

Execution interface for calls made from plugins. Modular Smart Contract Accounts **MUST** implement this interface to support execution from plugins.

The `executeFromPluginExternal` function SHOULD check whether the call's target implements the `IPlugin` interface via ERC-165.

**If the target of `executeFromPluginExternal` function is a plugin, the call SHOULD revert.**

This prevents accidental misconfiguration or misuse of plugins (both installed and uninstalled). Installed plugins MAY interact with other installed plugins via the `executeFromPlugin` function.

```solidity
interface IPluginExecutor {
    /// @notice Execute a call from a plugin through the account.
    /// @dev Permissions must be granted to the calling plugin for the call to go through.
    /// @param data The calldata to send to the account.
    /// @return The return data from the call.
    function executeFromPlugin(bytes calldata data) external payable returns (bytes memory);

    /// @notice Execute a call from a plugin to a non-plugin address.
    /// @dev If the target is a plugin, the call SHOULD revert. Permissions must be granted to the calling plugin
    /// for the call to go through.
    /// @param target The address to be called.
    /// @param value The value to send with the call.
    /// @param data The calldata to send to the target.
    /// @return The return data from the call.
    function executeFromPluginExternal(address target, uint256 value, bytes calldata data)
        external
        payable
        returns (bytes memory);
}
```

#### `IAccountLoupe.sol`

Plugin inspection interface. Modular Smart Contract Accounts **MAY** implement this interface to support visibility in plugin configuration on-chain.

```solidity
interface IAccountLoupe {
    /// @notice Config for an execution function, given a selector.
    struct ExecutionFunctionConfig {
        address plugin;
        FunctionReference userOpValidationFunction;
        FunctionReference runtimeValidationFunction;
    }

    /// @notice Pre and post hooks for a given selector.
    /// @dev It's possible for one of either `preExecHook` or `postExecHook` to be empty.
    struct ExecutionHooks {
        FunctionReference preExecHook;
        FunctionReference postExecHook;
    }

    /// @notice Get the validation functions and plugin address for a selector.
    /// @dev If the selector is a native function, the plugin address will be the address of the account.
    /// @param selector The selector to get the configuration for.
    /// @return The configuration for this selector.
    function getExecutionFunctionConfig(bytes4 selector) external view returns (ExecutionFunctionConfig memory);

    /// @notice Get the pre and post execution hooks for a selector.
    /// @param selector The selector to get the hooks for.
    /// @return The pre and post execution hooks for this selector.
    function getExecutionHooks(bytes4 selector) external view returns (ExecutionHooks[] memory);

    /// @notice Get the pre user op and runtime validation hooks associated with a selector.
    /// @param selector The selector to get the hooks for.
    /// @return preUserOpValidationHooks The pre user op validation hooks for this selector.
    /// @return preRuntimeValidationHooks The pre runtime validation hooks for this selector.
    function getPreValidationHooks(bytes4 selector)
        external
        view
        returns (
            FunctionReference[] memory preUserOpValidationHooks,
            FunctionReference[] memory preRuntimeValidationHooks
        );

    /// @notice Get an array of all installed plugins.
    /// @return The addresses of all installed plugins.
    function getInstalledPlugins() external view returns (address[] memory);
}
```

#### `IPlugin.sol`

Plugin interface. Plugins **MUST** implement this interface to support plugin management and interactions with MSCAs.

```solidity
interface IPlugin {
    /// @notice Initialize plugin data for the modular account.
    /// @dev Called by the modular account during `installPlugin`.
    /// @param data Optional bytes array to be decoded and used by the plugin to setup initial plugin data for the modular account.
    function onInstall(bytes calldata data) external;

    /// @notice Clear plugin data for the modular account.
    /// @dev Called by the modular account during `uninstallPlugin`.
    /// @param data Optional bytes array to be decoded and used by the plugin to clear plugin data for the modular account.
    function onUninstall(bytes calldata data) external;

    /// @notice Run the pre user operation validation hook specified by the `functionId`.
    /// @dev Pre user operation validation hooks MUST NOT return an authorizer value other than 0 or 1.
    /// @param functionId An identifier that routes the call to different internal implementations, should there be more than one.
    /// @param userOp The user operation.
    /// @param userOpHash The user operation hash.
    /// @return Packed validation data for validAfter (6 bytes), validUntil (6 bytes), and authorizer (20 bytes).
    function preUserOpValidationHook(uint8 functionId, UserOperation memory userOp, bytes32 userOpHash) external returns (uint256);

    /// @notice Run the user operation validationFunction specified by the `functionId`.
    /// @param functionId An identifier that routes the call to different internal implementations, should there be
    /// more than one.
    /// @param userOp The user operation.
    /// @param userOpHash The user operation hash.
    /// @return Packed validation data for validAfter (6 bytes), validUntil (6 bytes), and authorizer (20 bytes).
    function userOpValidationFunction(uint8 functionId, UserOperation calldata userOp, bytes32 userOpHash)
        external
        returns (uint256);

    /// @notice Run the pre runtime validation hook specified by the `functionId`.
    /// @dev To indicate the entire call should revert, the function MUST revert.
    /// @param functionId An identifier that routes the call to different internal implementations, should there be more than one.
    /// @param sender The caller address.
    /// @param value The call value.
    /// @param data The calldata sent.
    function preRuntimeValidationHook(uint8 functionId, address sender, uint256 value, bytes calldata data) external;

    /// @notice Run the runtime validationFunction specified by the `functionId`.
    /// @dev To indicate the entire call should revert, the function MUST revert.
    /// @param functionId An identifier that routes the call to different internal implementations, should there be
    /// more than one.
    /// @param sender The caller address.
    /// @param value The call value.
    /// @param data The calldata sent.
    function runtimeValidationFunction(uint8 functionId, address sender, uint256 value, bytes calldata data)
        external;

    /// @notice Run the pre execution hook specified by the `functionId`.
    /// @dev To indicate the entire call should revert, the function MUST revert.
    /// @param functionId An identifier that routes the call to different internal implementations, should there be more than one.
    /// @param sender The caller address.
    /// @param value The call value.
    /// @param data The calldata sent.
    /// @return Context to pass to a post execution hook, if present. An empty bytes array MAY be returned.
    function preExecutionHook(uint8 functionId, address sender, uint256 value, bytes calldata data) external returns (bytes memory);

    /// @notice Run the post execution hook specified by the `functionId`.
    /// @dev To indicate the entire call should revert, the function MUST revert.
    /// @param functionId An identifier that routes the call to different internal implementations, should there be more than one.
    /// @param preExecHookData The context returned by its associated pre execution hook.
    function postExecutionHook(uint8 functionId, bytes calldata preExecHookData) external;

    /// @notice Describe the contents and intended configuration of the plugin.
    /// @dev This manifest MUST stay constant over time.
    /// @return A manifest describing the contents and intended configuration of the plugin.
    function pluginManifest() external pure returns (PluginManifest memory);

    /// @notice Describe the metadata of the plugin.
    /// @dev This metadata MUST stay constant over time.
    /// @return A metadata struct describing the plugin.
    function pluginMetadata() external pure returns (PluginMetadata memory);
}
```

### Plugin manifest

The plugin manifest is responsible for describing the execution functions, validation functions, and hooks that will be configured on the MSCA during installation, as well as the plugin's metadata, dependencies, and permissions.

```solidity
enum ManifestAssociatedFunctionType {
    // Function is not defined.
    NONE,
    // Function belongs to this plugin.
    SELF,
    // Function belongs to an external plugin provided as a dependency during plugin installation. Plugins MAY depend
    // on external validation functions. It MUST NOT depend on external hooks, or installation will fail.
    DEPENDENCY,
    // Resolves to a magic value to always bypass runtime validation for a given function.
    // This is only assignable on runtime validation functions. If it were to be used on a user op validationFunction,
    // it would risk burning gas from the account. When used as a hook in any hook location, it is equivalent to not
    // setting a hook and is therefore disallowed.
    RUNTIME_VALIDATION_ALWAYS_ALLOW,
    // Resolves to a magic value to always fail in a hook for a given function.
    // This is only assignable to pre hooks (pre validation and pre execution). It should not be used on
    // validation functions themselves, because this is equivalent to leaving the validation functions unset.
    // It should not be used in post-exec hooks, because if it is known to always revert, that should happen
    // as early as possible to save gas.
    PRE_HOOK_ALWAYS_DENY
}

/// @dev For functions of type `ManifestAssociatedFunctionType.DEPENDENCY`, the MSCA MUST find the plugin address
/// of the function at `dependencies[dependencyIndex]` during the call to `installPlugin(config)`.
struct ManifestFunction {
    ManifestAssociatedFunctionType functionType;
    uint8 functionId;
    uint256 dependencyIndex;
}

struct ManifestAssociatedFunction {
    bytes4 executionSelector;
    ManifestFunction associatedFunction;
}

struct ManifestExecutionHook {
    bytes4 selector;
    ManifestFunction preExecHook;
    ManifestFunction postExecHook;
}

struct ManifestExternalCallPermission {
    address externalAddress;
    bool permitAnySelector;
    bytes4[] selectors;
}

struct SelectorPermission {
    bytes4 functionSelector;
    string permissionDescription;
}

/// @dev A struct holding fields to describe the plugin in a purely view context. Intended for front end clients.
struct PluginMetadata {
    // A human-readable name of the plugin.
    string name;
    // The version of the plugin, following the semantic versioning scheme.
    string version;
    // The author field SHOULD be a username representing the identity of the user or organization
    // that created this plugin.
    string author;
    // String descriptions of the relative sensitivity of specific functions. The selectors MUST be selectors for
    // functions implemented by this plugin.
    SelectorPermission[] permissionDescriptors;
}

/// @dev A struct describing how the plugin should be installed on a modular account.
struct PluginManifest {
    // List of ERC-165 interface IDs to add to account to support introspection checks. This MUST NOT include
    // IPlugin's interface ID.
    bytes4[] interfaceIds;
    // If this plugin depends on other plugins' validation functions, the interface IDs of those plugins MUST be
    // provided here, with its position in the array matching the `dependencyIndex` members of `ManifestFunction`
    // structs used in the manifest.
    bytes4[] dependencyInterfaceIds;
    // Execution functions defined in this plugin to be installed on the MSCA.
    bytes4[] executionFunctions;
    // Plugin execution functions already installed on the MSCA that this plugin will be able to call.
    bytes4[] permittedExecutionSelectors;
    // Boolean to indicate whether the plugin can call any external address.
    bool permitAnyExternalAddress;
    // Boolean to indicate whether the plugin needs access to spend native tokens of the account. If false, the
    // plugin MUST still be able to spend up to the balance that it sends to the account in the same call.
    bool canSpendNativeToken;
    ManifestExternalCallPermission[] permittedExternalCalls;
    ManifestAssociatedFunction[] userOpValidationFunctions;
    ManifestAssociatedFunction[] runtimeValidationFunctions;
    ManifestAssociatedFunction[] preUserOpValidationHooks;
    ManifestAssociatedFunction[] preRuntimeValidationHooks;
    ManifestExecutionHook[] executionHooks;
}

```

### Expected behavior

#### Responsibilities of `StandardExecutor` and `PluginExecutor`

`StandardExecutor` functions are used for open-ended calls to external addresses.

`PluginExecutor` functions are specifically used by plugins to request the account to execute with account's context. Explicit permissions are required for plugins to use `PluginExecutor`.

The following behavior MUST be followed:

- `StandardExecutor` can NOT call plugin execution functions and/or `PluginExecutor`. This is guaranteed by checking whether the call's target implements the `IPlugin` interface via ERC-165 as required.
- `StandardExecutor` can NOT be called by plugin execution functions and/or `PluginExecutor`.
- Plugin execution functions MUST NOT request access to `StandardExecutor`, they MAY request access to `PluginExecutor`.

#### Calls to `installPlugin`

The function `installPlugin` accepts 4 parameters: the address of the plugin to install, the Keccak-256 hash of the plugin's manifest, ABI-encoded data to pass to the plugin's `onInstall` callback, and an array of function references that represent the plugin's install dependencies.

The function MUST retrieve the plugin's manifest by calling `pluginManifest()` using `staticcall`.

The function MUST perform the following preliminary checks:

- Revert if the plugin has already been installed on the modular account.
- Revert if the plugin does not implement ERC-165 or does not support the `IPlugin` interface.
- Revert if `manifestHash` does not match the computed Keccak-256 hash of the plugin's returned manifest. This prevents installation of plugins that attempt to install a different plugin configuration than the one that was approved by the client.
- Revert if any address in `dependencies` does not support the interface at its matching index in the manifest's `dependencyInterfaceIds`, or if the two array lengths do not match, or if any of the dependencies are not already installed on the modular account.

The function MUST record the manifest hash and dependencies that were used for the plugin's installation. Each dependency's record MUST also be updated to reflect that it has a new dependent. These records MUST be used to ensure calls to `uninstallPlugin` are comprehensive and undo all edited configuration state from installation. The mechanism by which these records are stored and validated is up to the implementation.

The function MUST store the plugin's permitted function selectors, permitted external calls, and whether it can spend the account's native tokens, to be able to validate calls to `executeFromPlugin` and `executeFromPluginExternal`.

The function MUST parse through the execution functions, validation functions, and hooks in the manifest and add them to the modular account after resolving each `ManifestFunction` type.

- Each execution function selector MUST be added as a valid execution function on the modular account. If the function selector has already been added or matches the selector of a native function, the function SHOULD revert.
- If a validation function is to be added to a selector that already has that type of validation function, the function SHOULD revert.

The function MAY store the interface IDs provided in the manifest's `interfaceIds` and update its `supportsInterface` behavior accordingly.

Next, the function MUST call the plugin's `onInstall` callback with the data provided in the `pluginInstallData` parameter. This serves to initialize the plugin state for the modular account. If `onInstall` reverts, the `installPlugin` function MUST revert.

Finally, the function MUST emit the event `PluginInstalled` with the plugin's address, the hash of its manifest, and the dependencies that were used.

> **⚠️ The ability to install and uninstall plugins is very powerful. The security of these functions determines the security of the account. It is critical for modular account implementers to make sure the implementation of the functions in `IPluginManager` have the proper security consideration and access control in place.**

#### Calls to `uninstallPlugin`

The function `uninstallPlugin` accepts 3 parameters: the address of the plugin to uninstall, a bytes field that may have custom requirements or uses by the implementing account, and ABI-encoded data to pass to the plugin's `onUninstall` callback.

The function MUST revert if the plugin is not installed on the modular account.

The function SHOULD perform the following checks:

- Revert if the hash of the manifest used at install time does not match the computed Keccak-256 hash of the plugin's current manifest. This prevents unclean removal of plugins that attempt to force a removal of a different plugin configuration than the one that was originally approved by the client for installation. To allow for removal of such plugins, the modular account MAY implement the capability for the manifest to be encoded in the config field as a parameter.
- Revert if there is at least 1 other installed plugin that depends on validation functions added by this plugin. Plugins used as dependencies SHOULD NOT be uninstalled while dependent plugins exist.

The function SHOULD update account storage to reflect the uninstall via inspection functions, such as those defined by `IAccountLoupe`. Each dependency's record SHOULD also be updated to reflect that it has no longer has this plugin as a dependent.

The function MUST remove records for the plugin's manifest hash, dependencies, permitted function selectors, permitted external calls, and whether it can spend the account's native tokens.

The function MUST parse through the execution functions, validation functions, and hooks in the manifest and remove them from the modular account after resolving each `ManifestFunction` type. If multiple plugins added the same hook, it MUST persist until the last plugin is uninstalled.

If the account stored the interface IDs provided in the manifest's `interfaceIds` during installation, it MUST remove them and update its `supportsInterface` behavior accordingly. If multiple plugins added the same interface ID, it MUST persist until the last plugin is uninstalled.

Next, the function MUST call the plugin's `onUninstall` callback with the data provided in the `pluginUninstallData` parameter. This serves to clear the plugin state for the modular account. If `onUninstall` reverts, execution SHOULD continue to allow the uninstall to complete.

Finally, the function MUST emit the event `PluginUninstalled` with the plugin's address and whether the `onUninstall` callback succeeded.

> **⚠️ Incorrectly uninstalled plugins can prevent uninstalls of their dependencies. Therefore, some form of validation that the uninstall step completely and correctly removes the plugin and its usage of dependencies is required.**

#### Calls to `validateUserOp`

When the function `validateUserOp` is called on modular account by the `EntryPoint`, it MUST find the user operation validation function associated to the function selector in the first four bytes of `userOp.callData`. If there is no function defined for the selector, or if `userOp.callData.length < 4`, then execution MUST revert.

If the function selector has associated pre user operation validation hooks, then those hooks MUST be run sequentially. If any revert, the outer call MUST revert. If any are set to `PRE_HOOK_ALWAYS_DENY`, the call MUST revert. If any return an `authorizer` value other than 0 or 1, execution MUST revert. If any return an `authorizer` value of 1, indicating an invalid signature, the returned validation data of the outer call MUST also be 1. If any return time-bounded validation by specifying either a `validUntil` or `validBefore` value, the resulting validation data MUST be the intersection of all time bounds provided.

Then, the modular account MUST execute the validation function with the user operation and its hash as parameters using the `call` opcode. The returned validation data from the user operation validation function MUST be updated, if necessary, by the return values of any pre user operation validation hooks, then returned by `validateUserOp`.

#### Calls to execution functions

When a function other than a native function is called on an modular account, it MUST find the plugin configuration for the corresponding selector added via plugin installation. If no corresponding plugin is found, the modular account MUST revert. Otherwise, the following steps MUST be performed.

Additionally, when the modular account natively implements functions in `IPluginManager` and `IStandardExecutor`, the same following steps MUST be performed for those functions. Other native functions MAY perform these steps.

The steps to perform are:

- If the call is not from the `EntryPoint`, then find an associated runtime validation function. If one does not exist, execution MUST revert. The modular account MUST execute all pre runtime validation hooks, then the runtime validation function, with the `call` opcode. All of these functions MUST receive the caller, value, and execution function's calldata as parameters. If any of these functions revert, execution MUST revert. If any pre runtime validation hooks are set to `PRE_HOOK_ALWAYS_DENY`, execution MUST revert. If the runtime validation function is set to `RUNTIME_VALIDATION_ALWAYS_ALLOW`, the validation function MUST be bypassed.
- If there are pre execution hooks defined for the execution function, execute those hooks with the caller, value, and execution function's calldata as parameters. If any of these hooks returns data, it MUST be preserved until the call to the post execution hook. The operation MUST be done with the `call` opcode. If there are duplicate pre execution hooks (i.e., hooks with identical `FunctionReference`s), run the hook only once. If any of these functions revert, execution MUST revert.
- Run the execution function.
- If any post execution hooks are defined, run the functions. If a pre execution hook returned data to the account, that data MUST be passed as a parameter to the associated post execution hook. The operation MUST be done with the `call` opcode. If there are duplicate post execution hooks, run them once for each unique associated pre execution hook. For post execution hooks without an associated pre execution hook, run the hook only once. If any of these functions revert, execution MUST revert.

The set of hooks run for a given execution function MUST be the hooks specified by account state at the start of the execution phase. This is relevant for functions like `installPlugin` and `uninstallPlugin`, which modify the account state, and possibly other execution or native functions as well.

#### Calls made from plugins

Plugins MAY interact with other plugins and external addresses through the modular account using the functions defined in the `IPluginExecutor` interface. These functions MAY be called without a defined validation function, but the modular account MUST enforce these checks and behaviors:

The `executeFromPlugin` function MUST allow plugins to call execution functions installed by plugins on the modular account. Hooks matching the function selector provided in `data` MUST be called. If the calling plugin's manifest did not include the provided function selector within `permittedExecutionSelectors` at the time of installation, execution MUST revert.

The `executeFromPluginExternal` function MUST allow plugins to call external addresses as specified by its parameters on behalf of the modular account. If the calling plugin's manifest did not explicitly allow the external call within `permittedExternalCalls` at the time of installation, execution MUST revert.

## Rationale

ERC-4337 compatible accounts must implement the `IAccount` interface, which consists of only one method that bundles validation with execution: `validateUserOp`. A primary design rationale for this proposal is to extend the possible functions for a smart contract account beyond this single method by unbundling these and other functions, while retaining the benefits of account abstraction.

The function routing pattern of ERC-2535 is the logical starting point for achieving this extension into multi-functional accounts. It also meets our other primary design rationale of generalizing execution calls across multiple implementing contracts. However, a strict diamond pattern is constrained by its inability to customize validation schemes for specific execution functions in the context of `validateUserOp`, and its requirement of `delegatecall`.

This proposal includes several interfaces that build on ERC-4337 and are inspired by ERC-2535. First, we standardize a set of modular functions that allow smart contract developers greater flexibility in bundling validation, execution, and hook logic. We also propose interfaces that take inspiration from the diamond standard and provide methods for querying execution functions, validation functions, and hooks on a modular account. The rest of the interfaces describe a plugin's methods for exposing its modular functions and desired configuration, and the modular account's methods for installing and removing plugins and allowing execution across plugins and external addresses.

## Backwards Compatibility

No backward compatibility issues found.

## Reference Implementation

See `https://github.com/erc6900/reference-implementation`

## Security Considerations

The modular smart contract accounts themselves are trusted components. Installed plugins are trusted to varying degrees, as plugins can interact with an arbitrarily large or small set of resources on an account. For example, a wide-reaching malicious plugin could add reverting hooks to native function selectors, bricking the account, or add execution functions that may drain the funds of the account. However, it is also possible to install a plugin with a very narrow domain, and depend on the correctness of the account behavior to enforce its limited access. Users should therefore be careful in what plugins to add to their account.

Users should perform careful due diligence before installing a plugin and should be mindful of the fact that plugins are potentially dangerous. The plugin's manifest can give users an understanding of the domain of the plugin, i.e., the requested permissions to install certain validation functions and/or hooks on certain execution selectors. Generally, plugins that include native function selectors in their domain, e.g., plugins that add a validation hook to the native `uninstallPlugin()` function, can introduce significantly more harm than plugins that simply add validation hooks to function selectors that the plugin itself is adding to the account.

Plugins can also add validation hooks to function selectors installed by other plugins. While usually, such a plugin would, e.g., add additional pre-validation hooks, it can also cause the previously installed plugin to be executed in an unintended context. For example, if a plugin were to only be intended to operate in the user operation context, its plugin manifest might only define user operation validation functions. However, another plugin might add a passing runtime validation function to that function selector, causing, for example, a session key plugin to suddenly be executed in a runtime validation context, circumventing all the parameter-validation that would have happened during user operation validation and granting unrestricted access to all session keys. Therefore, it is strongly recommended to always add reverting validation hooks to the context the plugin is not intended to be executed in. This recommendation may change in the next iteration of the standard.    

It is worth mentioning that execution hooks have no awareness of other execution hooks being performed in the same function selector execution setting. Since execution hooks can perform state changes, this reveals an important security consideration: An execution hook can only assure that at the time of its own execution, certain conditions are met, but this can not be generalized to the entire pre-execution context of potentially multiple pre-execution hooks. For example, a pre-execution hook cannot be assured that the storage it performed validation upon does not get further updated in subsequent pre-execution hooks. Even an associated post-execution hook potentially repeating the validation cannot assure that the storage remains unmodified because a prior post-execution hook may have reset the state. As long as the requirements checked by a plugin as part of an execution hook are only modifiable by the plugin itself, this can be considered safe.

## Copyright

Copyright and related rights waived via [CC0](/LICENSE.md).
