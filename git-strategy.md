> **Please discuss about 2 below git branch strategy. All comments are welcome. After we finalize the git strategy, the other will be removed. I recommend the github flow, because we don't need to worry about merging and conflict when have a new release**


# Git flow

![git-flow](/.attachments/git-flow.png)

## Git branch 

| Index | Branch      | Note                                                                                        |
|-------|-------------|---------------------------------------------------------------------------------------------|
| 1     | master      | Code on master branch will be built to package for installer                                |
| 2     | hot-fix     | Branch to fix bug on production environment                                                 |
| 3     | staging     | Stable version of service. Code on staging branch will be deployed to QE cluster            |
| 4     | development | Experimental version of service. Code on development branch will be deployed to DEV cluster |
| 5     | feature     | Developer's work on feature, enhancement and bug fixing                                     |
| 6     | promote     | Create from a stable commit (tag) for deployment                                            |

## Merge strategy

| Source branch | Target branch | Merge type    |
|---------------|---------------|---------------|
| feature       | development   | Squash commit |
| hot-fix       | staging       | Squash commit |
| hot-fix       | master        | Squash commit |
| promote       | staging       | Merge (no ff) |
| staging       | master        | Rebase        |


## Deployment process
- Bug fixing, enhancement and new feature will be kept until end of sprint or a release date
- If have urgent / ad hoc testing on QE cluster, a new branch will be created from `staging`. Developer must merge this branch back to both `staging` and `development` branch. 
- Bugs are found on QE cluster will be reported for DEV team by QA team, and be fixed on next release
- Developer will mark their service stable version by a git tag, following schematic version for deployment
- When QA team want a release version of service, DevOps team will create a new branch `promote` from the tag they marked before. Then, they need to create a PR from this `promote` branch to `staging` branch. After the PR is merged into `staging` branch, the pipeline will be triggered to deploy `staging` branch to QE cluster
- If have issue with new version of service or urgent / ad hoc testing, DevOps team will need to revert the code on `staging` branch. After reverting, pipeline will automatically deploy to QE cluster
- After each release, need to sync the code between `master`, `staging` and `development` branch

# Github flow

![github-flow](/.attachments/github-flow.png)

## Git branch 

| Index | Branch      | Note                                                                                        |
|-------|-------------|---------------------------------------------------------------------------------------------|
| 1     | master      | Code on master branch will be built to package for installer                                |
| 2     | hot-fix     | Branch to fix bug on production environment                                                 |
| 3     | development | Experimental version of service. Code on development branch will be deployed to DEV cluster |
| 4     | feature     | Developer's work on feature, enhancement and bug fixing                                     |

## Merge strategy

| Source branch   | Target branch | Merge type    |
|-----------------|---------------|---------------|
| feature         | development   | Squash commit |
| staging-hot-fix | development   | Merge (no ff) |
| hot-fix         | master        | Squash commit |
| development     | master        | Merge (no ff) |

## Deployment process
- Bug fixing, enhancement and new feature will be kept until end of sprint or a release date
- No urgent / ad hoc testing on QE cluster to avoid service communication conflict block QA's work
- Bugs are found on QE cluster will be reported for DEV team by QA team, and be fixed on next release
- Developer will mark their service stable version by a git tag, following schematic version for deployment
- When QA team want a release version of service, DEV team will create or update the `staging` tag point to newest stable commit. After that, pipeline will automatically be triggered to deploy the commit with `staging` tag to QE cluster
- If have issue with new version of service, DEV team will manually trigger pipeline on a specific tag to rollback that version

## Staging sanity

![staging-sanity](/.attachments/staging-sanity.png)

- If you need to modify some code for QA environment, first create a new branch from last `staging` tag commit
- On this branch, add your change and commit
- When you satisfied with the change, redeploy it to QA environment by adding the tag `staging` on the commit you want
- You can merge the staging bug fixing branch back to `development` without any affect to QA environment

# Schematic version

Given a version number MAJOR.MINOR.PATCH, increment the:
- MAJOR version when you make incompatible API changes
- MINOR version when you add functionality in a backward compatible manner
- PATCH version when you make backward compatible bug fixes

Additional labels for pre-release and build metadata are available as extensions to the MAJOR.MINOR.PATCH format.

| Tag format           | Description        | Example                         |
|----------------------|--------------------|---------------------------------|
| MAJOR.MINOR.PATCH    | Production release | 1.0.0, 1.0.1, 2.1.3             |
| MAJOR.MINOR.PATCH-rc | Release candidate  | 1.0.0-rc1, 1.0.1-rc1, 2.1.3-rc2 |