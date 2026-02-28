# The Ethernaut : Token

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-Token)

## 핵심 개념

### Integer Underflow

Solidity 0.8.0 이전 버전에서는 unsigned integer(uint256) 연산 시 underflow가 발생할 수 있다.

## 취약점

컨트랙트의 `transfer` 함수에 결함이 있는 검증 로직이 존재한다:

```solidity
require(balances[msg.sender] - _value >= 0);
```

이 검사는 의미가 없다. unsigned integer에서 음수 결과가 나오면 underflow가 발생하여 극도로 큰 양수가 되기 때문이다.

## 풀이

1. 목표: 개인 잔액 증가
2. 현재 잔액을 초과하는 값으로 `transfer()` 호출
3. 뺄셈에서 underflow 발생 → 거대한 양수 값으로 변환
4. 공격자가 부풀려진 잔액을 획득

## PoC

Foundry 기반 익스플로잇으로, 취약한 토큰의 transfer 함수를 초과 금액(10101010101)으로 호출하여 underflow 취약점을 트리거하고 잔액을 대폭 증가시킨다.
