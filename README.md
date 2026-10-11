# Epinio GitLab CI/CD component

Reusable GitLab CI/CD component that installs the Epinio CLI and deploys an application with `epinio push`.

Also ships an equivalent **raw job snippet** for self-managed / older GitLab setups that cannot consume CI/CD Catalog components.

## Component usage

Add a `deploy` stage, set CI/CD variables, and include the component:

```yaml
stages:
  - deploy

include:
  - component: $CI_SERVER_FQDN/<group>/epinio-gitlab-ci/deploy@1.0.0
    inputs:
      app_name: $CI_PROJECT_NAME
      namespace: workspace
      source_path: "."
      # git_ref: "main"                 # optional: push from git instead of path
      # image_tag: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
      epinio_url: $EPINIO_URL
```

Replace `<group>/epinio-gitlab-ci` with your catalog project path after publishing.

### Inputs

| Input | Default | Description |
| --- | --- | --- |
| `app_name` | `$CI_PROJECT_NAME` | Epinio application name |
| `namespace` | `workspace` | Epinio namespace to target |
| `source_path` | `.` | Local path for path-based push |
| `git_ref` | `""` | When set, push from `$CI_PROJECT_URL.git` at this revision |
| `epinio_url` | `$EPINIO_URL` | Epinio API URL |
| `image_tag` | `""` | When set, deploy this container image (`registry/name:tag`) |

Push mode (first match wins):

1. `image_tag` set → `--container-image-url`
2. `git_ref` set → `--git`
3. otherwise → `--path` using `source_path`

### Required CI/CD variables

| Variable | Purpose |
| --- | --- |
| `EPINIO_URL` | API URL (unless overridden by `epinio_url`) |
| `EPINIO_TOKEN` **or** `EPINIO_USER` + `EPINIO_PASSWORD` | Authentication |

Optional: `EPINIO_TRUST_CA=true` to pass `--trust-ca` on user/password login (self-signed API certs).

## Catalog publishing

1. In the GitLab project: **Settings → General → CI/CD Catalog project** → enable.
2. Set a clear **project description**.
3. Ensure this `README.md` and `templates/deploy.yml` exist on the commit you tag.
4. Create a **semver** tag (`1.0.0` or `v1.0.0`). The `create-release` job uses the `release` keyword so the version is published to the CI/CD Catalog.

## Raw job

For teams that cannot use `include:component` (self-managed without catalog access, older GitLab, or paste-into-pipeline), use the job below. It is maintained at [`examples/raw-job.gitlab-ci.yml`](examples/raw-job.gitlab-ci.yml) and reproduced here **verbatim**.

```yaml
# Equivalent raw job for pipelines that cannot include the CI/CD component
# (older GitLab, air-gapped without catalog, or copy-paste into an existing
# .gitlab-ci.yml). Keep this file identical to the README "Raw job" section.
#
# Required CI/CD variables: EPINIO_URL, and either EPINIO_TOKEN or
# EPINIO_USER + EPINIO_PASSWORD.
# Optional: EPINIO_TRUST_CA=true, EPINIO_APP_NAME, EPINIO_NAMESPACE,
# EPINIO_SOURCE_PATH, EPINIO_GIT_REF, EPINIO_IMAGE_TAG.

stages:
  - deploy

deploy-epinio:
  stage: deploy
  image: alpine:3.21
  variables:
    EPINIO_APP_NAME: "$CI_PROJECT_NAME"
    EPINIO_NAMESPACE: "workspace"
    EPINIO_SOURCE_PATH: "."
    EPINIO_GIT_REF: ""
    EPINIO_IMAGE_TAG: ""
  before_script:
    - apk add --no-cache ca-certificates curl
    - |
      set -eu
      ARCH="$(uname -m)"
      case "$ARCH" in
        x86_64|amd64) ARCH=x86_64 ;;
        aarch64|arm64) ARCH=arm64 ;;
        armv7l) ARCH=armv7 ;;
        *) echo "Unsupported architecture: $ARCH"; exit 1 ;;
      esac
      VERSION="$(curl -fsSL https://api.github.com/repos/epinio/epinio/releases/latest | sed -n 's/.*"tag_name": "\(v[^"]*\)".*/\1/p' | head -n1)"
      test -n "$VERSION"
      curl -fsSL -o /usr/local/bin/epinio \
        "https://github.com/epinio/epinio/releases/download/${VERSION}/epinio-linux-${ARCH}"
      chmod +x /usr/local/bin/epinio
      epinio --version
  script:
    - |
      set -eu
      URL="${EPINIO_URL:-}"
      if [ -z "$URL" ]; then
        echo "EPINIO_URL variable is required"
        exit 1
      fi

      if [ -n "${EPINIO_TOKEN:-}" ]; then
        mkdir -p "${HOME}/.config/epinio"
        API="${URL%/}"
        case "$API" in
          https://*) WSS="wss://${API#https://}" ;;
          http://*) WSS="ws://${API#http://}" ;;
          *) WSS="$API" ;;
        esac
        printf '%s\n' \
          "api: \"${API}\"" \
          "wss: \"${WSS}\"" \
          'user: ""' \
          'pass: ""' \
          "colors: true" \
          "namespace: \"${EPINIO_NAMESPACE}\"" \
          "token:" \
          "  accesstoken: \"${EPINIO_TOKEN}\"" \
          > "${HOME}/.config/epinio/settings.yaml"
      elif [ -n "${EPINIO_USER:-}" ] && [ -n "${EPINIO_PASSWORD:-}" ]; then
        if [ "${EPINIO_TRUST_CA:-}" = "true" ]; then
          epinio login "$URL" --user "$EPINIO_USER" --password "$EPINIO_PASSWORD" --trust-ca
        else
          epinio login "$URL" --user "$EPINIO_USER" --password "$EPINIO_PASSWORD"
        fi
        epinio target "${EPINIO_NAMESPACE}"
      else
        echo "Set EPINIO_TOKEN, or EPINIO_USER and EPINIO_PASSWORD"
        exit 1
      fi

      if [ -n "${EPINIO_IMAGE_TAG}" ]; then
        epinio push --name "${EPINIO_APP_NAME}" --container-image-url "${EPINIO_IMAGE_TAG}"
      elif [ -n "${EPINIO_GIT_REF}" ]; then
        epinio push --name "${EPINIO_APP_NAME}" --git "${CI_PROJECT_URL}.git,${EPINIO_GIT_REF}"
      else
        epinio push --name "${EPINIO_APP_NAME}" --path "${EPINIO_SOURCE_PATH}"
      fi
```

## Mirror procedure (self-managed GitLab)

GitLab.com catalog components are not available on Self-Managed by default. To host this entry on your own instance:

1. Allow outbound network access to the upstream GitLab host you are mirroring from (for example `gitlab.com`), if you use pull mirroring.
2. Create a group to hold components (recommended name: `components`).
3. Create a **mirror** of this project in that group (**New project → Import project → Repository by URL**, or **Settings → Repository → Mirroring repositories** on an empty project).
4. Set a **project description** on the mirror (mirroring does not copy the description).
5. Enable **CI/CD Catalog project** on the mirror.
6. Publish into your instance catalog by running a pipeline for a **semver tag** (usually the latest upstream tag such as `1.0.0`). The `create-release` job must succeed so the `release` keyword registers the version.

After that, consumers on the self-managed instance include:

```yaml
include:
  - component: $CI_SERVER_FQDN/components/epinio-gitlab-ci/deploy@1.0.0
    inputs:
      epinio_url: $EPINIO_URL
```

If catalog mirroring is not possible, copy the [raw job](#raw-job) into the application pipeline instead.

## License

Apache License 2.0
