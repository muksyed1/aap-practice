# aap-practice

Practice repo for Ansible Automation Platform (AAP). It does two things:

1. `hello` role: prints "Hello World". AAP runs it through a job template.
2. `deploy` role: creates the AAP project, inventory, host and job template from code, so nothing is clicked together by hand.

## Layout

| Path | Purpose |
|---|---|
| `main.yml` | Playbook the AAP job template runs. Calls the `hello` role. |
| `hello/` | Role that prints "Hello World". |
| `configure.yml` | Playbook you run from your own machine. Calls the `deploy` role. |
| `deploy/` | Role that creates the AAP objects with the `awx.awx` modules. |
| `inventory.yml` | Local inventory for running `main.yml` on your machine. AAP does not read it. |
| `requirements.yml` | Ansible collections needed (`awx.awx`). |

`configure.yml` creates the objects, and the job template then runs `main.yml`. They are separate on purpose, so the template never tries to create itself.

## Prerequisites

- Ansible (ansible-core) installed, for example in a Python virtualenv.
- Access to an AAP instance and a user that can create projects, inventories and templates.
- The collection: `ansible-galaxy collection install -r requirements.yml`

## Connect to AAP

The playbook reads its connection settings from a file outside this repo. Nothing secret is committed. `.gitignore` excludes `*.cfg` as a safety net.

1. In AAP, create an OAuth application (Access Management, then OAuth Applications), then create a token for your user with **Write** scope. Use the access token, not the refresh token.
2. Create `~/tower_cli.cfg` and restrict it:

   ```
   host = https://<your-aap-host>
   oauth_token = <access token>
   verify_ssl = true
   ```

   ```
   chmod 600 ~/tower_cli.cfg
   ```

Use only the base URL for `host`, with no `/api/...` path. On AAP 2.5 and later the controller API sits behind the platform gateway at `/api/controller/`, and `awx.awx` defaults to `/api/`. `configure.yml` handles this by setting `CONTROLLER_OPTIONAL_API_URLPATTERN_PREFIX`.

## Run

Create the AAP objects:

```
ansible-playbook configure.yml --tags deploy
```

The role is tagged `never` and `deploy`, so it runs only when you ask for it with `--tags deploy`.

Then launch the job template in AAP (default name `aap-template-deploy-1`). The output should show `"msg": "Hello World"`.

To run the hello role locally instead:

```
ansible-playbook -i inventory.yml main.yml
```

## Settings

Defaults are in `deploy/defaults/main.yml` (organization, project, inventory, template names and the git URL). Override them with `-e`, for example:

```
ansible-playbook configure.yml --tags deploy -e template_name=my-template
```

## Notes

- `awx.awx` is the upstream AWX collection. It worked for the objects used here, but `ansible.controller` from Automation Hub is the supported collection for AAP.
- On a Red Hat Developer Sandbox, AAP can be idled. If the console shows "Application is not available", check `idle_aap` on the `AnsibleAutomationPlatform` resource and set it to `false`.
