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

1. healthchecks.io 에서 체크를 만든다: 이름 `uptime-probe`, Period **10분**, Grace **60분**
2. 체크의 ping URL 을 저장소 Secret `HC_PING_URL` 로 등록한다
   ```bash
   gh secret set HC_PING_URL -R eonware-kor/uptime-probe
   ```
3. Actions 탭에서 `probe` 를 수동 실행(`workflow_dispatch`)해 healthchecks 에 첫 ping 이 들어오는지 확인한다

Secret 이 없으면 점검만 하고 결과 전송은 건너뜁니다(경고 표시).

## ⚠️ 알아둘 것

- **공개 저장소라 Actions 가 무료·무제한**입니다. 비공개로 바꾸면 조직의 월 2,000분 무료 한도를 CI 와 나눠 쓰게 됩니다.
- GitHub 은 **60일간 저장소 활동이 없으면 예약 실행을 자동으로 끕니다.** 꺼지면 healthchecks.io 가 ping 끊김으로 알려 주므로, 알림이 오면 Actions 탭에서 워크플로를 다시 켜면 됩니다.
- GitHub 예약 실행은 best-effort 라 부하가 높을 때 수십 분 늦거나 빠질 수 있습니다(2026-10-12 생성 직후 40분 넘게 미실행 실측). 정각을 피한 분(:04, :14 …)으로 예약하고, healthchecks Grace 를 넉넉히(60분) 둡니다. 그래도 오탐이 잦으면 전용 무료 감시 서비스(UptimeRobot·cron-job.org)로 옮깁니다.

관련: `eonware-kor/eonware-infra` 의 `docs/` · `scripts/node-hardening/`
