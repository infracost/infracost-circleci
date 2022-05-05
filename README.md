# Infracost CircleCI

This project provides instructions for using Infracost in a CircleCI pipeline, it works with both GitHub and Bitbucket Cloud. This enables you to see cloud cost estimates for Terraform in pull requests. 💰

## Table of contents

* [GitHub Quick start](#github-quick-start)
* [Bitbucket Quick start](#bitbucket-cloud-quick-start)
* [Comment options](#comment-options)
* [Examples](#examples)
* [Contributing](#contributing)
* [License](#license)

## GitHub Quick start

![Example GitHub screenshot](https://github.com/infracost/actions/blob/master/.github/assets/screenshot.png?raw=true)

1. In CircleCI, go to your Project Settings > Environment Variables, and add environment variables for `INFRACOST_API_KEY`, `GITHUB_TOKEN`, and any other required credentials (e.g. `AWS_ACCESS_KEY_ID`).

2. We recommend you enable CircleCI's "Only build pull requests" option. This setting is in CircleCI's Project > Advanced settings page.

3. Create a new file at `.circleci/config.yml` in your repo with the following content.

    ```yml
    version: 2.1

    jobs:
      terraform_plan:
        docker:
          - image: hashicorp/terraform:latest
        environment:
          # If your terraform files are in a subdirectory, set TF_ROOT accordingly
          TF_ROOT: PATH/TO/TERRAFORM/CODE # Update this!
        steps:
          - run:
              name: Skip if not pull request
              command: |
                if [ "$CIRCLE_PULL_REQUEST" == "" ]; then
                  circleci step halt
                fi
          - checkout
          - run:
              name: Run terraform plan
              command: |
                # IMPORTANT: add any required steps here to setup cloud credentials so Terraform can run
                cd $TF_ROOT
                terraform init
                terraform plan -out tfplan.binary
                terraform show -json tfplan.binary > /tmp/plan.json
          - persist_to_workspace:
              root: /tmp
              paths:
                - plan.json
      infracost:
        working_directory: terraform
        docker:
          # Always use the latest 0.9.x version to pick up bug fixes and new resources.
          # See https://www.infracost.io/docs/integrations/cicd/#docker-images for other options
          - image: infracost/infracost:ci-0.9
        steps:
          - run:
              name: Skip if not pull request
              command: |
                if [ "$CIRCLE_PULL_REQUEST" == "" ]; then
                  circleci step halt
                fi
          - attach_workspace:
              at: /tmp
          - checkout
          - run:
              name: Run Infracost breakdown
              command: |
                # Generate Infracost JSON output, the following docs might be useful:
                # Multi-project/workspaces: https://www.infracost.io/docs/features/config_file
                # Combine Infracost JSON files: https://www.infracost.io/docs/features/cli_commands/#combined-output-formats
                infracost breakdown --path /tmp/plan.json --format json --out-file infracost.json
          - run:
              name: Run Infracost comment
              command: |
                # Extract the PR number from the PR URL
                PULL_REQUEST_NUMBER=${CIRCLE_PULL_REQUEST##*/}
                # See the 'Comment options' section in our README below for other options.
                infracost comment github --path infracost.json --repo $CIRCLE_PROJECT_USERNAME/$CIRCLE_PROJECT_REPONAME --pull-request $PULL_REQUEST_NUMBER --github-token $GITHUB_TOKEN

    workflows:
      infracost:
        jobs:
          - terraform_plan
          - infracost:
              requires:
                - terraform_plan
    ```

4. 🎉 That's it! Send a new pull request to change something in Terraform that costs money. You should see a pull request comment that gets updated, e.g. the 📉 and 📈 emojis will update as changes are pushed!

5. Follow [the docs](https://www.infracost.io/usage-file) if you'd also like to show cost for of usage-based resources such as AWS Lambda or S3. The usage for these resources are fetched from CloudWatch/cloud APIs and used to calculate an estimate.

## Bitbucket Cloud Quick start

![Example Bitbucket screenshot](https://bytebucket.org/infracost/infracost-bitbucket-pipeline/raw/master/screenshot.png)

1. In CircleCI, go to your Project Settings > Environment Variables, and add environment variables for `INFRACOST_API_KEY`, `BITBUCKET_TOKEN`, and any other required credentials (e.g. `AWS_ACCESS_KEY_ID`).

2. We recommend you enable CircleCI's "Only build pull requests" option. This setting is in CircleCI's Project > Advanced settings page.

3. Create a new file at `.circleci/config.yml` in your repo with the following content.

    ```yml
    version: 2.1

    jobs:
      terraform_plan:
        docker:
          - image: hashicorp/terraform:latest
        environment:  # If your terraform files are in a subdirectory, set TF_ROOT accordingly
          TF_ROOT: PATH/TO/TERRAFORM/CODE # Update this!
        steps:
          - run:
              name: Skip if not pull request
              command: |
                if [ "$CIRCLE_PULL_REQUEST" == "" ]; then
                  circleci step halt
                fi
          - checkout
          - run:
              name: Run terraform plan
              command: |
                # IMPORTANT: add any required steps here to setup cloud credentials so Terraform can run
                cd $TF_ROOT
                terraform init
                terraform plan -out tfplan.binary
                terraform show -json tfplan.binary > /tmp/plan.json
          - persist_to_workspace:
              root: /tmp
              paths:
                - plan.json
      infracost:
        working_directory: terraform
        docker:
          # Always use the latest 0.9.x version to pick up bug fixes and new resources.
          # See https://www.infracost.io/docs/integrations/cicd/#docker-images for other options
          - image: infracost/infracost:ci-0.9
        steps:
          - run:
              name: Skip if not pull request
              command: |
                if [ "$CIRCLE_PULL_REQUEST" == "" ]; then
                  circleci step halt
                fi
          - attach_workspace:
              at: /tmp
          - checkout
          - run:
              name: Run Infracost breakdown
              command: |
                # Generate Infracost JSON output, the following docs might be useful:
                # Multi-project/workspaces: https://www.infracost.io/docs/features/config_file
                # Combine Infracost JSON files: https://www.infracost.io/docs/features/cli_commands/#combined-output-formats
                infracost breakdown --path /tmp/plan.json --format json --out-file infracost.json
          - run:
              name: Run Infracost comment
              command: |
                # Extract the PR number from the PR URL
                PULL_REQUEST_NUMBER=$(echo "$CIRCLE_PULL_REQUEST" | sed 's/.*pull-requests\///')
                # See the 'Comment options' section in our README below for other options.
                infracost comment bitbucket --path infracost.json --repo $CIRCLE_PROJECT_USERNAME/$CIRCLE_PROJECT_REPONAME --pull-request $PULL_REQUEST_NUMBER --bitbucket-token $BITBUCKET_TOKEN

    workflows:
      infracost:
        jobs:
          - terraform_plan
          - infracost:
              requires:
                - terraform_plan
    ```

4. 🎉 That's it! Send a new pull request to change something in Terraform that costs money. You should see a pull request comment that gets updated, e.g. the '↑' and '↓' characters will update as changes are pushed!

## Comment options

For different commenting options `infracost comment` command supports the following flags:

- `--behavior <value>`: Optional, defaults to `update`. The behavior to use when posting cost estimate comments. Must be one of the following:
  - `update`: Create a single comment and update it on changes. This is the "quietest" option. Pull request followers will only be notified on the comment create (not updates), and the comment will stay at the same location in the comment history.
  - `delete-and-new`: Delete previous cost estimate comments and create a new one. Pull request followers will be notified on each comment.
  - `hide-and-new`: Minimize previous cost estimate comments and create a new one. Pull request followers will be notified on each comment. Only supported by GitHub.
  - `new`: Create a new cost estimate comment. Pull request followers will be notified on each comment.
- `--pull-request <pull-request-number>`: Required when posting a comment on a pull request. Mutually exclusive with `--commit` flag.
- `--commit <commit-sha>`: Required when posting a comment on a commit. Mutually exclusive with `--pull-request` flag. Not available when bitbucket-server-url is set.
- `--tag <tag>`:  Optional. Customize hidden markdown tag used to detect comments posted by Infracost. This is useful if you have multiple workflows that post comments to the same pull request or commit and you want to avoid them over-writing each other.

Run `infracost comment github --help` or `infracost comment bitbucket --help` to see the the full list of options or [see our docs](https://www.infracost.io/docs/features/cli_commands#comment-on-pull-requests).

## Examples

We don't yet have examples for different use cases with Infracost for CircleCI, but we do have a selection of [examples for GitLab](https://gitlab.com/infracost/infracost-gitlab-ci/-/tree/master/examples#examples) that can be modified to work with CircleCI.

## Contributing

Issues and pull requests are welcome. Please create issues in [this repo](https://github.com/infracost/infracost) or [join our community Slack slack](https://www.infracost.io/community-chat), we are a friendly bunch and happy to help you get started :)

## License

[Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/)
