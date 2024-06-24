# Solidity CTF Notes: PasswordStore Contract

## 1. Find the security misconfigurations 

- CTF URL: [CodeHawks Contest](https://www.codehawks.com/contests/clnuo221v0001l50aomgo4nyn)
- GitHub Repository: [Cyfrin/2023-10-PasswordStore](https://github.com/Cyfrin/2023-10-PasswordStore/tree/main/src)

## PasswordStore

A smart contract application for storing a password. Users should be able to store a password and then retrieve it later. Others should not be able to access the password.

### Contract Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

/*
 * @author not-so-secure-dev
 * @title PasswordStore
 * @notice This contract allows you to store a private password that others won't be able to see. 
 * You can update your password at any time.
 */
contract PasswordStore {
    error PasswordStore__NotOwner();

    address private s_owner;
    string private s_password;

    event SetNetPassword();

    constructor() {
        s_owner = msg.sender;
    }

    /*
     * @notice This function allows only the owner to set a new password.
     * @param newPassword The new password to set.
     */
    function setPassword(string memory newPassword) external {
        s_password = newPassword;
        emit SetNetPassword();
    }

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
}

### Findings

1. **Improper Access Control for `setPassword` Function:**
   - According to the description, this contract is intended for a single user. However, the `setPassword` function is marked as `external`, allowing anyone to access and change the password. The function should verify that the caller is the owner before allowing the password change.

2. **Insecure Storage of Sensitive Information:**
   - Secret or sensitive information should never be stored directly in smart contracts, as they can be accessed by anyone. Despite `s_password` being declared as `private`, it is still publicly accessible through various means. Sensitive data should be handled off-chain or through more secure methods.
