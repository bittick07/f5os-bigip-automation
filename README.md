# F5OS and BIG-IP automation starter

This repository has two intentionally safe workflows: one for F5OS and one for BIG-IP. Both collect facts only and make **no configuration changes**. Keeping them separate means each platform can have its own AWX job, credentials, schedules, approvals, and future change playbooks.

## Repository layout

- `inventory/hosts.example.yml` — sample device inventory. Real inventory is ignored by Git.
- `group_vars/` — platform connection settings; passwords are not stored here.
- `playbooks/f5os_connectivity.yml` — read-only F5OS smoke test.
- `playbooks/bigip_connectivity.yml` — read-only BIG-IP smoke test.
- `playbooks/bigip_virtual_server_example.yml` — a deliberately incomplete change example.
- `requirements.yml` and `execution-environment.yml` — F5 Ansible dependencies for AWX.

## 1. Put this in GitLab

1. Create a **private** GitLab project, for example `network/f5-automation`.
2. In this folder, initialize Git, commit these files, and push the default branch.
3. Add real inventory only in AWX, or copy the example to `inventory/hosts.yml` locally. Never commit credentials, tokens, backups, or production device exports.

The included pipeline performs syntax checks on each branch and merge request. It never contacts your devices.

## 2. Build an AWX execution environment

The default AWX execution environment may not include the F5 collections. Build one from `execution-environment.yml`, publish it to a registry AWX can pull from, then register it at **Administration → Execution Environments**. This definition installs `f5networks.f5os`, `f5networks.f5_modules`, and `ansible.netcommon`.

For a first lab test, you can also use an execution environment that already has those three collections installed.

## 3. Configure AWX

Create these objects in this order:

1. **Credential** → type **Machine**. Set the F5 management username and password. Attach it to both job templates. Use an account with only the privileges the job needs.
2. **Inventory** → name it `F5 Lab`. Add groups `f5os` and `bigip`; add each device as a host and set its `ansible_host` variable to its management IP or DNS name. Do not put passwords in host variables.
3. **Project** → source control type **Git**, with your GitLab clone URL and a GitLab Personal Access Token or deploy token credential. Set the branch to `main`.
4. **Job Template** → create `F5OS connectivity check`; select the `f5os` inventory group, project, F5OS machine credential, custom execution environment, and `playbooks/f5os_connectivity.yml`. Turn on **Update Revision on Launch**.
5. **Job Template** → create `BIG-IP connectivity check`; select the `bigip` inventory group, project, BIG-IP machine credential, custom execution environment, and `playbooks/bigip_connectivity.yml`. Turn on **Update Revision on Launch**.

Use separate AWX credentials even if the usernames happen to match. This makes it easy to give the F5OS and BIG-IP jobs only the permissions they require.

Before the first production connection, keep certificate validation enabled. If a lab device uses a self-signed certificate, trust its CA in the execution environment; only use `ansible_httpapi_validate_certs: false` temporarily in a non-production lab.

## 4. Run the safe test

Launch each connectivity job in AWX. A successful job confirms AWX can clone the repository, has the F5 collections, can reach that platform's management interface, and can authenticate.

## What to do next

After the connectivity job succeeds, choose one small, reversible task: creating a DNS/NTP entry on F5OS or managing one non-production BIG-IP virtual server. Put the desired state in a new playbook, require a GitLab merge request, and run it from a separate AWX job template with a limited credential.

F5OS uses the `f5networks.f5os` HTTPAPI connection; BIG-IP uses `f5networks.f5_modules`. Refer to the [F5OS collection guide](https://clouddocs.f5.com/products/orchestration/ansible/devel/f5os/f5os.html) and [F5 Ansible collection overview](https://clouddocs.f5.com/products/orchestration/ansible/devel/overview.html) when adding modules.
