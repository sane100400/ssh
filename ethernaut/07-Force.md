# The Ethernaut : Force

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-Force)

## 핵심 개념

### `selfdestruct(address recipient)`

컨트랙트를 파괴하고, 남은 이더를 지정된 주소로 전송하는 함수이다.

## 문제 설명

빈 Force 컨트랙트가 주어진다. 목표는 이 컨트랙트에 이더를 보내는 것이다.

## 풀이

`selfdestruct()`는 일반적인 receive/fallback 함수 제한을 우회하여 강제로 이더를 전송할 수 있다.

1. Attack 컨트랙트를 배포하며 1 wei를 전송
2. `selfdestruct()`를 호출하여 자신을 파괴하고 Force 컨트랙트 주소로 이더 전송

## PoC

Attack 컨트랙트는:
- 생성자에서 대상 컨트랙트 주소를 받음
- 배포 시 1 wei를 수신
- `selfdestruct()`를 호출하여 축적된 자금을 Force 컨트랙트 주소로 전송

Foundry/Forge 프레임워크를 사용한 Script 컨트랙트로 Attack 컨트랙트를 1 wei와 함께 배포하고 destruct 함수를 실행하여 챌린지를 완료한다.
