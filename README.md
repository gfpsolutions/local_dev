# Quickstart

If you have not launched a Postgres VM, you must first do that before launching the ansible script for the Odoo VM(s). You must have ansible, multipas and openssl installed in your machine.

## Odoo VM

Launch the VM

```
multipass launch 20.04 -d 10G -n clientdb
```

Add a host rule for quality of life

```bash
sudo sh -c "echo '192.168.64.10    clientdb_dev.gfp' >> /etc/hosts"
```

Input your ssh key for ansible ease of use

```bash
multipass shell clientdb
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDX388XiAaRljheka35mrqzbKTszotIrxnsFJgvhHT2t1qQ05318JzuV9GQCxCD2qUBMvHPrrJVbDMejpU6pPR1A08eeUy9A/9cgTsGkvw1COwbaSOAsKwcpKwGubAcwKwc+qPKXALW6oQ2rn7TGyX0SL5RSaEz11il/GLwYUVQsjxKentpmT/AeXKQyuA75v2z4+e21Pp/CKLWXVzYXs8byHZA75VQoQtukUlJ9qay6T/AyZUDnhEVcg2GdMXQ9Mh19x5w5i8Itt76qzGRjj2REunwiSIfT/g+7DYB0pe7Wn7xDCbOfKN2j8YopP3ZBp7U954uUW51wJAuXpWzgpZt gfp@gfp.local" >> /home/ubuntu/.ssh/authorized_keys
sudo service sshd restart
sudo service ssh restart
```

Configure Ansible script

```yml
#Add the host on the hosts file under the production tag
[production]
clientdb_dev.gfp

#create a new file under the host_vars directory, named after the host (e.g. clientdb_dev.gfp). Required: odoo_db, odoo_version, addons_path
odoo_db: clientdb
odoo_version: 18.0
custom_module: base_client
addons_path: /home/odoo/src/enterprise,/home/odoo/src/odoo/addons,/home/odoo/src/custom_modules/client
add_python_lib: untangle paramiko
```

Run Ansible script

```bash
ansible-playbook -i hosts local_dev.yml
```

or

```bash
ansible-playbook -i hosts local_dev.yml -l "clientdb_dev.gfp"
```

mount the custom modules directory to the VM

```
multipass mount /Users/gfp/Documents/git/custom_modules clientdb:/home/odoo/src/custom_modules
```

Run Odoo

```bash
odoo@clientdb:~$ odoo_run
2020-01-09 09:19:23,408 18654 INFO ? odoo: Odoo version 16.0
2020-01-09 09:19:23,408 18654 INFO ? odoo: addons paths: ['/home/odoo/.local/share/Odoo/addons/16.0', '/home/odoo/src/enterprise', '/home/odoo/src/odoo/addons', '/home/odoo/src/custom_modules/clientdb', '/home/odoo/src/odoo/odoo/addons']
2020-01-09 09:19:23,408 18654 INFO ? odoo: database: default@default:default
```

## Postgres VM

```bash
multipass launch 24.04 -d 20G -n postgres
```

Add a host rule for quality of life

```bash
sudo sh -c "echo '192.168.64.12    postgres.gfp' >> /etc/hosts"
```

Input your ssh key for ansible ease of use

```bash
multipass shell postgres
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDX388XiAaRljheka35mrqzbKTszotIrxnsFJgvhHT2t1qQ05318JzuV9GQCxCD2qUBMvHPrrJVbDMejpU6pPR1A08eeUy9A/9cgTsGkvw1COwbaSOAsKwcpKwGubAcwKwc+qPKXALW6oQ2rn7TGyX0SL5RSaEz11il/GLwYUVQsjxKentpmT/AeXKQyuA75v2z4+e21Pp/CKLWXVzYXs8byHZA75VQoQtukUlJ9qay6T/AyZUDnhEVcg2GdMXQ9Mh19x5w5i8Itt76qzGRjj2REunwiSIfT/g+7DYB0pe7Wn7xDCbOfKN2j8YopP3ZBp7U954uUW51wJAuXpWzgpZt gfp@gfp.local" >> /home/ubuntu/.ssh/authorized_keys
sudo service sshd restart
sudo service ssh restart
```

Configure Ansible script

```yml
#Add the host on the hosts file under the production tag
[postgres]
postgres.gfp

#create a new file under the host_vars directory, named after the host (e.g. postgres.gfp)
postgres_user: odoo
```

Run Ansible script

```bash
ansible-playbook -i hosts postgres.yml
```

Uncomment `listen_addresses = '*'` in `/etc/postgresql/<version>/main/postgresql.conf`

Add line `host    all    all    xxx.xxx.xx.0/24    trust` to `/etc/postgresql/16/main/pg_hba.conf` where the IP is the IP of your machine in your local network. You can also add md5 and password support for better security

restart postgres

```bash
sudo server postgresql restart
```

Ensure the following ansible variables are set to the correct values in `/roles/odoo/defaults/main.yml`

```yml
odoo_postgres: postgres.gfp
odoo_postgres_ip: xxx.xxx.xx.xx
odoo_postgres_user: odoo
```

## Restore a database

```bash
createdb -h postgres.gfp -U odoo clientdb
```

```bash
pg_restore -h postgres.gfp -U odoo --no-acl --no-owner -d clientdb ./clientdb.dump
```

or

```bash
psql -h postgres.gfp -U odoo clientdb < dump.sql
```

## Neutralize a database

```bash
psql -h postgres.gfp -U odoo -d clientdb -c "UPDATE ir_cron SET active = 'f';"
psql -h postgres.gfp -U odoo -d clientdb -c "DELETE FROM ir_mail_server where id > 0;"
psql -h postgres.gfp -U odoo -d clientdb -c "DELETE FROM fetchmail_server where id > 0;"
psql -h postgres.gfp -U odoo -d clientdb -c "UPDATE ir_config_parameter SET value = 'http://client_db.gfp' WHERE key = 'web.base.url';"
psql -h postgres.gfp -U odoo -d clientdb -c "DELETE FROM ir_config_parameter WHERE key = 'database.enterprise_code';"
```

_Remember_ to disable external connections by disabling API keys, etc.

```bash
psql -h postgres.gfp -U odoo -d clientdb -c "UPDATE res_company SET some_api_key = false";'
```

Resetting the admin's password

```bash
psql -h postgres.gfp -U odoo -d clientdb -c "UPDATE res_users SET password='admin' WHERE login = 'admin';"
```

or

```bash
psql -h postgres.gfp -U odoo -d clientdb -c "UPDATE res_users SET password='admin' WHERE id = 2;"
```

# Misc

## Adding more disk space to a VM

```bash
multipass stop clientdb
multipass set local.clientdb.disk=10G
multipass start clientdb
```
