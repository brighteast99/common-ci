# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 목적

`common-ci`는 도메인과 무관하게 재사용하는 GitHub Actions 공통 부품(composite action, reusable workflow, pre-commit hook)을 제공합니다. `terraform-ci` 같은 도메인별 CI 레포도 이 레포의 부품을 기반으로 동작합니다.

소비 레포가 어떤 방식으로 참조할지는 정할 수 없으므로, 릴리스에 포함된 모든 내용은 공개 계약으로 취급합니다. 입력과 출력, 동작, 경로를 호환되지 않게 바꾸는 작업은 breaking change이므로 major 버전을 올려야 합니다.

## 개발 방식

reusable workflow, composite action, pre-commit hook을 추가하거나 동작을 변경할 때는 `/sdd` 스킬을 기반으로 개발합니다. 문서 정리나 레포 설정처럼 부품의 동작이 바뀌지 않는 작업에는 사용하지 않습니다.

개발 과정에서 정의하는 스펙은 `/sdd`의 규칙을 따릅니다. 각 `README.md`는 부품을 사용하는 소비자를 위한 매뉴얼이므로, 요구사항이나 수용 기준, 설계 결정을 적지 않습니다.

## 디렉터리 구성

| 부품 | 위치 | 매뉴얼 |
| --- | --- | --- |
| 공개 composite action (step 단위) | `actions/<name>/action.yml` | `actions/<name>/README.md` |
| 내부 composite action | `actions/internal/<name>/action.yml` | 작성하지 않습니다. |
| reusable workflow (job 단위) | `.github/workflows/<name>.yml` (`on: workflow_call`) | `.github/workflows/README.md`에 모든 워크플로의 매뉴얼을 모읍니다. |
| pre-commit hook | 루트의 `.pre-commit-hooks.yaml` | |

- **내부 composite action:** 외부에 공개할 목적이 아니라, 이 레포의 여러 reusable workflow에 반복되는 로직을 한곳에 모으기 위한 액션입니다. 외부 사용을 의도하지 않더라도 릴리스에 포함되므로 공개 계약 원칙은 그대로 적용됩니다. 특히 워크플로가 내부 액션을 이동 태그로 참조하면, 이미 릴리스된 이전 버전의 워크플로도 최신 내부 액션을 실행하게 됩니다. 따라서 내부 액션도 같은 major 안에서는 이전 버전의 워크플로와 호환되어야 합니다.
- **루트 `README.md`:** 레포 소개와 전체 카탈로그(부품 목록과 한 줄 소개)만 담습니다. 상세한 사용법은 위 표의 매뉴얼에 작성하고, 카탈로그에서 링크합니다. 부품을 추가하거나 삭제하면 카탈로그도 함께 갱신합니다.

## 버전 정책

SemVer를 따르며, 릴리스마다 고정 태그 `vX.Y.Z`와 이동 태그 `vX.Y`(해당 minor의 최신 patch), `vX`(해당 major의 최신 릴리스)를 제공합니다.

## 참조 고정 정책

이 레포의 액션과 워크플로가 다른 액션을 참조할 때는 다음 규칙을 따릅니다.

| 대상 | 고정 방식 | 근거 |
| --- | --- | --- |
| 자체 부품 (`brighteast99/*`) | 이동 태그 (`@vX`) | 태그 이동 주체의 직접 통제, 업데이트 자동 전파 |
| GitHub 공식 액션 (`actions/*`, `github/*`) | ref | 신뢰 가능한 배포 주체 |
| 그 외 서드파티 | 전체 commit SHA와 `# vX.Y.Z` 버전 주석 | 태그 임의 이동에 따른 공급망 공격 대비 |

## composite action 작성 규칙

- 액션 하나당 디렉터리 하나를 사용합니다. 공개 액션에는 매뉴얼인 `README.md`를 함께 두고, 사용 예시, 입력, 출력, 필요한 권한을 적습니다.
- 디렉터리 이름이 곧 공개 경로이므로, 이름을 바꾸는 작업은 breaking change입니다.
- `run` step에는 `shell: bash`를 명시합니다.
- 입력값은 `run` 본문에 `${{ inputs.* }}`로 직접 넣지 않고 `env`로 전달합니다. 표현식이 셸 코드로 해석되는 스크립트 인젝션을 막기 위해서입니다.

## 파일 규칙

- `.gitattributes`로 줄 끝을 LF로 고정합니다. CRLF로 저장된 셸 스크립트는 러너에서 "bad interpreter" 오류를 일으키므로, Windows에서 스크립트를 만들 때 유의해야 합니다.
- `.editorconfig`: UTF-8, 들여쓰기 공백 2칸, 파일 끝 개행, 줄 끝 공백 제거. 단, `*.md`는 줄 끝 공백 2칸으로 줄바꿈하는 문법을 쓰기 때문에 줄 끝 공백을 유지합니다.
- `.claude/settings.json` 같은 공유용 Claude 설정은 git으로 추적하고, 개인 설정인 `.claude/settings.local.json`은 제외합니다.
- 문서와 주석은 한국어로 작성합니다.
