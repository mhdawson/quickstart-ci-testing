# Quickstart CI Testing

CI system that clones, deploys, and tests quickstarts from the [rh-ai-quickstart](https://github.com/rh-ai-quickstart) org. Adding a new quickstart requires a workflow file that calls the shared composite actions. Each workflow owns its own secrets so the shared actions stay generic.

## Architecture

```
.github/
├── actions/
│   ├── setup-and-deploy/   # Prepare runner, clone repo, deploy (OpenShift or KinD)
│   ├── run-quickstart-tests/ # Run pre-test setup and tests
│   ├── cleanup/              # Tear down deployment (runs always)
│   ├── prepare-runner/     # Free disk space on ubuntu-latest runners
│   ├── kind/               # KinD cluster lifecycle (opt-in)
│   ├── clone-quickstart/   # Clone a quickstart repo
│   ├── deploy-quickstart/  # Deploy to OpenShift (default)
│   └── run-tests/          # Run tests via configurable command
└── workflows/
    ├── test-it-self-service-agent.yml
    ├── test-speeding-up-ticket-resolution.yml
    ├── test-rag.yml
    └── test-ai-supply-chain-agent.yml
```

**Default deploy target:** OpenShift (via `oc login`).
**Alternative:** KinD (set `use_kind: true` on the `setup-and-deploy` action).

All quickstart tests share a single concurrency group (`quickstart-ci-tests`) so only one runs at a time.

## Quickstarts

| Quickstart | Deploy Command | Test Command |
|---|---|---|
| it-self-service-agent | `make helm-install-test` | `make test-short-resp-integration-request-mgr` |
| speeding-up-ticket-resolution | `make install` | `make test-short-ticket-laptop-refresh` |
| RAG | `helm install rag deploy/helm/rag` | `pytest tests/integration/llamastack/ -v` |
| ai-supply-chain-agent | `make helm-install` | `make kind-verify` |

## Adding a New Quickstart

Create a new file `.github/workflows/test-<name>.yml`. Set secrets via step-level `env:` on each action — deploy and test can have different secrets:

```yaml
name: Test - My Quickstart

on:
  workflow_dispatch:

concurrency:
  group: quickstart-ci-tests
  cancel-in-progress: false

jobs:
  test:
    name: my-quickstart
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - uses: ./.github/actions/setup-and-deploy
        with:
          repo_url: https://github.com/rh-ai-quickstart/my-quickstart
          deploy_command: "make helm-install"
        env:
          PROD_TOKEN: ${{ secrets.PROD_TOKEN }}
          PROD_SERVER: ${{ secrets.PROD_SERVER }}
          DEPLOY_SECRET: ${{ secrets.DEPLOY_SECRET }}

      - uses: ./.github/actions/run-quickstart-tests
        with:
          test_command: "make test"
        env:
          TEST_SECRET: ${{ secrets.TEST_SECRET }}

      - uses: ./.github/actions/cleanup
        if: always()
        with:
          cleanup_command: "make helm-uninstall NAMESPACE=$NAMESPACE"
```
