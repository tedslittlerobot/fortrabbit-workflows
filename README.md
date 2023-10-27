Fortrabbit Workflows for Github
===============================

A series of useful workflows for use with Github Actions and Fortrabbit [Fortrabbit](https://fortrabbit.com/).

# Setup

Create an application-specific SSH key pair:

```bash
ssh-keygen -t rsa -b 4096 -m pem -C deploy@github.com -f /tmp/key-github
```

Copy the public key (on macOS, `cat /tmp/key-github.pub | pbcopy` will copy this to the clipboard for you)

On your Fortrabbit dashboard, go to `Settings -> Show all settings -> App-only keys`.

Paste in your new public key and save.

Then in your Github repository, go to `Repository Settings -> Secrets and Variables -> Actions`, and add a new `Repository Secret`, and call it something sensible - such as `STAGING_SSH_KEY`.

Copy your new private key (on macOS, `cat /tmp/key-github | pbcopy` will copy this to the clipboard for you), and paste it into the secret and save (Ensure that it has the `---- BEGIN/END ----` sections)

**Now you are all set up!**

See [webfactory/ssh-agent](https://github.com/webfactory/ssh-agent) for full details of the SSH Agent Action.

# Workflows

With the above setup done, you can simply add any of the following workflows!

A note, it is not advisable to run any of these workflows in parallel with each other, as Fortrabbit applications do not play well when being contacted in parallel!

## Deploy

> tedslittlerobot/fortrabbit-workflows/.github/workflows/deploy.yml@v1

This will deploy your project to Fortrabbit using the git deployment method. Under the hood, this creates a new blank repository and force pushes it to the git remote.

The example here will respond to pushes to the develop branch, and will deploy the current (develop) branch to it.

```yaml
name: Deploy Staging

on:
  push:
    branches:
      - develop

jobs:
  deploy:
    uses: tedslittlerobot/fortrabbit-workflows/.github/workflows/deploy.yml@v1
    with:
      remote: my-website-stg@deploy.eu2.frbit.com:my-website-stg.git
    secrets:
      deploy_key: "${{ secrets.STAGING_SSH_KEY }}"
```

If you have JS and CSS assets to build, you can enable this by specifying the node version you wish to use. This will automatically run `npm install`, and `npm run build`, and expects the built files to be in the `public/build` directory (see below for details and overrides).

```yaml
name: Deploy Staging

on:
  push:
    branches:
      - develop

jobs:
  deploy:
    uses: tedslittlerobot/fortrabbit-workflows/.github/workflows/deploy.yml@v1
    with:
      node_version: 18.x
      remote: my-website-stg@deploy.eu2.frbit.com:my-website-stg.git
    secrets:
      deploy_key: "${{ secrets.STAGING_SSH_KEY }}"
```

### Options

- `remote` (required): The git remote URL supplied by fortrabbit
- `node_version` (optional): The node version to use when building assets. If this is not set, assets will not be built.. See [actions/setup-node](https://github.com/actions/setup-node) for more details.
- `npm_build_cmd` (optional, default: `npm run build`): The command to use to build your assets. You do not need to specify `npm install` - this is run automatically before your build command.
- `public_build_path` (optional, default: `public/build`): You must specify where your assets build _to_. These files should usually be git ignored in you repository. This option allows us to force-add the git ignored build files to the deployment.
- `pre_deploy_script` (optional, default: none): You can specify a script file to run before the deployment. Scripts can do any final changes to the project before the final commit and deploy. A recommended value is `.github/hooks/pre-deploy.bash`
- `fortrabbit_branch` (optional, default: main): The branch to push to on the fortrabbit remote. Odds are this doesn't need overriding.
- `git_email` (optional, default: deploy@github.com): The git email to use for commits. Odds are this doesn't need overriding.
- `git_name` (optional, default: Github-Fortrabbit Deployer): The git username to use for commits. Odds are this doesn't need overriding.

## Artisan

> tedslittlerobot/fortrabbit-workflows/.github/workflows/artisan.yml@v1

This will run an artisan command on a deployment using an SSH connection.

```yaml
name: Deploy Staging

on:
  push:
    branches:
      - develop

jobs:
  deploy: # ... deploy job would go here ...
  migrate:
    uses: tedslittlerobot/fortrabbit-workflows/.github/workflows/artisan.yml@v1
    needs:
      - deploy
    with:
      command: migrate
      flags: --force
      remote: my-website-stg@deploy.eu2.frbit.com
    secrets:
      deploy_key: "${{ secrets.STAGING_SSH_KEY }}"
```

### Options

- `remote` (required): The remote SSH URL supplied by fortrabbit
- `command` (required): The artisan command to run. Do not put `php artisan` or `artisan` in front of it.
- `flags` (optional): Any flags you wish to add to the command. You could put them in with the main command as well

## Execute

> tedslittlerobot/fortrabbit-workflows/.github/workflows/execute.yml@v1

This will run an arbitrary command on a deployment using an SSH connection.

```yaml
name: Deploy Staging

on:
  push:
    branches:
      - develop

jobs:
  deploy: # ... deploy job would go here ...
  migrate:
    uses: tedslittlerobot/fortrabbit-workflows/.github/workflows/execute.yml@v1
    needs:
      - deploy
    with:
      command: ls -al
      remote: my-website-stg@deploy.eu2.frbit.com
    secrets:
      deploy_key: "${{ secrets.STAGING_SSH_KEY }}"
```

### Options

- `remote` (required): The remote SSH URL supplied by fortrabbit
- `command` (required): The artisan command to run. Do not put `php artisan` or `artisan` in front of it.

# Action

> tedslittlerobot/fortrabbit-workflows@v1

The `deploy` workflow is also available as an action to use in any of your steps.

# Links / Notes / References

Heavily based on (this blog post on the Fortrabbit website)[https://blog.fortrabbit.com/how-to-use-github-actions].

Uses [webfactory/ssh-agent](https://github.com/webfactory/ssh-agent) under the hood.

# FAQ's

## I am seeing `repository is busy: probably deployment in progress (prep)`

Either you failed to heed the warning about parallelisation above, or some other service is also running something against your fortrabbit application.

Do not run these commands in parallel. Github actions run in parallel by default - provide a `needs` block as in the examples above to chain them in series.
