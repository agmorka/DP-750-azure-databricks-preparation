# Module 14: Implement development lifecycle processes in Azure Databricks

Module link: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/

Part of: [Deploy and Maintain Data Pipelines and Workloads with Azure Databricks](../paths/deploy-maintain-data-pipelines-workloads.md)

## Overview

- Units: introduction, Git version control, branching and pull requests, testing strategy, Declarative Automation Bundles, CLI deployment, exercise, knowledge check, and summary
- Prerequisites: Git basics, Python, and Azure Databricks workspaces
- Why it matters: the exam tests Git folder workflows, the testing pyramid, bundle structure (`databricks.yml`, targets, variables), and the CLI commands (validate, plan, deploy)

## Learning objectives

1. Apply Git version control best practices with Git folders.
2. Manage branches, pull requests, and merge conflicts.
3. Implement a testing strategy: unit, integration, end-to-end, and UAT.
4. Configure and customize Declarative Automation Bundles.
5. Deploy bundles with the Databricks CLI across environments.

## Key points

### 1. Git folders

- Git folders connect a workspace folder to a remote repository on Azure DevOps, GitHub, GitLab, or Bitbucket. You can clone, branch, commit, and push from the workspace.
- To create one: Workspace, Create, Git folder. Enter the URL, pick the provider, name the folder, and create it.
- Each team member should use their own Git folder against the shared repository. Several users switching branches in one shared folder affect each other.
- Pull regularly. Pulling changes to notebook source code resets the notebook state (outputs, comments, version history for that notebook).
- Use `.gitignore` for temporary output, credentials, and environment-specific files. Files already tracked must be removed from Git before `.gitignore` applies to them.
- Limits: working branches up to 1 GB, and Databricks recommends fewer than 20,000 assets. Use sparse checkout for large repositories.
- Local branches persist for up to 30 days after the remote branch is deleted.

### 2. Branches, commits, and pull requests

- Create a branch in the Git dialog (Create Branch), based on `main`.
- Naming patterns from the unit: `feature/...`, `bugfix/...`, and `users/<username>/...`.
- Uncommitted changes carry over when you switch branches if they do not conflict. Switching can also delete workspace assets that do not exist on the target branch.
- **Commit & Push** sends changes to the remote. If you cannot commit to a protected branch such as `main`, create a feature branch first.
- Notebook outputs are not committed for source formats such as `.py`, `.scala`, `.sql`, or `.r`. Use IPYNB if outputs must be versioned.
- Pull requests are created in the Git provider, not in Databricks. The typical flow: clone, branch from `main`, change, commit and push, open a PR, review, merge into the deployment branch, then pull.

### 3. Conflicts

- Conflicts happen when two people change the same lines during pull, merge, or rebase.
- Options: **Keep all current changes**, **Take all incoming changes**, or manual edit. Manual edits use conflict markers (`<<<<<<< HEAD`, `=======`, `>>>>>>> branch`).
- After editing, select **Mark As Resolved**, then **Continue Merge** or **Continue Rebase**. **Abort** cancels and returns to the previous state.
- Pull often, so merges stay small.

### 4. Testing strategy

- The testing pyramid: many fast **unit tests**, fewer **integration tests**, and a few **end-to-end** tests. **UAT** sits on top and confirms business requirements.
- Unit tests use pytest. Test files must be named `test_*.py` or `*_test.py`. Use `@pytest.fixture` for sample data. Use synthetic data that matches production structure.
- Run pytest in a notebook: `%pip install pytest`, then `pytest.main([".", "-v", "-p", "no:cacheprovider"])`, and assert the return code is 0.
- Tip: functions should return one predictable type. Return an empty DataFrame, not `False`.
- Integration tests run against real Databricks resources, using a dedicated test catalog or schema. Never run them against production tables.
- End-to-end tests run the whole pipeline on known input and check the output. They are slow, so run them on a schedule or in the deployment pipeline.
- UAT needs acceptance criteria agreed before development, a staging environment that mirrors production, documented expected outcomes, and a feedback process.
- Project layout from the unit: `src/` for code, `tests/unit`, `tests/integration`, `tests/e2e`, and a `notebooks/run_tests.py` runner. Keep functions in `.py` files inside a Git folder and import them into notebooks.
- Run the test notebook as a Databricks job before deploying to production.

### 5. Declarative Automation Bundles

- Bundles were previously called Databricks Asset Bundles. They are infrastructure as code. Configuration is YAML by default. Python configuration is in Public Preview.
- The root file is `databricks.yml`. Its main sections are `bundle` (name), `include` (other YAML files, for example `resources/*.yml`), `workspace`, `resources` (jobs, pipelines), and `targets`.
- **Variables** are defined under `variables:` with a `description` and a `default`. Reference them as `${var.name}`. Override values in a target's `variables`, on the command line with `--var="name=value"`, or with an environment variable that starts with `BUNDLE_VAR_`.
- **Jobs** have a `name` and `tasks`. Each task has a `task_key`. Task types include `notebook_task` and `spark_python_task`. `depends_on` sets dependencies and `base_parameters` passes parameters.
- Compute: `existing_cluster_id` for an existing cluster, or `job_clusters` with a `job_cluster_key` for a job cluster.
- Paths must be relative to the bundle root.
- **Targets** are deployment environments. Each has its own workspace and settings.
  - `mode: development` prefixes resource names with `[dev <username>]`, pauses schedules, and allows concurrent runs.
  - `mode: production` checks that the local Git branch matches the configured branch, and recommends a service principal for deployments.
  - `presets` fine-tune a target. The unit's example sets a name prefix, a paused trigger status, and tags for staging.

### 6. CLI deployment

- `databricks bundle validate`: checks the configuration and shows the bundle name, target, and workspace.
- `databricks bundle plan`: previews what will be created, updated, or removed. Use `-t <target>` for a non-default target.
- `databricks bundle deploy`: uploads artifacts and creates or updates resources. Removed resources are deleted from the workspace. Use `-t dev` to choose a target. Add `--auto-approve` in CI/CD.
- `databricks bundle summary`: lists deployed resources with links.
- `databricks bundle open <resource>`: opens a resource in the browser.
- `databricks auth login --host <url>`: sets credentials.
- Deployment state is stored in the workspace. Each deployment is identified by bundle name, target, and deploying user. Two people deploying to the same target conflict.
- Troubleshooting: `--force-lock` clears a stale lock, but only when no other deployment is running. `--fail-on-active-runs` stops a deployment if its jobs are running. Validation warnings often mean a CLI version mismatch.
- The unit also notes that bundles can be managed from the workspace UI for ad hoc work, while the CLI is the recommended route for CI/CD.
- The module's CLI pages do not cover `bundle run` or `bundle destroy`. Check the CLI reference before the exam.

## Quick review checklist

- [ ] I can explain why each developer should use their own Git folder.
- [ ] I know the branch naming patterns and the rule for protected branches.
- [ ] I know that notebook outputs are not committed for `.py` and `.sql` source files.
- [ ] I can name the three conflict resolution options and the Continue and Abort buttons.
- [ ] I know the four test levels and where each one runs.
- [ ] I know that integration tests must not run against production tables.
- [ ] I can read a `databricks.yml` with `bundle`, `include`, `resources`, and `targets`.
- [ ] I know the three ways to set a bundle variable and the `${var.name}` syntax.
- [ ] I know what `mode: development` and `mode: production` change.
- [ ] I know the order of validate, plan, and deploy.
- [ ] I know when to use `--force-lock` and `--fail-on-active-runs`.

## Knowledge check and exercise

Assessment: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/8-knowledge-check

The exercise and knowledge check were not captured in these notes. Review them on the module page before the exam.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/
- Knowledge check: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/8-knowledge-check
- Unit 2, Git version control: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/2-apply-version-control-git
- Unit 3, branching and pull requests: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/3-manage-branching-pull-requests-conflicts
- Unit 4, testing strategy: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/4-implement-testing-strategy
- Unit 5, bundles: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/5-configure-package-databricks-asset-bundles
- Unit 6, CLI deployment: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/6-deploy-bundle-databricks-cli
- Unit 9, summary: https://learn.microsoft.com/en-us/training/modules/implement-development-lifecycle-processes-in-azure-databricks/9-summary
- Collaborate on bundles in the workspace: https://learn.microsoft.com/en-us/azure/databricks/dev-tools/bundles/workspace

Source note: several unit pages returned condensed text rather than verbatim content, so some wording above is a summary. Check exact syntax against the module pages before relying on it.

## Back to index

[README](../README.md)
