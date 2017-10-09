#### ansible issue 31409

this repository contains these branches to point out what could be wrong with
ansible importing roles conditionally [#31409](https://github.com/ansible/ansible/issues/31409).

##### why conditional roles?

We have one big playbook `site.yml` for all our hosts which includes all roles
we might ever need for these hosts. A host array specifies which roles should
be run for a specific host.

##### why conditional includes?

Some of our roles might be used either standalone or as a dependency. Think of
databases: one server only runs a dbms, e.g. as a backend for a load balanced
setup, another server runs a webserver with a cms which requires a database on
the same host.

To avoid multiple runs of the same tasks, like:
1. install mariadb
2. setup a mysql root user
3. write mysql server config
4. start mysql service
5. remove test databases and test users
6. do other stuff
7. create a `$database`
8. grant privileges to `$user`
9. import schemas

We added a task after step 6 that adds its `role_name` to `roles_completed` and
another tasks before step 1 that skips steps 1 to 6 if `roles_completed` already
contains the `role_name`.

##### branches

###### `master`

In the role `dependency-or-standalone` the task `include_tasks` is used to make
ansible run the tasks in `dependency-or-standalone/tasks/main_run.yml`.

We never see the debug output "`in dependency-or-standalone`".

###### `ansible-2.3`

Here `include` is used instead of `include_tasks`, ansible 2.3 style.

The debug output "`in dependency-or-standalone`" is shown correctly.

###### `ansible-2.4-import_tasks`

Instead of `include_tasks` we use `import_tasks`. The debug output is shown
correctly but using a static include has other unwanted side effects.

###### `ansible-2.4-include_role`

The roles are the same as in `master` branch, but instead of
```yaml
  roles:
    - role: ...
```
we use
```yaml
  tasks:
    - include_role: name=...
```
The debug output is shown correctly, but in more complex playbooks other tasks
get skipped. Also roles might run even though they shouldn't.

###### `ansible-2.4-removed-role`

The roles are the same as in `master` branch, but we removed `role1` from the
playbook. I suspect that ansible skips adding its conditions to `dependency-or-standalone`
here.

##### conclusion?

afaics ansible merges conditions and parent conditions in a wrong way. I suspect
the tasks in `dependency-or-standalone` are not run because `dependency-or-standalone` has
at least one parent role that is forbidden to run (`"role1" in server_roles`). The condition
probably gets inherited by `dependency-or-standalone`.

This bug might be related to [#23733](https://github.com/ansible/ansible/issues/23733).
