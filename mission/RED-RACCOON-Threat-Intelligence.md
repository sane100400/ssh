# RED RACCOON 실전 Threat Intelligence 추적 챌린지

> [Blog 원문](https://velog.io/@sane100400/RED-RACCOON-%EC%8B%A4%EC%A0%84-Threat-Intelligence-%EC%B6%94%EC%A0%81-%EC%B1%8C%EB%A6%B0%EC%A7%80)

**Tags:** CTF, Threat Intelligence, Phishing, OSINT

---

## 문제 개요

RaccoonCoin 직원들을 대상으로 한 피싱 캠페인이 발생했다. CEO Tony Raccoon을 사칭하여 복지 포털 변경 안내 메일을 보냈으며, 실제 CEO는 해당 메일을 보낸 적이 없다고 부인했다. 목표는 공격 인프라를 end-to-end로 추적하는 것이다.

---

## 이메일 체인 분석

조사 결과 `mail.tutanota.de` (IP: 203.0.113.45)에서 발송된 위조 메시지로, `tony.raccoon@racooncoin.site`에서 보낸 것으로 위장되어 있다. (오타 주목: "racooncoin" vs 정상 "raccooncoin")

**침해 지표 (IoC):**
- SPF/DKIM/DMARC 인증 실패
- 타이포스쿼팅 도메인 등록
- 가짜 로그인 페이지를 통한 크리덴셜 수집

---

## 피싱 기법

HTML 콘텐츠가 사용자를 `vpn.racooncoin.site`로 유도하며 다음과 같은 텍스트를 표시한다: "로그인하여 변경 사항을 확인해 주세요"

**Typosquatting:** 유사한 도메인을 등록하여 사용자의 오타를 악용, 피싱 및 사기 작전을 수행하는 기법.

**Credential Harvesting:** 가짜 인증 페이지를 통해 사용자 자격 증명을 수집하는 기법.

---

## 리다이렉트 서버 분석

JavaScript 코드에서 피싱 흐름이 드러난다:

```javascript
window.location.href = 'http://140.238.194.224';
```

크리덴셜 캡처 후 사용자는 IP `140.238.194.224`를 거쳐 정상 사이트로 리다이렉트된다.

---

## 인프라 추적

**nmap 스캔**으로 리다이렉터에서 3개의 열린 포트를 식별한다. GitHub 리포지토리에서 유출된 크리덴셜(사용자명: spark)을 사용하여 SSH 접근:

- `plan.txt` - 공격 전략 문서
- `netstat` - 포트 2222에서 C2 피벗 확인
- 숨겨진 `.bashrc` 파일 - Tor onion 도메인 및 C2 서버 IP `158.180.6.169` 포함

---

## 딥웹 조사

`.onion` 주소에 접근하면 탈취된 데이터 아티팩트가 있는 관리자 패널이 나타나며, `stolen_payment_records.csv`에 침해 증거가 포함되어 있다.

---

## 악성코드 분석

마지막 단계로 `ransom_loader_v2.exe`를 다운로드하고 SHA256 해시를 계산한다:

- **PowerShell:** `Get-Filehash -Algorithm SHA256`
- **Linux:** `sha256sum` 명령어

**주의:** 플래그 제출 시 해시 값은 소문자여야 한다.

---

## 핵심 교훈

이 챌린지는 현실적인 공격 체인을 시연한다:

**이메일 스푸핑 → 크리덴셜 수집 → 인프라 정찰 → C2 피벗 → 악성코드 배포**

다중 경로 풀이 접근 방식은 단일 답변보다 위협 분석 방법론에 보상을 준다.
