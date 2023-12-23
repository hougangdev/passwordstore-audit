---
title: PasswordStore Protocol Audit Report
author: hougangdev
date: Dec 23, 2023
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries Protocol Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Cyfrin.io\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [hougangdev](https://cyfrin.io)
Auditor: hougangdev

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-01\] Storing the password on-chain makes it visible to anyone, and no longer private](#h-01-storing-the-password-on-chain-makes-it-visible-to-anyone-and-no-longer-private)
    - [\[H-02\] `PasswordStore::setPassword` has no access controls, meaning a non-owner could change the password](#h-02-passwordstoresetpassword-has-no-access-controls-meaning-a-non-owner-could-change-the-password)
  - [Medium](#medium)
  - [Low](#low)
  - [Informational](#informational)
    - [\[I-03\] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect](#i-03-the-passwordstoregetpassword-natspec-indicates-a-parameter-that-doesnt-exist-causing-the-natspec-to-be-incorrect)
  - [Gas](#gas)

# Protocol Summary

Protocol is a protocol to store and retrieve user's password on the blockchain. It is designed to be used by a single user.
# Disclaimer

The hougangdev team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

Commit Hash: 
```
2e8f81e263b3a9d18fab4fb5c46805ffc10a9990
```

## Scope 

```
./src/
#-- PasswordStore.sol
```

## Roles

- Owner: The user who can set the password and read the password.
- Outsides: No one else should be able to set or read the password.

# Executive Summary

Summary goes here

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 2                      |
| Medium   | 0                      |
| Low      | 0                      |
| Info     | 1                      |
| Total    | 3                      |


# Findings
## High

### [H-01] Storing the password on-chain makes it visible to anyone, and no longer private

**Description:**  All data stored on-chain is visile to anyone, and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be a private variable and only accessed through the `PasswordStore::getPassword` function, which is intended to be only called by the owner of the contract.

**Impact:** Anyone can read the private password, severly breaking the functionality of the protocol.

**Proof of Concept:** (Proof of Code)

The below test case shows how anyone can read the password directly from the blockchain.

Create locally running chan using make anvil
Deploy contract to chain
make deploy
run storage tool - cast storage `contract address 1` --rpc-url <rpc-url>

cast parse-bytes32-string `string`

**Recommended Mitigation:** 
Overall architecture of the contract should be rethought. One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the password. However, you'd also likely want to remove the view functions as you wouldn't want the user to accidentally send a transaction with the password that decrypts your password.

### [H-02] `PasswordStore::setPassword` has no access controls, meaning a non-owner could change the password

**Description:** 
The `PasswordStore::setPassword` function is set to be `external` function, however the natspec of the function and overall purpose of the smart contract is that `This function allows only the owner to set a new password.`

**Impact:** Anyone can set and change the password, severly breaking the contract functionality
```Javascript
    function setPassword(string memory newPassword) external {
        s_password = newPassword;
        emit SetNetPassword();
    }
```

**Proof of Concept:**
Add the following to the `Passwordstore.t.sol` test file.
<details>
<summary>Code</summary>

```Javascript
    function test_anyone_can_set_password(address randomAddress) public {
            vm.assume(randomAddress != owner);
            vm.prank(randomAddress);
            string memory expectedPassword = "myNewPassword";
            passwordStore.setPassword(expectedPassword);

            vm.prank(owner);
            string memory actualPassword = passwordStore.getPassword();
            assertEq(actualPassword, expectedPassword);
        }
```
</details>

**Recommended Mitigation:** Add an access control to the `setPassword` function.
```javascript
if(msg.sender != s_owner) revert("PasswordStore: Not owner");
```


## Medium
## Low 
## Informational

### [I-03] The `PasswordStore::getPassword` natspec indicates a parameter that doesn't exist, causing the natspec to be incorrect

**Description:** 
```javascript
/*
     * @notice This allows only the owner to retrieve the password.
     // @audit there's no newPassword param here.
     * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {
        if (msg.sender != s_owner) {
            revert PasswordStore__NotOwner();
        }
        return s_password;
    }
```


`passwordStore_getPassword` is the function signature, whereas the Natspec suggests the function should be `getPassword` with a string. The divergence results in incorrect NatSpec

**Impact:** The Natspec is incorrect

**Recommended Mitigation:** Remove the incorrect natspec line.

```diff
-   @param newPassword The new password to set.
```


## Gas 