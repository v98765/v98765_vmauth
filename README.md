Role Name: Victoriamertics vmauth
=========

Deploy and configure [vmauth](https://docs.victoriametrics.com/vmauth.html)

Requirements
------------

Ansible 2.10

Role Variables
--------------

Name | Default Value
---|---|---
`vmauth_version` | 1.69.0
`vmauth_system_user` | vmauth
`vmauth_system_group` | vmauth
`vmauth_config_dir` | /etc/vmauth
`vmauth_install_dir` | /usr/local/bin
`vmauth_repo_dir` | /var/tmp/archive
`vmauth_reloadauthkey` | RkU7FhjinrCv7N7f
`vmauth_users` | check default/main.yml


Read this [https://docs.victoriametrics.com/#environment-variables](https://docs.victoriametrics.com/#environment-variables) аnd set more vars, put it into `templates/vmauth.j2`

```sh
mkdir -p /var/tmp/archive
cd /var/tmp/archive
wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.69.0/vmutils-amd64-v1.69.0.tar.gz
```

Example Playbook
----------------

```yaml
---
- hosts: vmdb
  gather_facts: true
  connection: ssh
  roles:
    - v98765_vmauth

```
host_vars/vmdb.yml
```yaml
vmauth_reloadauthkey: "new_password"
- username: "new_users1"
  password: "new_pass1"
  url_prefix: "http://localhost:8428"

- username: "new_user2"
  password: "new_pass2"
  url_map:
  - src_paths:
    - "/api/v1/query.*"
    - "/api/v1/series.*"
    - "/api/v1/label.+"
    - "/vmui.*"
    url_prefix: "http://localhost:8428"
```
License
-------

BSD
