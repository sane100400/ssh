# The Ethernaut : Re-entrancy

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-Re-entrancy)

## 핵심 개념

### Re-entrancy (재진입 공격)

**"컨트랙트가 자금을 보내는 도중, 내부 정리를 완료하기 전에 다시 호출되는"** 취약점이다.

## 취약점 분석

문제 컨트랙트 `Reentrance`의 흐름:

1. **`donate()`** - 사용자 잔액을 추적하는 mapping에 입금
2. **`balanceOf()`** - 사용자 잔액을 반환하는 view 함수
3. **`withdraw()`** - 기록된 잔액을 줄이기 **전에** 외부로 ETH 전송

핵심 결함: `withdraw()`에서 `msg.sender.call{value: _amount}("")` 외부 호출이 잔액 감소 `balances[msg.sender] -= _amount` **이전에** 실행된다. 이 순서가 취약점 창을 만든다.

## 풀이

1. 공격 컨트랙트가 대상에 자금을 입금하여 잔액 기록 생성
2. 즉시 해당 금액으로 withdraw 호출
3. ETH를 수신하면 공격자의 `receive()` 함수가 트리거되어 `withdraw()`를 다시 호출
4. 대상 컨트랙트에 자금이 남아있는 동안 이 사이클이 반복되어 완전히 drain

## 핵심 교훈

취약점은 **외부 호출 이후에 상태 변경이 일어나는 것**에서 비롯된다. **"checks-effects-interactions"** 패턴을 따르거나 reentrancy guard를 사용하면 이 공격 벡터를 방지할 수 있다.
