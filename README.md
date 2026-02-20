# nwcommerce GitHub Organization

## Shared Workflows & Templates

### `slack-notify` — Reusable Slack Notification Workflow

배포 시작/성공/실패 알림을 Slack으로 전송하는 재사용 가능한 Workflow입니다.  
각 repo에서 `workflow_call`로 호출하며, 메시지 템플릿은 각 repo에서 직접 관리합니다.

---

#### 레퍼런스

- [GitHub Reusable Workflows 공식 문서](https://docs.github.com/en/actions/sharing-automations/reusing-workflows)
- [Slack Block Kit Builder](https://app.slack.com/block-kit-builder)
- [Slack Incoming Webhooks](https://api.slack.com/messaging/webhooks)

---

#### Inputs

| Input | 필수 | 설명 |
|---|---|---|
| `template_path` | ✅ | Slack Block Kit JSON 템플릿 파일 경로 (호출 repo 기준) |
| `environment` | ✅ | 배포 환경 (예: `PROD`, `STG`, `DEV`) |
| `project_name` | ✅ | 프로젝트명 (예: `collabmaker-api`) |
| `deploy_target` | ❌ | 배포 대상 식별자 (Lambda 함수명, ECS 서비스명, Task 정의명 등) |

#### Secrets

| Secret | 필수 | 설명 |
|---|---|---|
| `slack_webhook_url` | ✅ | Slack Incoming Webhook URL |

#### 템플릿 플레이스홀더

JSON 템플릿 내에서 아래 `{{PLACEHOLDER}}` 형식을 사용하면 자동으로 치환됩니다.

| 플레이스홀더 | 값 |
|---|---|
| `{{ENVIRONMENT}}` | `inputs.environment` |
| `{{PROJECT_NAME}}` | `inputs.project_name` |
| `{{DEPLOY_TARGET}}` | `inputs.deploy_target` |
| `{{GITHUB_SHA}}` | 전체 커밋 SHA |
| `{{GITHUB_SHA_SHORT}}` | 커밋 SHA 앞 7자 |
| `{{GITHUB_ACTOR}}` | 워크플로우 트리거한 사용자 |
| `{{GITHUB_REF_NAME}}` | 브랜치명 |
| `{{GITHUB_SERVER_URL}}` | GitHub 서버 URL |
| `{{GITHUB_REPOSITORY}}` | `owner/repo` |
| `{{GITHUB_RUN_ID}}` | 워크플로우 실행 ID |

---

#### 예제 템플릿

`slack-templates/` 디렉터리에 배포 대상별 예제 템플릿이 있습니다.  
각 repo의 `.github/slack/` 디렉터리에 복사 후 필요에 따라 수정하세요.

```
slack-templates/
├── lambda/
│   ├── notify-start.json
│   ├── notify-success.json
│   └── notify-failure.json
├── ecs-service/
│   ├── notify-start.json
│   ├── notify-success.json
│   └── notify-failure.json
└── ecs-task/
    ├── notify-start.json
    ├── notify-success.json
    └── notify-failure.json
```

---

#### 사용법

**1. Slack Webhook URL을 repo Secret에 등록**

`Settings > Secrets and variables > Actions > New repository secret`
- Name: `DEPLOY_WEBHOOK_URL`

**2. Slack 템플릿 파일 복사**

`slack-templates/{lambda|ecs-service|ecs-task}/` 의 파일을 호출 repo의 `.github/slack/` 에 복사 후 수정.

**3. Workflow에서 호출**

```yaml
# .github/workflows/deploy.yml

jobs:
  notify-start:
    uses: nwcommerce/.github/.github/workflows/slack-notify.yml@main  # ← 주의: workflow-templates/ 아님
    with:
      template_path: .github/slack/notify-start.json
      environment: PROD
      project_name: my-service
      deploy_target: my-lambda-function-name
    secrets:
      slack_webhook_url: ${{ secrets.DEPLOY_WEBHOOK_URL }}

  deploy:
    needs: notify-start
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploy steps here"

  notify-result:
    needs: [notify-start, deploy]
    if: always()
    uses: nwcommerce/.github/.github/workflows/slack-notify.yml@main
    with:
      template_path: ${{ needs.deploy.result == 'success' && '.github/slack/notify-success.json' || '.github/slack/notify-failure.json' }}
      environment: PROD
      project_name: my-service
      deploy_target: my-lambda-function-name
    secrets:
      slack_webhook_url: ${{ secrets.DEPLOY_WEBHOOK_URL }}
```

> **Note:** `workflow-templates/` 디렉터리는 [GitHub Starter Workflow](https://docs.github.com/en/actions/sharing-automations/creating-workflow-templates-for-your-organization) 등록용 위치입니다.  
> Reusable Workflow로 `uses:` 호출 시에는 반드시 `.github/workflows/` 경로에 있어야 합니다.

---

#### Reusable Workflow 경로 규칙

GitHub는 `uses:` 로 호출하는 Reusable Workflow를 `.github/workflows/` 아래에서만 인식합니다.  
`workflow-templates/`에 있는 파일은 호출 불가합니다 ([공식 문서](https://docs.github.com/en/actions/sharing-automations/reusing-workflows#calling-a-reusable-workflow)).
