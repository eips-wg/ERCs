# EIP 6982 implementation

As a reference implementation of EIP-6982 we use the Nduja Labs ERC721Lockable contract.

To run the tests, run the following commands:

```bash
npm i -g pnpm
pnpm i
pnpm test
```

## Directory Listing

- [`contracts/ERC721Lockable.sol`](./contracts/ERC721Lockable.sol)
- [`contracts/IERC6982.sol`](./contracts/IERC6982.sol)
- [`contracts/IERC721Lockable.sol`](./contracts/IERC721Lockable.sol)
- [`contracts/mocks/ERC721LockableMock.sol`](./contracts/mocks/ERC721LockableMock.sol)
- [`contracts/mocks/MyLocker.sol`](./contracts/mocks/MyLocker.sol)
- [`test/Lockable.test.js`](./test/Lockable.test.js)
- [`test/helpers.js`](./test/helpers.js)
