# common-ci

도메인과 무관하게 재사용하는 GitHub Actions 공통 부품(composite action, reusable workflow, pre-commit hook)을 제공합니다.
terraform-ci 같은 도메인별 CI 레포도 이 레포의 부품을 기반으로 동작합니다.

## 사용

```yaml
# composite action (step 단위)
steps:
  - uses: brighteast99/common-ci/actions/<name>@v1

# reusable workflow (job 단위)
jobs:
  example:
    uses: brighteast99/common-ci/.github/workflows/<name>.yml@v1
```

## 버전

[SemVer](https://semver.org/)를 따르며, 릴리스마다 다음 태그를 제공합니다.

- `vX.Y.Z`: 릴리스별 고정 태그
- `vX.Y`: 해당 minor 버전의 최신 patch를 가리키는 이동 태그
- `vX`: 해당 major 버전의 최신 릴리스를 가리키는 이동 태그

## 카탈로그

### Composite actions

| 이름 | 설명 |
| ---- | ---- |

### Reusable workflows

| 이름 | 설명 |
| ---- | ---- |

### Pre-commit hooks

| 이름 | 설명 |
| ---- | ---- |

## 라이선스

[MIT](LICENSE)
