# The Ethernaut Writeups

[The Ethernaut](https://ethernaut.openzeppelin.com/)은 OpenZeppelin에서 만든 Solidity 스마트 컨트랙트 보안 워게임입니다.

각 문제의 취약점을 분석하고 Foundry 기반 PoC를 작성하여 풀이합니다.

## Challenges

| # | Challenge | Core Vulnerability |
|---|-----------|-------------------|
| 1 | [Fallback](01-Fallback.md) | `receive()` 함수를 통한 소유권 탈취 |
| 2 | [Fallout](02-Fallout.md) | Constructor 함수명 오타로 인한 접근 제어 우회 |
| 4 | [Telephone](04-Telephone.md) | `tx.origin` vs `msg.sender` 차이를 이용한 우회 |
| 5 | [Token](05-Token.md) | Integer Underflow를 통한 잔액 조작 |
| 7 | [Force](07-Force.md) | `selfdestruct()`를 통한 강제 이더 전송 |
| 8 | [Vault](08-Vault.md) | `private` 변수의 온체인 스토리지 직접 읽기 |
| 9 | [King](09-King.md) | `revert()`를 이용한 DoS 공격 |
| 10 | [Re-entrancy](10-Re-entrancy.md) | 재진입 공격으로 컨트랙트 자금 탈취 |
| 11 | [Elevator](11-Elevator.md) | 인터페이스 구현을 통한 반환값 조작 |
| 12 | [Privacy](12-Privacy.md) | Storage Slot 직접 접근으로 비공개 데이터 읽기 |
| 13 | [Gatekeeper One](13-GatekeeperOne.md) | `gasleft()` 조작 및 바이트 캐스팅 |

## Tools

- **Foundry (Forge)** - 스마트 컨트랙트 개발 및 테스트 프레임워크
- **Solidity ^0.8.0** - 스마트 컨트랙트 언어
