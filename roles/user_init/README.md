container_init
==============

A role for managing users using a local database.

Requirements
------------

This role depends on the community.general.read_csv ansible task.

Additionally, a CSV file in the passwd database format is required (see passwd(5) for more details on the format). 

Fields in the passwd.csv file should be seperated by ':'. Optional fields should be left blank rather then removed. The following is an ordered list of the fields:

- `name`: Name of the user.
- `password`: Legacy method of storing user password. Value should always be 'x' or '*'.
- `uid`: UID of user.
- `gid`: *Optional* GID of user's primary group.
- `comment`: *Optional* The GECOS field of the user. Defaults to "`name` Service Account".
- `home`: *Optional* Path to the users home. Defaults to "/home/`name`".
- `shell`: *Optional* The users login shell. Defaults to "/bin/bash".

Role Variables
--------------

- `user_init_name`: User account to create or update.
- `user_init_database`: *Optional* Path to local user database. Defaults to `vars/passwd.csv`.

Example Playbooks
-----------------

With the following database and playbook `user1` will be created on `servers` hosts with an UID of 4141 and a primary group with a GID of 4141.

vars/passwd.csv:
``` csv
user1:x:4141::::
user2:x:4242:4242:User2 Account:/var/lib/user2:/bin/bash
```

playbook.yml:
``` yaml
- hosts: servers
  become: yes

  roles:
    - role: user_init
      user: user1
```
