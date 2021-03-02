# Automate Your Workflows with GitHub Actions and GitHub Packages

:bookmark: The following instructions will take on the assumption that you will be completing the steps on the GitHub UI. Feel free to use the terminal or any other GUIs you are comfortable with.

![workflow](https://user-images.githubusercontent.com/38323656/99338458-dcf99580-2849-11eb-9d17-a2e41376f721.png)

# Workflow 1: Steps to set up CI workflow

1. This repository comes with `npm` project files, namely `package.json` (contains your node project meta data) and `index.js` (your source code).

1. Fork this repository and edit the `package.json` file by replacing the placeholders `OWNER` with your GitHub handle or organization name (if you created the repository in an organization), and `REPOSITORY_NAME`.

1. Create the following file to the default branch:

    - `.github/workflows/ci.yml`
      - This will be the workflow file taking care of building and testing your source code
      - The file content will be empty for now

1. After the above steps are finished, you should have the following files in your repository;
    - `package.json`
    - `index.js`
    - `.github/workflows/ci.yml`

1. :tada: If everything looks fine so far, it's time to start creating our CI workflow!

1. Go to `.github/workflows/ci.yml` and enter edit mode by clicking the pencil :pencil: icon
        <details>
        <summary><b>Click here to view file contents to copy:</b></summary>
        </br>

      ```yaml
      #####################################
      #      Automate your workflow       #
      #####################################

      # This workflow will run CI on your codebase, label your PR, and comment on the result

      name: MYWORKFLOW

      on:
        pull_request: # the workflow will trigger on every pull request event

      jobs:
        build:
          runs-on: ubuntu-latest

          strategy:
            matrix:
              node-version: [12.x, 14.x] # matrix for building and testing your code across multiple node versions

          steps:
            - name: Checkout
              uses: actions/checkout@v2
            - name: Build node version ${{ matrix.node-version }}
              uses: actions/setup-node@v1
              with:
                  node-version: ${{ matrix.node-version }}
            - run: npm install
            - run: npm run build --if-present
            - run: npm test

        label:
          runs-on: ubuntu-latest

          needs: build #this ensures that we only trigger the label job if ci is successful

          steps:
            - name: Checkout
              uses: actions/checkout@v2
            - uses: actions/github-script@v3
              with:
                github-token: ${{ secrets.GITHUB_TOKEN }}
                script: |
                  github.issues.addLabels({
                    issue_number: context.issue.number,
                    owner: context.repo.owner,
                    repo: context.repo.repo,
                    labels: ['release']
                  })

        comment:
          runs-on: ubuntu-latest

          needs: [build, label]

          steps:
            - name: Checkout
              uses: actions/checkout@v2
            - name: Comment on the result
              uses: actions/github-script@v3
              with:
                github-token: ${{ secrets.GITHUB_TOKEN }}
                script: |
                  github.issues.createComment({
                    issue_number: context.issue.number,
                    owner: context.repo.owner,
                    repo: context.repo.repo,
                    body: `
                    Great job **@${context.payload.sender.login}**! Your CI passed, and the PR has been automatically labelled.

                    Once ready, we will merge this PR to trigger the next part of the automation :rocket:
                    `
                  })
      ```
      </details>

- Update the name of your workflow
- Within the `comment` job, you can edit the body of the comment as you see fit
- Commit the changes to your default branch

- **Knowledge Check**

  - How many jobs are there in the workflow?
    <details><summary><b>Answer</b></summary>
    The workflow contains three jobs:

    - a build-job,
    - a label-job,
    - a comment-job
  </details>

  - What is the event that will trigger this workflow?
    <details><summary><b>Answer</b></summary>
    The workflow is triggered by any pull request events.
    </details>

  - Does this workflow use a build matrix?
    <details><summary><b>Answer</b></summary>
    Yes, this workflow will build and test across multiple node versions.
    </details>

:tada: Awesome, now our CI workflow should be complete!

## Test the CI Workflow

1. To test our CI workflow, we need to trigger it. And the event that triggers it is `pull_request`. So let's create a pull request to get this workflow running

1. Go to `index.js` file and enter edit mode.

1. Make some changes/add some text.

1. Commit your changes to a new branch e.g. `add-ci-workflow` and click **Commit changes**

1. In the pull request, leave the title and the body to default and click **Create pull request**

1. In your PR, click on the **Checks** tab
    - You should now see your workflow kicking off, and executing all the steps defined in `.github/workflows/ci.yml`

1. After the workflow has completed, check that the following is true in your PR;
    - A label `release` has been added
    - A comment is added to the PR with the text corresponding to what you defined in the `comment` job inside `.github/workflows/ci.yml`

1. If your workflow fails, inspect the log output:
    - Which job failed?
    - Did you update your `package.json` file correctly?
    - Does the log indicate any syntax errors with your CI workflow file?

:warning: Do not merge your pull request just yet! There's more to do.

# Workflow 2: Steps to set up release and publish package workflow

## Add a workflow that publishes an NPM package and creates a draft release

1. We are going to be using the [`release-drafter` Action](https://github.com/marketplace/actions/release-drafter).
    - Add a workflow file at this directory `.github/workflows/release.yml`:

        <details>
        <summary><b>Click here to view file contents to copy:</b></summary>
        </br>

      ```yaml
      #####################################
      #      Automate your workflow       #
      #####################################

      # This workflow will create a NPM package and a new release

      name: Publish and release

      on:
        push:
          # branches to consider in the event; optional, defaults to all
          branches:
            - main

      jobs:

        update_release_draft:
          runs-on: ubuntu-latest

          steps:
            # Drafts your next Release notes as Pull Requests are merged into "master"
            - uses: release-drafter/release-drafter@v5
              with:
                config-name: release-drafter-config.yml
              env:
                GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

        publish-gpr:
          runs-on: ubuntu-latest

          needs: update_release_draft

          steps:
            - uses: actions/checkout@v2

            - uses: actions/setup-node@v1
              with:
                node-version: 12
                registry-url: https://npm.pkg.github.com/

            - run: npm publish
              env:
                NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      ```
    </details>

    - Notice how this workflow calls for a special config file called `release-drafter-config.yml`. This is a config file that allows you to add more customization. So let's go ahead and add a config file for this action at `github/release-drafter-config.yml`:
        <details>
        <summary><b>Click here to view file contents to copy:</b></summary>
        </br>

      ```yaml
      name-template: 'v$RESOLVED_VERSION 🌈'
      tag-template: 'v$RESOLVED_VERSION'
      categories:
        - title: '🚀 You did it!'
          labels:
            - 'release'
      change-template: '- $TITLE @$AUTHOR (#$NUMBER)'
      change-title-escapes: '\<*_&' # You can add # and @ to disable mentions, and add ` to disable code blocks.
      version-resolver:
        major:
          labels:
            - 'major'
        minor:
          labels:
            - 'minor'
        patch:
          labels:
            - 'patch'
        default: patch
      template: |
        ## Changes

        $CHANGES
      ```
      </details>

    - You can read more about the configuration options for this Action [here](https://github.com/marketplace/actions/release-drafter#configuration-options).
    - This action will create a release with release notes when the pull request is merged into the `main` branch.

## Watch your actions come to life!

1. Merge the pull request

1. Click on the `Code` tab and click on `Packages` on the right sidebar to find your package

1. Click on `Releases` on the right sidebar to find your new draft Release


## Additional Exercise

Customize your `release-drafter` Action such that it is able to categorize release notes based on labels that were added to pull requests. For example, if the label `bugfix` is added to a pull request, then when the pull request is merged, the Action would add the pull request to the release notes and show it as a Bugfix. Here's an example:

<img src="https://github.com/release-drafter/release-drafter/blob/master/design/screenshot-2.png" align-=center width=400>

   - You can find out how to add more customizations from the configuration options for this Action [here](https://github.com/marketplace/actions/release-drafter#configuration-options).
        <details>
        <summary><b>Click here to see the example</b></summary>
        </br>

        ```yaml
        name-template: 'v$RESOLVED_VERSION 🌈'
        tag-template: 'v$RESOLVED_VERSION'
        categories:
          - title: '🚢 Features'
            labels:
              - 'feature'
              - 'enhancement'
          - title: '🐛 Bug Fixes'
            labels:
              - 'fix'
              - 'bugfix'
              - 'bug'
          - title: '🧰 Maintenance'
            label: 'chore'
        change-template: '- $TITLE @$AUTHOR (#$NUMBER)'
        change-title-escapes: '\<*_&' # You can add # and @ to disable mentions, and add ` to disable code blocks.
        version-resolver:
          major:
            labels:
              - 'major'
          minor:
            labels:
              - 'minor'
          patch:
            labels:
              - 'patch'
          default: patch
        template: |
          ## Changes

          $CHANGES
        ```

        <b>The config file above allows you to add more labels like 'feature' and 'bug'.</b>
  </details>

