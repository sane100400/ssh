# SSH - Security Study Hub

> Blockchain & Smart Contract Security 동아리 활동 기록

[![Velog](https://img.shields.io/badge/Blog-velog-20C997?style=for-the-badge&logo=velog&logoColor=white)](https://velog.io/@sane100400/posts)

---

## About

블록체인 보안, 스마트 컨트랙트 취약점 분석, CTF/Wargame 풀이 등을 기록하는 공간입니다.

## Contents

### The Ethernaut

Solidity 스마트 컨트랙트 보안 워게임 [The Ethernaut](https://ethernaut.openzeppelin.com/) 문제 풀이 모음입니다.

| # | Challenge | Keyword | Writeup |
|---|-----------|---------|---------|
| 1 | Fallback | `receive()` / `fallback()` | [풀이](ethernaut/01-Fallback.md) |
| 2 | Fallout | Constructor 취약점 | [풀이](ethernaut/02-Fallout.md) |
| 4 | Telephone | `tx.origin` vs `msg.sender` | [풀이](ethernaut/04-Telephone.md) |
| 5 | Token | Integer Underflow | [풀이](ethernaut/05-Token.md) |
| 7 | Force | `selfdestruct()` | [풀이](ethernaut/07-Force.md) |
| 8 | Vault | Storage Layout | [풀이](ethernaut/08-Vault.md) |
| 9 | King | DoS with Revert | [풀이](ethernaut/09-King.md) |
| 10 | Re-entrancy | Reentrancy Attack | [풀이](ethernaut/10-Re-entrancy.md) |
| 11 | Elevator | Interface Abuse | [풀이](ethernaut/11-Elevator.md) |
| 12 | Privacy | Storage Slot Reading | [풀이](ethernaut/12-Privacy.md) |
| 13 | Gatekeeper One | `gasleft()` / Byte Casting | [풀이](ethernaut/13-GatekeeperOne.md) |

### Solana Security Research

| Title | Link |
|-------|------|
| Solana 핫월렛 개인키 탈취 취약점 (Ed25519 Shared-R) 분석 | [Blog](https://velog.io/@sane100400/Solana-%ED%95%AB%EC%9B%94%EB%A0%9B-%EA%B0%9C%EC%9D%B8%ED%82%A4-%ED%83%88%EC%B7%A8-%EC%B7%A8%EC%95%BD%EC%A0%90-Ed25519-Shared-R-%EB%B6%84%EC%84%9D) |

### Other CTF / Challenge

| Title | Link |
|-------|------|
| RED RACCOON 실전 Threat Intelligence 추적 챌린지 | [Blog](https://velog.io/@sane100400/RED-RACCOON-%EC%8B%A4%EC%A0%84-Threat-Intelligence-%EC%B6%94%EC%A0%81-%EC%B1%8C%EB%A6%B0%EC%A7%80) |

---

## Tech Stack

![Solidity](https://img.shields.io/badge/Solidity-363636?style=flat-square&logo=solidity&logoColor=white)
![Foundry](https://img.shields.io/badge/Foundry-3C3C3D?style=flat-square&logo=ethereum&logoColor=white)
![Ethereum](https://img.shields.io/badge/Ethereum-3C3C3D?style=flat-square&logo=ethereum&logoColor=white)
![Solana](https://img.shields.io/badge/Solana-9945FF?style=flat-square&logo=solana&logoColor=white)
