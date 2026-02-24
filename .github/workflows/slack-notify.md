# slack-notify — Reusable Slack Notification Workflow

배포 시작/성공/실패 알림을 Slack으로 전송하는 재사용 가능한 Workflow입니다.
각 repo에서 `workflow_call`로 호출하며, 중앙 관리되는 템플릿을 `template_name`으로 참조합니다.

---

<details>
<summary><strong>Inputs & Secrets</strong></summary>

#### Inputs

| Input | 필수 | 설명 |
|---|---|---|
| `template_name` | ✅ | 중앙 템플릿 경로 (예: `ecs-service/notify-start.json`). `slack-templates/` 기준 상대 경로 |
| `environment` | ✅ | 배포 환경 (예: `PROD`, `STG`, `DEV`) |
| `project_name` | ✅ | 프로젝트명 (예: `collabmaker-api`) |
| `deploy_target` | ❌ | 배포 대상 식별자 (Lambda 함수명, ECS 서비스명, Task 정의명 등) |
| `cluster_name` | ❌ | ECS 클러스터명 (ecs-task, ecs-service 템플릿용) |

#### Secrets

| Secret | 필수 | 설명 |
|---|---|---|
| `slack_webhook_url` | ✅ | Slack Incoming Webhook URL |

</details>

---

<details>
<summary><strong>템플릿 플레이스홀더</strong></summary>

JSON 템플릿 내에서 아래 `{{PLACEHOLDER}}` 형식을 사용하면 자동으로 치환됩니다.

| 플레이스홀더 | 값 |
|---|---|
| `{{ENVIRONMENT}}` | `inputs.environment` |
| `{{PROJECT_NAME}}` | `inputs.project_name` |
| `{{DEPLOY_TARGET}}` | `inputs.deploy_target` |
| `{{CLUSTER_NAME}}` | `inputs.cluster_name` (ECS 전용) |
| `{{GITHUB_SHA}}` | 전체 커밋 SHA |
| `{{GITHUB_SHA_SHORT}}` | 커밋 SHA 앞 7자 |
| `{{GITHUB_ACTOR}}` | 워크플로우 트리거한 사용자 |
| `{{GITHUB_REF_NAME}}` | 브랜치명 |
| `{{GITHUB_SERVER_URL}}` | GitHub 서버 URL |
| `{{GITHUB_REPOSITORY}}` | `owner/repo` |
| `{{GITHUB_RUN_ID}}` | 워크플로우 실행 ID |

</details>

---

<details>
<summary><strong>중앙 템플릿 목록</strong></summary>

`slack-templates/` 디렉터리에 배포 대상별 템플릿이 있습니다.
`template_name`으로 직접 참조할 수 있습니다.

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

템플릿은 Slack `attachments` + `color` 를 사용해 상태별 color bar를 표시합니다.

| 템플릿 | Color bar | Emoji |
|---|---|---|
| `notify-start.json` | `#000000` (black) | 🚀 |
| `notify-success.json` | `#2EB67D` (green) | 🎉 |
| `notify-failure.json` | `#E01E5A` (red) | 🚨 |

</details>

---

### 사용법

**1. Slack Webhook URL을 repo Secret에 등록**

`Settings > Secrets and variables > Actions > New repository secret`
- Name: `DEPLOY_WEBHOOK_URL`

**2. Workflow에서 호출**

> **주의:** `workflow_call`의 `with` 블록에서는 `needs` 컨텍스트를 사용할 수 없습니다.
> 성공/실패 알림은 `notify-success` / `notify-failure` job을 각각 분리하고 `if:` 조건으로 제어하세요.

<details>
<summary><strong>Lambda 예시</strong></summary>

```yaml
# .github/workflows/deploy.yml

jobs:
  notify-start:
    uses: nwcommerce/.github/.github/workflows/slack-notify.yml@main
    with:
      template_name: lambda/notify-start.json
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

  notify-success:
    needs: [notify-start, deploy]
    if: needs.deploy.result == 'success'
    uses: nwcommerce/.github/.github/workflows/slack-notify.yml@main
    with:
      template_name: lambda/notify-success.json
      environment: PROD
      project_name: my-service
      deploy_target: my-lambda-function-name
    secrets:
      slack_webhook_url: ${{ secrets.DEPLOY_WEBHOOK_URL }}

  notify-failure:
    needs: [notify-start, deploy]
    if: needs.deploy.result != 'success'
    uses: nwcommerce/.github/.github/workflows/slack-notify.yml@main
    with:
      template_name: lambda/notify-failure.json
      environment: PROD
      project_name: my-service
      deploy_target: my-lambda-function-name
    secrets:
      slack_webhook_url: ${{ secrets.DEPLOY_WEBHOOK_URL }}
```

</details>

<details>
<summary><strong>ECS 예시</strong> (ecs-service / ecs-task 공통)</summary>

```yaml
# .github/workflows/deploy.yml

jobs:
  notify-start:
    uses: nwcommerce/.github/.github/workflows/slack-notify.yml@main
    with:
      template_name: ecs-service/notify-start.json
      environment: PROD
      project_name: my-service
      deploy_target: my-ecs-service-name
      cluster_name: my-ecs-cluster
    secrets:
      slack_webhook_url: ${{ secrets.DEPLOY_WEBHOOK_URL }}

  deploy:
    needs: notify-start
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploy steps here"

  notify-success:
    needs: [notify-start, deploy]
    if: needs.deploy.result == 'success'
    uses: nwcommerce/.github/.github/workflows/slack-notify.yml@main
    with:
      template_name: ecs-service/notify-success.json
      environment: PROD
      project_name: my-service
      deploy_target: my-ecs-service-name
      cluster_name: my-ecs-cluster
    secrets:
      slack_webhook_url: ${{ secrets.DEPLOY_WEBHOOK_URL }}

  notify-failure:
    needs: [notify-start, deploy]
    if: needs.deploy.result != 'success'
    uses: nwcommerce/.github/.github/workflows/slack-notify.yml@main
    with:
      template_name: ecs-service/notify-failure.json
      environment: PROD
      project_name: my-service
      deploy_target: my-ecs-service-name
      cluster_name: my-ecs-cluster
    secrets:
      slack_webhook_url: ${{ secrets.DEPLOY_WEBHOOK_URL }}
```

</details>

---

<details>
<summary><strong>Reusable Workflow 경로 규칙</strong></summary>

GitHub는 `uses:` 로 호출하는 Reusable Workflow를 `.github/workflows/` 아래에서만 인식합니다.
`workflow-templates/`에 있는 파일은 호출 불가합니다 ([공식 문서](https://docs.github.com/en/actions/sharing-automations/reusing-workflows#calling-a-reusable-workflow)).

</details>

---

#### 레퍼런스

- [GitHub Reusable Workflows 공식 문서](https://docs.github.com/en/actions/sharing-automations/reusing-workflows)
- [Slack Block Kit Builder](https://app.slack.com/block-kit-builder)
- [Slack Incoming Webhooks](https://api.slack.com/messaging/webhooks)
