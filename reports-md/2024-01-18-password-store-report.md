---
title: Protocol Audit Report
author: NarwhalGuard
date: January 18, 2024
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
\vspace{2cm}
{\Huge\bfseries Protocol Audit Report\par}
\vspace{1cm}
{\Large Version 1.0\par}
\vspace{2cm}
{\Large\itshape NarwhalGuard\par}
\vfill
{\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [NarwhalGuard]()
Lead Auditors:

- Ammar Robbani

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
    - [\[H-1\] The confidentiality of the password stored on-chain as private variable is compromised, rendering the protocol's password storage mechanism ineffective.](#h-1-the-confidentiality-of-the-password-stored-on-chain-as-private-variable-is-compromised-rendering-the-protocols-password-storage-mechanism-ineffective)
    - [\[H-2\] The `PasswordStore::setPassword` function is susceptible to misuse, allowing any user to alter the password, contrary to the intended functionality specified in the natspec documentation.](#h-2-the-passwordstoresetpassword-function-is-susceptible-to-misuse-allowing-any-user-to-alter-the-password-contrary-to-the-intended-functionality-specified-in-the-natspec-documentation)
  - [Low](#low)
    - [\[L-1\] The `PasswordStore` contract exhibits an initialization timeframe vulnerability, which means that there is a period between contract deployment and the explicit call to `PasswordStore::setPassword` during which the password remains in its default state.](#l-1-the-passwordstore-contract-exhibits-an-initialization-timeframe-vulnerability-which-means-that-there-is-a-period-between-contract-deployment-and-the-explicit-call-to-passwordstoresetpassword-during-which-the-password-remains-in-its-default-state)
  - [Informational](#informational)
    - [\[I-1\] The `PasswordStore::getPassword` function lacks the `newPassword` parameter, even though it is mentioned in the natspec documentation.](#i-1-the-passwordstoregetpassword-function-lacks-the-newpassword-parameter-even-though-it-is-mentioned-in-the-natspec-documentation)

# Protocol Summary

A smart contract applicatoin for storing a password. Users should be able to store a password and then retrieve it later. Others should not be able to access the password.

# Disclaimer

The NarwhalGuard team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details

**Repository for the audit:**

https://github.com/Cyfrin/3-passwordstore-audit

**Commit Hash:**

```
2e8f81e263b3a9d18fab4fb5c46805ffc10a9990
```

## Scope

```
./src/
---PasswordStore.sol
```

## Roles

- Owner: The user who can set the password and read the password.

For this contract, only the owner should be able to interact with the contract.

# Executive Summary

## Issues found

| Severity          | Number of issues found |
| ----------------- | ---------------------- |
| High              | 2                      |
| Medium            | 0                      |
| Low               | 1                      |
| Info              | 1                      |
| Gas Optimizations | 0                      |
| Total             | 0                      |

# Findings

## High

### [H-1] The confidentiality of the password stored on-chain as private variable is compromised, rendering the protocol's password storage mechanism ineffective.

**Description:** Despite being declared as private, the `PasswordStore::s_password` variable is observable by external entities. The term private merely signifies inaccessibility by other smart contracts.

**Impact:** Any actor can access and view the content of `PasswordStore::s_password`.

**Proof of Concept:**

1. Launch the local chain:

```
make anvil
```

2. In a separate terminal, deploy the `PasswordStore` contract locally:

```bash
make deploy
// == Return ==
//0: contract PasswordStore 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

3. Obtain the contract address and access storage with index `1` (the storage index corresponds to the variable order in the contract):

```
cast storage 0x5FbDB2315678afecb367f032d93F642f64180aa3 1
// 0x6d7950617373776f726400000000000000000000000000000000000000000014
```

4. Convert the result from hex to string:

```
cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
//myPassword
```

**Recommended Mitigation:**
Employ encrypted passwords for the contract. Pre-encrypt the password off-chain before setting the encrypted password in the contract. Avoid including encryption and/or decryption methods within the contract to prevent public visibility by others.

### [H-2] The `PasswordStore::setPassword` function is susceptible to misuse, allowing any user to alter the password, contrary to the intended functionality specified in the natspec documentation.

**Description:** The `PasswordStore::setPassword` function is susceptible to misuse, allowing any user to alter the password, contrary to the intended functionality specified in the natspec documentation.

<details><summary>Function: </summary>

```js
    function setPassword(string memory newPassword) external {
@>      // there is no access control here
        s_password = newPassword;
        emit SetNetPassword();
    }
```

</details>
<br/>

**Impact:** All users possess the ability to repeatedly modify the password without adhering to the intended ownership restriction.

**Proof of Concept:**

```js
    function test_non_owner_can_set_password() public {
        // Owner set the password
        vm.startPrank(owner);
        string memory prevPassword = "myPrevPassword";
        passwordStore.setPassword(prevPassword);
        vm.stopPrank();

        // The other set up the new password
        address nonOwner = makeAddr("user");
        vm.startPrank(nonOwner);
        string memory newPassword = "myNewPassword";
        passwordStore.setPassword(newPassword);
        vm.stopPrank();

        // The owner checks if the current password is the same as the previous one
        vm.startPrank(owner);
        string memory newActualPassword = passwordStore.getPassword();
        assertNotEq(newActualPassword, prevPassword);
        assertEq(newActualPassword, newPassword);
        vm.stopPrank();
    }
```

**Recommended Mitigation:**
To address this issue, two effective mitigation methods are proposed:

1.  Access control from OpenZeppelin library:\
    Integrate the Ownable contract from the OpenZeppelin library into the `PasswordStore` contract. This approach establishes a robust access control mechanism, allowing precise determination of users authorized to execute the `PasswordStore::setPassword` function. For further details on access control, refer to the [OpenZeppelin documentation](https://docs.openzeppelin.com/contracts/5.x/access-control).

    <details><summary>Example code</summary>

    ```js
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.8.18;

    import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

    contract PasswordStore is Ownable {
        constructor() Ownable(initialOwner) {
            initialOwner = msg.sender;
        }

        function setPassword() public onlyOwner {
            // only the owner can call specialThing()!
            ...
        }
    }
    ```

    </details><br/

2.  Use validation for the owner:\
    Implement a validation step within the `PasswordStore::setPassword` function to verify whether the caller is the designated owner of the contract. By incorporating this validation, the function will only permit password changes initiated by the owner, aligning with the ownership-based access control stipulated in the natspec documentation.

```diff
    function setPassword(string memory newPassword) external {
+       if (msg.sender != s_owner) {
+           revert PasswordStore__NotOwner();
+       }
        s_password = newPassword;
        emit SetNetPassword();
    }
```

## Low

### [L-1] The `PasswordStore` contract exhibits an initialization timeframe vulnerability, which means that there is a period between contract deployment and the explicit call to `PasswordStore::setPassword` during which the password remains in its default state.

**Description:** The contract does not set the password during its construction (in the constructor). As a result, when the contract is initially deployed, the password remains uninitialized, taking on the default value for a string, which is an empty string.

During this initialization timeframe, the contract's password is effectively empty and can be considered a security gap.

**Impact:** The impact of this vulnerability is that during the initialization timeframe, the contract's password is left empty, potentially exposing the contract to unauthorized access or unintended behavior.

**Recommended Mitigation:** To mitigate the initialization timeframe vulnerability, consider setting a password value during the contract's deployment (in the constructor). This initial value can be passed in the constructor parameters.

## Informational

### [I-1] The `PasswordStore::getPassword` function lacks the `newPassword` parameter, even though it is mentioned in the natspec documentation.

**Description:** The `PasswordStore::getPassword` function lacks `newPassword` as an input parameter, contrary to the indication in the natspec documentation.

<details><summary>Function: </summary>

```js
    /*
     * @notice This allows only the owner to retrieve the password.
     * @param newPassword The new password to set.
     */
    function getPassword() external view returns (string memory) {
        if (msg.sender != s_owner) {
            revert PasswordStore__NotOwner();
        }
        return s_password;
    }
```

</details>
<br/>

**Recommended Mitigation:**
Revise the natspec documentation to accurately reflect the absence of the `newPassword` parameter in the `PasswordStore::getPassword` function. This adjustment will prevent confusion among developers and ensure accurate understanding of the function's signature.
