# 3주차 — 무중단 배포 & 엔터프라이즈 기반 정비 (Chapter 5·6)

> 교재: 조훈, 『AI 시대에 개발자가 알아야 할 인프라 구성 배포 with 클로드 코드』 (길벗)
> 기간: 2026.07.13 ~ 07.19

## 들어가며

3·4장에서 "자동으로 배포되고, 보이는" 시스템을 만들었다면, 5·6장은 **"끊기지 않고, 안전하게"**로 넘어간다. 5장은 트래픽 관리(Gateway API)와 Blue/Green 전환, 6장은 캐시(Valkey)·시크릿 관리·Canary 배포다.

내 환경(AWS EKS 변형) 기준으로는 이번 주가 **치환 난이도가 가장 높다.** 지금까지의 도구들은 클라우드 중립이라 거의 그대로였는데, Gateway API와 Secret 관리는 책이 GKE 관리형 기능에 기대는 구간이라 구현체 선택부터 다시 해야 한다. 지난주처럼 개념 정리 + EKS 치환 설계 중심으로 정리했다.

## Chapter 5 — 무중단 배포

### 롤링 업데이트로도 왜 끊기는가

`maxUnavailable: 0`으로 Pod 교체 자체는 매끄럽지만, 트래픽 관점에선 빈틈이 두 군데 있다.

- **나가는 쪽**: 종료되는 Pod에 남아 있는 요청 — SIGTERM 처리와 `preStop`으로 LB에서 빠질 시간을 벌어야 한다 (graceful shutdown).
- **들어오는 쪽**: 헬스체크가 새 백엔드를 인지하기까지의 지연 — 체크 간격을 줄여도 "최소 1회 통과"라는 구조적 지연은 못 없앤다.

가시다님이 채널에 공유해주신 U+ 글("종료했는데 왜 502·504가 날까")이 딱 이 주제다. 그래서 5장은 배포 단위가 아니라 **트래픽 단위의 제어**(Gateway API)와 원자적 전환(Blue/Green)으로 간다.

### Gateway API — EKS 치환 최대 지점

책(GKE)은 옵션 하나로 관리형 GatewayClass가 생기지만, EKS엔 기본 구현체가 없어 선택부터 해야 한다:

| 선택지 | 판단 |
|---|---|
| **Envoy Gateway** | ✅ 채택 — 표준 구현이라 책의 Gateway/HTTPRoute 매니페스트가 거의 그대로 이식 |
| AWS Gateway API Controller (VPC Lattice) | 시맨틱이 책과 달라 학습 목적에 안 맞음 |
| AWS LB Controller + Ingress | 실무 표준이지만 이번 목표는 Gateway API 학습 |

GKE 전용 요소(HealthCheckPolicy CRD, proxy-only 서브넷)는 EKS에선 해당 없다 — Envoy는 readinessProbe 기반이라 별도 헬스체크 정책이 필요 없다.

덧붙여 가시다님 실측 공유가 인상적이었다 — 공개된 Gateway 주소로 **30분 만에 설정값을 노리는 스캔**이 들어왔다는 것. 실습용 임시 노출도 조심할 일이다.

### Blue/Green (Argo Rollouts)

Deployment를 `kind: Rollout` + `strategy.blueGreen`(active/preview 서비스, autoPromotion)으로 전환한다. preview로 새 버전을 확인한 뒤 트래픽을 한 번에 넘기는 구조. 주의점 둘 — **deployment.yaml은 반드시 삭제**(공존 시 이중 배포), **3장 CI의 태그 교체 대상도 rollout.yaml로 변경**(두 곳). 다만 B/G도 전환 "순간"의 유실까지 없애진 못한다 — 6장 Canary로 이어지는 이유다.

### 마무리 — ADR

`docs/architecture-decisions.md`에 그간의 도구 선택을 ADR 형식(결정 + 이유 3~4개)으로 누적한다. memory가 "나의 작업 맥락"이라면 ADR은 "팀의 결정 기록" — 나중에 "Ingress로 바꿀까?" 물으면 AI가 memory가 아니라 **Git의 ADR을 읽고 근거로 답하는** 구조가 좋다.

## Chapter 6 — Enterprise를 위한 기반 정비

### 들어가기 전: 리소스 계산

예산표 기준 이번 장이 최위험 구간이다. Secret CSI의 DaemonSet이 노드마다 붙으면서 소형 노드 2대의 CPU가 꽉 찬다. 진입 전 관측 스택 requests·replicas 축소가 전제고, B/G는 전환 순간 active+preview가 동시에 떠서 Pod 수가 2배라는 것도 계산에 넣어야 한다.

### Valkey — 상태를 Pod 밖으로

인메모리 카운터인 `/id`를 Valkey의 `INCR`로 옮긴다. 주의점: bitnami 차트 기본 preset이 커서 `resourcesPreset=none` 필요, 앱에 **연결 재시도 로직 필수**(Pod 기동 순서에 따라 CrashLoopBackOff), 의존성이 생기니 Dockerfile에 `go.sum` COPY 추가.

### Secret 관리 — EKS 치환 2번째 지점

| 책 (GCP) | 내 환경 (AWS/EKS) |
|---|---|
| Google Secret Manager | AWS Secrets Manager |
| GKE managed CSI (provider `gke`) | 오픈소스 Secrets Store CSI Driver + AWS provider |
| Workload Identity | **IRSA** (IAM Roles for Service Accounts) |
| 파일 마운트 패턴 | 동일 |

이 지점은 EKS가 오히려 유리하다 — GKE managed CSI는 리소스를 줄일 수 없는데(GKE가 복원) EKS 쪽은 오픈소스 설치라 직접 조정된다. 함정 하나: 시크릿 저장 시 개행이 들어가면(`echo` 대신 `echo -n`) `WRONGPASS`가 난다.

### Canary — 5장의 빈틈을 메우는 답

트래픽을 20% → 50% → 80%로 점진 노출해 B/G의 "전환 순간" 문제를 구조적으로 줄인다. Gateway API로 weight를 조정하려면 Argo Rollouts의 TrafficRouting 플러그인이 필요한데, Envoy Gateway 조합에서도 성립하는지가 내 환경의 검증 포인트다.

전환 절차는 순서 의존적이다: 새 전략을 **git push 먼저** → 그 다음 기존 Rollout 삭제. 반대로 하면 auto-sync가 옛 B/G 버전을 즉시 복원한다. preview 서비스는 canaryService로 재사용되니 삭제 금지.

### 마무리 — claude-context/

`claude-context/architecture.md`에 현재 아키텍처 스냅샷을 정리한다. 지식이 3층으로 나뉜다 — **CLAUDE.md**(항상 로드되는 규칙) / **claude-context/**(지금 어떻게 동작하는가) / **ADR**(왜 이렇게 결정했는가). 사람 온보딩 문서로도 그대로 쓰일 구조다.

## 소감

- 이번 주는 주말 출근이 있어서 제대로 된 작업이 힘들었어서 많은 부분을 AI 에 의존했는데, 너무 잘 해준다... 이미 코딩도 AI 가 다 해주는데 조만간 인프라도 사람이 설 자리가 없어지는 게 아닐까?
- 5·6장은 지난주 소감의 연장선에 있다. selfHeal이 "클러스터를 손대지 마라"를 구조로 만들었듯, Blue/Green과 Canary는 "배포 순간을 조심해라"를 구조로 만든다. **사람의 조심을 시스템의 보장으로 옮기는 작업**이 이 책의 일관된 방향이라는 게 점점 선명해진다.
- 클라우드 종속의 경계가 확 넓어졌다. 지난주엔 CI 인증부 정도였는데, 이번 주는 트래픽 인입(Gateway 구현체)과 시크릿(CSI provider + 인증)이 통째로 치환 대상이다. 그래도 대체재가 전부 표준 인터페이스(Gateway API, CSI Driver) 위라서 치환이 "설계 다시"가 아니라 "구현체 교체" 수준에 머문다. 표준을 따르는 도구를 고르는 것의 가치를 체감한다.
- 지난주 가드레일 우려의 각론 하나가 잡혔다. Canary 전환의 "git push 먼저, 그 다음 삭제" 같은 **순서 의존 절차**야말로 LLM이 어기기 쉬운 유형이다 — 순서가 바뀌어도 겉보기엔 비슷하게 진행되다가, auto-sync가 옛 버전을 되살리는 식으로 조용히 어긋난다. 이런 절차는 자연어 지침에 맡기지 말고 스크립트나 CI 단계로 순서 자체를 고정하는 게 맞다고 본다. "프롬프트 바깥의 강제 장치"를 고민만 해봤다고 했는데, 첫 적용 대상이 정해진 셈이다.
