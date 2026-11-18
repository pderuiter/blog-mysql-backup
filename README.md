# blog-mysql-backup

Companion code for the blog post
[MySQL Backup with Ansible](https://blog.bsdserver.nl/mysql-backup-with-ansible/).

It builds the setup from the post. It creates two Ubuntu 24.04 VMs (a MySQL server and a backup server) and installs
MySQL with a read-only backup user. Then it runs the post's backup playbook, packaged as roles: `mysqldump | gzip` to
`/tmp/backup` on the MySQL server, then `rsync` over SSH (the `synchronize` module) to the backup server.

## What's in here

| Path | What it does |
| --- | --- |
| `terraform/` | Two Proxmox VE VMs from the Ubuntu 24.04 cloud image (bpg/proxmox provider). Outputs an Ansible inventory. |
| `ansible/roles/mysql_server` | Installs `mysql-server`, creates the database and the backup user (`SELECT, LOCK TABLES, SHOW VIEW, TRIGGER, EVENT, PROCESS`). |
| `ansible/roles/mysql_backup` | The post's playbook. On the MySQL server it creates an SSH key and runs the `mysqldump` pipeline with `set -o pipefail`. The `sync` task file pushes `/tmp/backup/` to the backup server with `ansible.posix.synchronize`, delegated to the MySQL server. |
| `ansible/roles/backup_target` | Creates the backup user and destination directory on the backup server and authorizes the MySQL server's key. |
| `ansible/site.yml` | Runs all of the above in the same order as the post. |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VMs)
- Ansible >= 2.16 (ansible-core) and the collections in `ansible/requirements.yml`
- SSH access with sudo to both machines (the Terraform cloud-init user is `ubuntu`)

## Quick start

```bash
# 1. Machines (optional; any two Ubuntu 24.04 hosts work)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in endpoint, token, node, IPs, SSH key
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configuration
cd ../ansible
ansible-galaxy collection install -r requirements.yml
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml   # set vault_mysql_password
ansible-playbook site.yml --ask-vault-pass
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and put your hosts in the `mysql`
and `backup` groups.

Run `site.yml` again whenever you want a fresh backup. The dump is reproducible (`--skip-dump-date`, `gzip -n`),
so the backup file and the rsync copy are only reported as changed when the database actually changed.

## Variables

| Variable | Default | Meaning (same names as the post) |
| --- | --- | --- |
| `mysql_user` | `backup_ro` | MySQL user that runs `mysqldump` |
| `mysql_password` | `{{ vault_mysql_password }}` | Its password, from `vault.yml` |
| `mysql_database` | `appdb` | Database to back up |
| `backup_user` | `mysqlbackup` | User on the backup server that receives the files (set in the inventory) |
| `backup_dest` | `/home/{{ backup_user }}/mysql/` | Destination directory on the backup server |
| `ssh_key` | `~/.ssh/mysql_backup_ed25519` | Key on the MySQL server used to reach the backup server |

### Two differences from the post

- **Backup user connection:** the post sets `backup_user` as `ansible_user` of the backup server. Here the backup
  server keeps your admin user, so Ansible can create the backup account. The copy then runs against an in-memory
  alias of the same host (`add_host`, group `backup_rsync`) whose `ansible_user` is `backup_user`. The result is
  the same rsync command as in the post.
- **Key location:** synchronize runs rsync on the delegated host without privilege escalation. That is why `ssh_key` lives in the
  home directory of the user Ansible connects as, and not under `/root`.

## Restore

```bash
# on the backup server
zcat /home/mysqlbackup/mysql/appdb.sql.gz | ssh db01 'mysql appdb'
```

## Tested

On Ubuntu 24.04 (systemd containers) with ansible-core 2.16 (Ansible 9.13.0) and MySQL 8.0:

- first run completes
- second run reports `changed=0`
- after data changes, exactly the dump and the copy change
- the backup restores into an empty database with all rows
- a failing `mysqldump` fails the play, thanks to `pipefail`

`ansible-lint` (production profile), `yamllint`, `terraform fmt -check`, `terraform validate` and `tflint` are clean.
Terraform has not been applied as part of these tests.

## License

MIT
