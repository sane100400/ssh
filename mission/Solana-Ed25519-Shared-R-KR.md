# Solana 핫월렛 개인키 탈취 취약점 (Ed25519 Shared-R) 분석

> [Blog 원문](https://velog.io/@sane100400/Solana-%ED%95%AB%EC%9B%94%EB%A0%9B-%EA%B0%9C%EC%9D%B8%ED%82%A4-%ED%83%88%EC%B7%A8-%EC%B7%A8%EC%95%BD%EC%A0%90-Ed25519-Shared-R-%EB%B6%84%EC%84%9D)

**Tags:** ED25519, Nonce reuse, Shared-R, blockchain, solana

---

## 1. 사건 개요

2025년, 한 암호화폐 거래소의 Solana 핫월렛에서 수천억 원 규모의 자산이 유출되었다. Jacquot와 Donnet의 연구(FC 2026)에 따르면, Bitcoin, Ethereum을 포함한 6개 블록체인에서 nonce 재사용으로 인해 3,620개의 개인키가 노출되었으며, 약 1억 1백만 유로의 자산이 위험에 처해 있다.

---

## 2. 배경 지식

### 2.1 타원곡선 전자서명

블록체인은 지갑 소유권을 증명하기 위해 타원곡선 암호(ECC)를 사용한다. 고정된 기준점 G에 비밀 스칼라 a를 곱하면 공개키 A가 생성된다:

**A = a · G**

A에서 a를 역산하는 것은 계산적으로 불가능하다. 이 단방향 성질이 전자서명 보안의 기반이다.

전자서명의 목표는 비밀키 a를 공개하지 않고 a를 알고 있음을 증명하는 것이다. 이를 위해 서명마다 새로운 일회용 nonce r을 생성하여 서명 방정식에 포함시킨다.

#### 서명 과정

1. 일회용 nonce r 생성
2. R = r · G 계산 (nonce에 대응하는 타원곡선 점)
3. 메시지 M의 해시, 비밀키 a, nonce r을 사용하여 S 계산: S = r + H(R, A, M) · a
4. (R, S) 쌍을 서명으로 공개

#### 검증 과정

서명 시 S = r + H(R, A, M) · a이므로, 양변에 G를 곱하면:

**S · G = r · G + H(R, A, M) · a · G = R + H(R, A, M) · A**

검증자는 S · G가 R + H(R, A, M) · A와 같은지 확인한다. 우변은 서명 구성요소 R, 공개키 A, 메시지 M만으로 계산 가능하므로, 누구나 a를 알지 못한 채 서명을 검증할 수 있다.

#### ECDSA (Bitcoin, Ethereum)

```
r ← 외부 난수 생성기 (예: OS /dev/urandom)
R = r * G
S = (H(M) + R.x * a) / r (mod n)
```

nonce r을 외부에서 매번 새로 가져온다. 난수 생성기 장애로 nonce가 재사용되면 두 S 값의 차이로 비밀키를 복구할 수 있다는 단점이 있다.

#### Ed25519 (Solana, SSH, TLS)

```
nonce_prefix ← SHA-512(seed)의 하위 32바이트
r = SHA-512(nonce_prefix || M)
R = r * G
S = r + H(R, A, M) * a (mod L)
```

Ed25519는 비밀키에서 파생된 nonce_prefix와 메시지 M을 연결하여 해시한 결과로 nonce를 생성한다. nonce_prefix는 비밀키에서 한 번 파생되는 고정값이며, 다른 메시지는 다른 해시 출력을 산출하여 고유한 nonce를 생성한다. 외부 난수 생성기 의존성을 제거하지만 **nonce_prefix가 서명자마다 고유해야 한다**. 두 서명자가 같은 nonce_prefix를 공유하면 같은 메시지 서명 시 동일한 r 값이 생성된다.

### 2.2 Ed25519 비밀키 구조

Ed25519에서 단일 비밀키(32바이트 seed)를 SHA-512로 해시하면 64바이트가 생성되며, 두 반으로 분할된다:

```
SHA-512(seed) → 64 bytes

앞 32바이트 → scalar (a): 서명 연산에 사용되는 실제 비밀 스칼라
뒤 32바이트 → nonce prefix: 서명별 nonce 파생에 사용되는 시드 값
```

scalar a는 서명 수학 연산에 직접 사용되는 비밀 숫자이고, nonce prefix는 ECDSA에서 OS 난수 생성기가 하는 역할을 수행한다. **nonce를 비밀키 자체에서 파생하는 것**이 Ed25519의 핵심 설계 원칙이다.

### 2.3 서명 (R, S)의 생성 과정

Ed25519가 메시지 M(트랜잭션 데이터 등)에 서명할 때, (R, S) 쌍이 생성된다:

```
r = SHA-512(nonce_prefix || M)  ... nonce 생성
R = r * G                        ... nonce에 대응하는 타원곡선 점
S = r + H(R, A, M) * a          ... 비밀키 a를 포함한 연산
```

| 기호 | 설명 | 공개 여부 |
|------|------|----------|
| r | Nonce; nonce_prefix와 메시지 M의 연결 해시 | 아니오 (중간값) |
| R | r에 대응하는 타원곡선 점 | **예** (서명에 포함) |
| a | 비밀키 스칼라 | 아니오 (보호됨) |
| S | 비밀키 a를 포함한 연산 결과 | **예** (서명에 포함) |
| H(R, A, M) | R, 공개키 A, 메시지 M의 해시 | 누구나 계산 가능 |
| G | 모든 참여자가 아는 고정 기준점 | 예 (상수) |

핵심 성질: **nonce_prefix가 서명자마다 다르면, 같은 메시지에 대해서도 r이 다르고, 따라서 R (= r · G)도 다르다**. 서명자별 nonce_prefix 고유성이 Ed25519 보안을 유지한다.

> Solana에서 서명은 R (32바이트)과 S (32바이트)를 연결한 64바이트 값으로 공개된다.

### 2.4 Shared-R이 위험한 이유

두 서명자가 **같은 nonce_prefix**를 가지면 어떻게 될까?

두 서명자가 같은 트랜잭션(같은 메시지 M)에 서명하는 경우:

```
signer A: r_A = SHA-512(prefix_A || M)    R_A = r_A * G
signer B: r_B = SHA-512(prefix_B || M)    R_B = r_B * G
```

nonce_prefix가 서명자마다 다르면, r_A ≠ r_B이고 따라서 R_A ≠ R_B이다.

그러나 prefix_A = prefix_B이면, 같은 메시지 M에 대해 r_A = r_B이고, 따라서 R_A = R_B이다.

두 다른 서명자가 같은 R을 생성하는 이 현상을 **Shared-R**이라 한다. R과 S 모두 블록체인에 공개 기록되므로, Shared-R은 두 S 값의 차이로 비밀키를 복구할 수 있는 연립방정식을 만든다.

### 2.5 Shared-R의 근본 원인: 키 파생 구현 차이

Solana 지갑은 일반적으로 단일 마스터 시드에서 여러 자식키를 파생하는 **HD (Hierarchical Deterministic) 키 파생**을 사용하며, 단일 시드 백업으로 모든 자식키를 복원할 수 있다. Solana는 이 과정에 SLIP-0010 표준을 사용한다.

SLIP-0010 사양은 Ed25519 자식키 파생을 다음과 같이 정의한다:

> "the returned child key k_i is I_L"
> : SLIP-0010, Private parent key → private child key

여기서 k_i는 파생된 자식키이고 I_L은 64바이트 SHA-512 출력의 왼쪽 32바이트이다. 사양은 32바이트 자식키 생성 방법만 정의한다.

그러나 Ed25519 서명에는 scalar (32바이트)와 nonce_prefix (32바이트), 총 64바이트가 필요하다. 32바이트 자식키를 SHA-512에 다시 통과시켜 64바이트를 생성해야 한다. 이 단계를 **re-expansion**이라 한다.

올바른 구현에서는 re-expansion을 수행하여 각 자식이 **고유한 nonce_prefix**를 얻는다:

```
올바름: child_key → SHA-512(child_key) → [new scalar | new prefix] (자식마다 고유)
```

그러나 re-expansion을 생략하고 마스터 시드에서 파생된 nonce prefix를 모든 자식에 재사용하면:

```
잘못됨: master_seed → SHA-512 → [master_scalar | master_prefix]
        child_0: [child_scalar_0 | master_prefix]    ← 같은 prefix
        child_1: [child_scalar_1 | master_prefix]    ← 같은 prefix
```

**모든 자식이 같은 prefix를 공유하며**, 같은 메시지에 서명하면 동일한 R 값이 생성된다.

re-expansion은 명시적으로 강제된 사양 단계가 아니므로, 키 파생을 최적화하거나 단순화하는 커스텀 구현에서 이를 생략하여 Shared-R 취약점을 도입할 수 있다.

### 2.6 관련 취약점

| 출처 | 설명 |
|------|------|
| CVE-2022-50237 / RUSTSEC-2022-0093 | ed25519-dalek 라이브러리에서 scalar/nonce_prefix 분리를 허용, 두 서명에서 추출 가능 |
| orlp/ed25519 Issue #3 | ed25519_add_scalar 함수가 nonce prefix 업데이트에 실패, R 재사용 발생 |
| MystenLabs/ed25519-unsafe-libs | 동일 취약점 클래스의 영향을 받는 40+ Ed25519 라이브러리 카탈로그 |

---

## 3. 개인키 추출 방법

Shared-R이 나타나는 두 트랜잭션이 주어지면, 다음 절차로 개인키를 추출할 수 있다.

### 3.1 서명 방정식

서명 공식 S = r + H(R, A, M) · a를 두 서명자(signer 0, signer 1)와 두 트랜잭션(TX1, TX2)에 적용:

```
TX1: s_01 = r_1 + h_01 · a_0 (mod L)  ...(1)
     s_11 = r_1 + h_11 · a_1 (mod L)  ...(2)
TX2: s_02 = r_2 + h_02 · a_0 (mod L)  ...(3)
     s_12 = r_2 + h_12 · a_1 (mod L)  ...(4)
```

- s_ij: 서명의 S 구성요소 (온체인에서 공개 조회 가능)
- r_j: nonce (Shared-R로 인해 같은 TX 내 두 서명자가 동일)
- h_ij = SHA-512(R || A_i || M_j) mod L: challenge hash (온체인 데이터로 계산 가능)
- a_i: 비밀키 스칼라 (추출 대상)
- L: Ed25519 그룹의 위수 (고정 상수)

### 3.2 Nonce 소거

(1)-(2)와 (3)-(4)를 빼면 nonce r이 소거된다:

```
d_1 = s_01 - s_11 = h_01 · a_0 - h_11 · a_1 (mod L)
d_2 = s_02 - s_12 = h_02 · a_0 - h_12 · a_1 (mod L)
```

미지수 2개(a_0, a_1)에 방정식 2개이므로 풀 수 있다.

### 3.3 크래머 공식 적용

행렬 형태로 표현:

```
| h_01  -h_11 | | a_0 |   | d_1 |
| h_02  -h_12 | | a_1 | = | d_2 |

det = h_01 · (-h_12) - (-h_11) · h_02 (mod L)

a_0 = [d_1 · (-h_12) - (-h_11) · d_2] / det (mod L)
a_1 = [h_01 · d_2 - d_1 · h_02] / det (mod L)
```

**모든 입력 값(s, h, d)은 온체인에서 공개적으로 조회 가능한 데이터로 계산할 수 있다.**

비밀키 스칼라 a_0과 a_1은 완전히 결정된다.

### 3.4 검증

추출된 스칼라를 사용해 공개키를 재계산하고 원본과 비교하여 검증한다:

**a · G = A (Ed25519 기준점 곱셈)**

일치하면 개인키 추출 성공이 확인된다.

---

## 4. Devnet 재현

실제 악용 가능성을 확인하기 위해 Solana devnet에서 동일 조건을 재현했다.

### 4.1 재현 절차

1. 랜덤 마스터 시드에서 SLIP-0010을 사용하여 두 자식키 파생
2. 자식 스칼라는 정상 파생하되, 두 자식 모두 마스터의 nonce prefix를 재사용 (취약 구현 재현)
3. 두 서명자가 같은 트랜잭션 메시지에 서명하고 R이 동일한지 확인
4. 두 트랜잭션을 devnet에 제출
5. 온체인에서 공개적으로 조회 가능한 데이터만 사용하여 크래머 공식으로 개인키 추출
6. 추출된 개인키로 공개키를 재계산하고 원본과 일치하는지 검증

### 4.2 실행 결과

```
========================================================================
  Shared-R PoC — Ed25519 nonce reuse 취약점 재현 (DEVNET)
========================================================================

[Phase 1] 취약 키 쌍 생성 (SLIP-0010 + master prefix 재사용)
  Signer 0: GxKGiE5sGmgAs3QrLFEkrmwYtMV8crxqq5Zo2ZaqAe4J
  Signer 1: EXrP8x4UWSvWZJ9GjhumL21bodu2qz69D6ZC6nr34bQ1
  Shared prefix (hex): 51338c9a28c92c134555dbfc913a634a...
  키 생성 검증: OK

[Phase 3] 트랜잭션 1 구성 및 제출
  Sig 0 R: affb089cfd802e72b1d4d04373a8cefd...
  Sig 1 R: affb089cfd802e72b1d4d04373a8cefd...
  Shared-R: YES
  TX1 제출 완료

[Phase 4] 트랜잭션 2 구성 및 제출
  Sig 0 R: bee13c44cbb4b9f044cc990080ce375b...
  Sig 1 R: bee13c44cbb4b9f044cc990080ce375b...
  Shared-R: YES
  TX2 제출 완료

[Phase 5] 온체인 데이터 조회
  TX1 Shared-R 온체인 확인: YES
  TX2 Shared-R 온체인 확인: YES

[Phase 6] 개인키 추출 (크래머 공식 — 온체인 데이터만 사용)
  입력: TX1/TX2의 서명(R, S), 메시지, 공개키
  추출된 Scalar 0 (hex LE): a0cf256a02afea8cce5ad165f20502ab...
  추출된 Scalar 1 (hex LE): c0003bf2194765209bf14844564c25b1...
```

### 4.3 검증 요약

| 검증 항목 | 결과 |
|-----------|------|
| TX1 내 Signer 0의 R과 Signer 1의 R 일치 (Shared-R)? | YES |
| TX2 내 Signer 0의 R과 Signer 1의 R 일치 (Shared-R)? | YES |
| TX1의 Signer 0 R과 TX2의 Signer 0 R 일치? | NO (다른 메시지) |
| 추출된 scalar * G가 Signer 0의 공개키와 일치? | YES |
| 추출된 scalar * G가 Signer 1의 공개키와 일치? | YES |
| TX1 nonce 복구 시 두 서명자에 대해 같은 r 계산? | YES |
| TX2 nonce 복구 시 두 서명자에 대해 같은 r 계산? | YES |
| 추출된 scalar이 원본 scalar과 직접 비교 시 일치? | YES |

### 4.4 온체인 증거

Shared-R 현상은 다음 Solana devnet 트랜잭션에서 직접 확인할 수 있다.

**서명자 주소:**

| 역할 | 주소 |
|------|------|
| Signer 0 | GxKGiE5sGmgAs3QrLFEkrmwYtMV8crxqq5Zo2ZaqAe4J |
| Signer 1 | EXrP8x4UWSvWZJ9GjhumL21bodu2qz69D6ZC6nr34bQ1 |

**트랜잭션:**
- TX1: https://solscan.io/tx/4X4x9xfAF63MwdbxeLrGHUawL5J6BSTMGvz48TqHsykB3k1UWvCAcJi9v4tZEJa2d6vb5KRZmEvBet9WtwHNXLnn?cluster=devnet
- TX2: https://solscan.io/tx/4pM2kVVYjLNko3EC5p5gaPQ2xN4fCoqEgigsvYLLCj7Ceb5ijbCaaEt6tnvfHc6UxMviz1QnbTmU56f7s5N5Vo8s?cluster=devnet

---

## 5. 참고 자료

| 자료 | 링크 |
|------|------|
| RFC 8032 (Ed25519 사양) | https://datatracker.ietf.org/doc/html/rfc8032#section-5.1 |
| SLIP-0010 (HD 키 파생 표준) | https://github.com/satoshilabs/slips/blob/master/slip-0010.md |
| CVE-2022-50237 (NVD) | https://nvd.nist.gov/vuln/detail/CVE-2022-50237 |
| RUSTSEC-2022-0093 (Rust 보안 권고) | https://rustsec.org/advisories/RUSTSEC-2022-0093 |
| ed25519-unsafe-libs (취약 라이브러리 카탈로그) | https://github.com/MystenLabs/ed25519-unsafe-libs |
| orlp/ed25519 Issue #3 | https://github.com/orlp/ed25519/issues/3 |
| Double Public Key Signing Attack 논문 | https://arxiv.org/abs/2308.1500 |
| Jacquot and Donnet, FC 2026 | https://fc26.ifca.ai/preproceedings/55.pdf |

---

> **Disclaimer:** 이 글은 특정 거래소나 사건에 대한 원인 규명을 의도하지 않습니다. 공개된 암호학적 서명 구조에서 발생할 수 있는 nonce 재사용 취약점 클래스에 대한 기술적 설명과 방어 및 검증 관점에서의 함의를 정리한 것입니다.
