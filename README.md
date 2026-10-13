# uptime-probe

Eonware 공개 서비스를 **클러스터와 AWS 밖(GitHub Actions)** 에서 10분마다 점검해 healthchecks.io 로 보고하는 외부 감시기입니다.

## 왜 있나

2026-10-09~12 운영 장애 때 클러스터 안의 모니터링(Prometheus)이 장애 노드와 함께 멈췄습니다. 외부 dead-man's switch(healthchecks.io)는 금요일 15:48 KST 에 감지해 이메일을 보냈지만 주말 동안 아무도 보지 못했습니다. 이 저장소는 **서비스 자체가 응답하는지**를 바깥에서 직접 확인해, 같은 알림 경로(healthchecks.io → 휴대폰 알림)로 "어느 서비스가 죽었는지"를 알려 줍니다.

## 동작

| 결과 | healthchecks.io 로 보내는 것 |
|---|---|
| 전부 정상 | `HC_PING_URL` (성공 ping, 본문에 전체 결과) |
| 하나라도 실패 (3회 재시도 후) | `HC_PING_URL/fail` (본문에 실패 목록) → 즉시 DOWN 알림 |
| 워크플로 자체가 안 돎 | ping 이 끊겨 grace 이후 DOWN 알림 |

점검 대상과 기대 코드는 `.github/workflows/probe.yml` 의 `targets` 에 있습니다. 전부 공개 URL 이며, 이 저장소에는 비밀 정보가 없습니다.

## 설정

실행 트리거는 **cron-job.org**(무료 외부 cron)가 맡습니다. GitHub 예약 실행은 best-effort 라 신규 저장소에서 11시간에 1회만 실행된 실측(2026-10-12~13)이 있어 보조로만 둡니다.

1. healthchecks.io 체크 `uptime-probe`: Period **10분**, Grace **60분**. ping URL 은 Secret `HC_PING_URL`
2. GitHub **fine-grained PAT** — 이 저장소의 Actions 실행 권한만
   - Resource owner `eonware-kor`, Repository access: **Only select repositories → `uptime-probe`**
   - Repository permissions: **Actions = Read and write** (Metadata = Read 는 자동)
3. cron-job.org 작업
   - URL `https://api.github.com/repos/eonware-kor/uptime-probe/actions/workflows/probe.yml/dispatches`, Method **POST**
   - Headers: `Authorization: Bearer <PAT>`, `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`, `Content-Type: application/json`
   - Body `{"ref":"main"}`, 실행 주기 **10분**, 정상 응답 **204**
   - 실패 알림(이메일) 켜기 — PAT 만료·폐기 시 여기서 먼저 알 수 있다

PAT 가 만료되면 호출이 401 로 실패하고, cron-job.org 실패 메일과 healthchecks DOWN 메일이 함께 옵니다. 새 PAT 를 발급해 cron-job.org 헤더만 바꾸면 됩니다.

## ⚠️ 알아둘 것

- **공개 저장소라 Actions 가 무료·무제한**입니다. 비공개로 바꾸면 조직의 월 2,000분 무료 한도를 CI 와 나눠 쓰게 됩니다.
- GitHub 은 **60일간 저장소 활동이 없으면 예약 실행을 자동으로 끕니다.** 꺼지면 healthchecks.io 가 ping 끊김으로 알려 주므로, 알림이 오면 Actions 탭에서 워크플로를 다시 켜면 됩니다.
- PAT 는 이 저장소 하나의 Actions 권한만 갖도록 제한했습니다. 노출돼도 할 수 있는 일은 이 공개 워크플로를 실행하거나 취소하는 정도입니다.

관련: `eonware-kor/eonware-infra` 의 `docs/` · `scripts/node-hardening/`
