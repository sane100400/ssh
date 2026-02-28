# The Ethernaut : Privacy

> [Blog 원문](https://velog.io/@sane100400/The-Ethernaut-Privacy)

## 핵심 개념

### Storage Layout

Solidity에서 가시성 수정자 `private`, `public`은 **언어 레벨 규칙**일 뿐, 실제 데이터 암호화 메커니즘이 아니다.

**Storage 기본 규칙:**
- 각 스토리지 슬롯은 32 바이트
- 작은 데이터 타입 (uint8, bool)은 같은 슬롯에 패킹
- 큰 타입 (uint256, bytes32)은 전체 슬롯 차지
- 고정 크기 배열은 연속 슬롯 차지

## 스토리지 슬롯 분석

컨트랙트 변수의 슬롯 배치:
- **Slot 0**: `bool locked`
- **Slot 1**: `uint256 ID`
- **Slot 2**: 세 개의 uint8/uint16 값 (패킹)
- **Slot 3-5**: `bytes32[3]` 배열

## 풀이

목표: `data[2]` (슬롯 5)를 읽어 `bytes16`으로 변환한 후 `unlock()`에 전달

1. Foundry의 `vm.load()`로 슬롯 5 직접 읽기
2. 필요한 bytes16 값 추출
3. 추출한 값으로 `unlock()` 함수 호출
4. `locked`를 false로 설정

생성자 인자를 알 필요 없이 스토리지에서 직접 비밀번호를 읽을 수 있다.
