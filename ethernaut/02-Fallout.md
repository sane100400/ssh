# The Ethernaut : Fallout

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-Fallout)

## 핵심 개념

### Solidity Constructor의 변천사

과거 Solidity에서는 컨트랙트 이름과 동일한 함수가 생성자 역할을 했다:

```solidity
contract C {
    function C() public { ... }  // 구버전 생성자 방식
}
```

오타와 이름 불일치로 인한 버그가 발생하면서, 현대 Solidity에서는 명시적 문법을 강제한다:

```solidity
contract C {
    constructor() public { ... }
}
```

하지만 레거시 코드, CTF 챌린지, 교육 자료에서는 여전히 구버전 패턴이 존재하여 유사한 취약점이 발생할 수 있다.

## 취약점

Fallout 컨트랙트에 오타가 있다: 함수 이름이 `Fal1out()` (숫자 1)으로 되어 있어 컨트랙트 이름 `Fallout`과 매칭되지 않는다. 이로 인해 의도된 생성자가 실행되지 않고, 일반 public 함수로 호출 가능하게 남는다.

## 풀이

`Fal1out()`을 직접 호출하면 인증 검사 없이 소유권이 호출자에게 이전된다.
