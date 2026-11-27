---
id: "001"
change: ./change.md
status: approved
---

# 001 릴리스 이동 태그 구현 계획

## 접근 방식

공개 reusable workflow가 호출 저장소의 Release 이벤트를 검증하고, 해당 저장소에 설치된 GitHub App의 토큰으로 이동 태그를 갱신합니다. 같은 저장소의 갱신 job을 직렬화하고, 각 job이 실행될 때 발행된 Release 목록을 다시 조회합니다. 이벤트의 버전은 갱신할 major와 minor 범위를 정하는 데 사용합니다.

워크플로 매뉴얼에 호출 예시, App 준비 절차, 실패 시 재실행 방법을 문서화합니다. 저장소 보호 설정은 요구사항이 아니므로, 이동 태그를 보호하려는 소비자를 위한 권장 ruleset 안내로만 다룹니다.

## 설계 항목

### D-1. 진입점과 이벤트 검증 (FR-4, FR-6, FR-7, FR-9)

- `.github/workflows/move-tags.yml`에 `workflow_call` 진입점을 둡니다. 공용 인터페이스로 대상 저장소나 태그를 입력받지 않고 호출 이벤트의 저장소 및 Release를 사용합니다.
- 인증 전에 이벤트 이름과 작업 유형을 확인합니다. `release.published` 이외의 호출은 warning annotation을 남기고 정상 종료합니다. 발행 이벤트라도 사전 릴리스이거나 태그가 정확한 `vX.Y.Z` 형식이 아니면 갱신을 건너뜁니다.
- 이 저장소의 `.github/workflows/release.yml`은 `release: types: [published]`에서 본체를 로컬 경로로 호출합니다. 첫 릴리스에도 이동 태그 없이 호출할 수 있습니다.
- 호출 예시와 권한, 오류 처리 방법은 `.github/workflows/README.md`에 작성하고 루트 README 카탈로그에서 링크합니다.

### D-2. 호출 저장소의 App 인증 (FR-5, FR-7, FR-10)

- 호출자가 App 클라이언트 ID와 개인 키를 전달합니다. 초안 인터페이스는 문자열 입력 `app-client-id`와 명시적 시크릿 `app-private-key`입니다. 이벤트 검증을 먼저 수행할 수 있게 선언 단계에서 필수로 지정하지 않고, 갱신 대상 이벤트에서 둘 중 하나가 없으면 실패를 보고합니다.
- GitHub 공식 App 토큰 액션으로 호출 저장소 한 곳에 한정한 설치 토큰을 발급합니다. 토큰의 권한은 `Contents: write`로 지정하고, 기본 `GITHUB_TOKEN` 권한은 비웁니다. 토큰은 갱신 job 안에서만 사용합니다.
- `.github/workflows/README.md`에 권장 ruleset을 안내합니다. 이동 태그의 생성·갱신을 제한하는 ruleset을 쓰는 소비자는 전용 App을 그 ruleset의 예외로 등록해야 합니다. `main`과 `vX.Y.Z` 태그를 보호하는 ruleset에는 App을 예외로 두지 않도록 안내합니다.

### D-3. Release 조회와 태그 갱신 (FR-1, FR-2, FR-3, FR-4, FR-5, FR-8, NFR-1)

- GitHub REST의 Release 목록을 모든 페이지에 걸쳐 조회합니다. 발행된 안정 Release의 태그를 형식 검증한 뒤, major·minor·patch를 각각 숫자로 비교합니다. API 응답 순서와 GitHub의 Latest 표시에는 의존하지 않습니다.
- 이벤트 태그가 속한 major 범위와 minor 범위에서 각각 최고 버전을 선택합니다. 선택한 고정 태그의 Git 참조를 조회하고, 주석 태그는 최종 커밋까지 해석합니다. 커밋을 확인할 수 없으면 쓰기 전에 실패합니다.
- 이동 태그를 조회해 대상과 같으면 쓰기를 생략하고, 참조가 없다는 사실을 확인했으면 생성하며, 기존 참조가 다르면 갱신합니다. 권한, 연결, 조회 오류를 태그 부재로 처리하지 않습니다.
- 갱신 실패 시 태그 이름과 실패한 작업을 보고합니다. 이미 적용한 갱신은 유지하고, 재실행에서는 현재 Release 목록으로 대상을 다시 계산합니다.

### D-4. 동시 실행 제어 (NFR-5, FR-8)

- 태그 쓰기 job에 호출 저장소 단위의 전용 concurrency 그룹을 지정합니다. 버전별로 그룹을 나누면 서로 다른 minor가 같은 major 태그를 동시에 쓸 수 있으므로 같은 저장소에서는 하나의 그룹을 공유합니다.
- `queue: max`로 대기 실행을 유지하고 진행 중인 실행은 취소하지 않습니다. Release 목록 조회부터 마지막 쓰기까지 그룹 안에서 처리합니다. 순서는 Release 발행 순서와 달라도 됩니다.

## 기술 선택과 이유

| 결정 | 선택 | 기각한 대안 | 이유 |
| --- | --- | --- | --- |
| 버전 비교 | REST Release 목록을 순회해 버전 범위별로 비교 | 이벤트 태그 또는 Latest Release만 사용 | 미발행 태그를 제외하고 이전 minor 패치와 재실행에서도 올바른 대상을 결정할 수 있습니다. |
| Git 참조 조작 | 공식 `actions/github-script`에서 REST 호출 | 저장소 체크아웃 후 Git push | 필요한 태그만 조회하고 오류 유형을 구분하기 쉽습니다. |
| 동시 실행 | 저장소 단위 직렬화와 다중 대기 | 기본 단일 대기 또는 버전별 그룹 | 기본 대기 설정에서는 처리해야 할 minor가 누락될 수 있고, 버전별 그룹은 공유 major 태그의 경쟁을 막을 수 없습니다. |
| 갱신 실패 재현 | 검증 저장소에 시험용 ruleset을 잠시 적용 | 로컬 API 대역 | 실제 GitHub 응답을 근거로 검수할 수 있습니다. 인라인 스크립트를 YAML에서 추출하는 시험 장치와 로컬 Node.js 환경이 필요하지 않습니다. |

## 검증 방법

| 수용 기준 | 검증 방법 |
| --- | --- |
| AC-1 | 검증 저장소에서 경량 고정 태그와 주석 고정 태그를 각각 발행하고 두 이동 태그의 최종 커밋을 비교합니다. |
| AC-2 | `v1.9.0`, `v1.10.0`을 순서대로 발행한 뒤 major와 기존 minor 참조를 비교합니다. |
| AC-3 | `v1.3.0` 이후 이전 minor의 패치를 발행하고 더 오래된 실행도 재실행하여 두 이동 태그를 확인합니다. |
| AC-4 | 사전 릴리스, 사전 릴리스로 표시된 정식 형식의 태그, 단축 태그 이벤트를 주고 쓰기가 발생하지 않는지 확인합니다. |
| AC-5 | 검증 저장소에 기존 이동 태그의 갱신을 막는 시험용 ruleset을 App 예외 없이 걸고 실행하여, 실행 실패와 로그에 생성 요청이 없는지 확인합니다. 검수가 끝나면 ruleset을 지웁니다. |
| AC-6 | 같은 Release 실행을 두 번 수행하고 각 실행 전후에 고정 태그와 이동 태그의 참조를 비교합니다. |
| AC-10 | 동일 버전의 태그 푸시, 초안 저장, Release 발행 순서대로 실행 기록을 확인합니다. |
| AC-11 | 별도 호출 저장소에 App을 설치해 공개 워크플로의 고정 ref로 호출하고, 호출 저장소와 제공 저장소의 태그 참조를 비교합니다. |
| AC-12 | 더 높은 버전의 태그를 미발행 상태로 두고 낮은 버전을 먼저 발행한 뒤, 높은 버전을 발행하여 각 시점의 참조를 비교합니다. |
| AC-13 | 시험용 ruleset으로 `v1` 갱신만 막아 한쪽 이동 태그 갱신을 실패시키고 기존 성공 결과를 확인한 뒤, ruleset을 지우고 재실행하여 두 참조를 비교합니다. |
| AC-14 | `workflow_dispatch`에서 호출하여 warning annotation, 정상 종료, 참조 불변을 확인합니다. |
| AC-15 | 같은 major 내 두 minor의 Release 실행 세 개를 겹치게 하고, 실행 순서를 바꿔 각 minor와 major의 최종 참조를 비교합니다. |
| AC-16 | App 자격 증명을 전달하지 않는 호출 워크플로로 Release를 발행하고, 실행 실패와 이동 태그 참조 불변을 확인합니다. |

## 위험 요소

- **대기열 한도:** GitHub의 다중 대기는 최대 100개입니다. 대기열이 가득 차면 취소된 실행을 기록에서 확인하고 재실행해야 합니다. 큐 취소가 발생한 상태에서는 모든 발행이 처리되었다고 볼 수 없습니다.
- **토큰 권한:** `Contents: write`는 태그 전용 권한이 아니어서 브랜치에도 쓸 수 있습니다. 토큰의 범위를 대상 저장소 한 곳으로 한정하고, 매뉴얼에서 `main`과 `vX.Y.Z` 보호 규칙에 App을 예외로 두지 않도록 안내합니다. App 비밀값을 출력하거나 다른 job으로 전달하지 않습니다.
- **플랫폼 구문:** `queue: max`의 GitHub.com 지원은 공식 문서에서 확인했습니다. 로컬에는 actionlint가 설치되어 있지 않으므로 Docker 이미지로 실행하며, 이 키를 인식하는지는 미확인입니다. 검증 저장소에서도 워크플로가 로드되고 세 실행이 대기하는지 확인합니다.

## 참고 자료

- [GitHub reusable workflow와 컨텍스트](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)
- [GitHub App 설치 토큰 액션](https://github.com/actions/create-github-app-token)
- [Release 목록 API](https://docs.github.com/en/rest/releases/releases#list-releases)
- [Git 참조 API](https://docs.github.com/en/rest/git/refs)
- [GitHub Actions 동시 실행 설정](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-workflow-concurrency)
- [GitHub ruleset 설정](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository)
