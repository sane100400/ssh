# The Ethernaut : Vault

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-Vault)

## 핵심 개념

### Storage Layout

Solidity에서 `private`, `public` 같은 가시성 수정자는 **언어 레벨**의 규칙일 뿐, 실제로 블록체인 상의 데이터를 숨기지 않는다.

모든 컨트랙트 변수는 이더리움 상태 트리에 "storage slot"으로 저장된다. RPC 엔드포인트, 컨트랙트 주소, 슬롯 번호만 있으면 `eth_getStorageAt`, Foundry의 `vm.load`, `cast storage` 등으로 누구나 직접 읽을 수 있다.

## 취약점

`Vault` 컨트랙트는 비밀번호를 `private bytes32` 변수로 저장한다. `unlock()` 함수에 올바른 비밀번호를 전달해야 잠금을 해제할 수 있지만, 비밀번호를 스토리지 슬롯 1에서 직접 읽을 수 있다.

## 풀이

1. Foundry의 `vm.load()`로 대상 컨트랙트의 스토리지 슬롯 1에 접근
2. 비밀번호 값 추출
3. 추출한 비밀번호로 `unlock()` 호출
4. `locked`를 false로 설정

온체인에 저장된 민감한 데이터는 암호화나 가시성 수정자로 보호할 수 없다.
