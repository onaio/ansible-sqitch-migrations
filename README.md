# Ansible Sqitch Migrations [![Build Status](https://github.com/onaio/ansible-sqitch-migrations/workflows/CI/badge.svg)](https://github.com/onaio/ansible-sqitch-migrations/actions?query=workflow%3ACI)

Use this role to set up a database using [sqitch](https://sqitch.org/) migrations.

The database setup process is in three stages:

1. Set up
   - One or more repositories that contain database migrations are cloned.  Both private and public git repositories can be cloned.  Private repositories require that you have the relevant ssh key added to ssh-agent.
   - Create a [pgpass](https://www.postgresql.org/docs/9.3/libpq-pgpass.html) file to make authentication against the DB easy and standard throughout the role
   - Execute any SQL queries that need to be run to set up the database.  These may be things such as creating databases and database users, creating postgres extensions etc.
2. Sqitch migrations - the sqitch migrations are deployed and verified.  The role is able to support running multiple sqitch migration plans, for different databases.
3. Set up periodic jobs - Ensure any SQL queries that need to be run periodically are set up a cron jobs.  These may be queries such as a daily data dump, refreshing materialized views, etc.

**Note**:

- At this moment, only PostgreSQL is supported.
- This role does not concern itself with installing database systems.

## Role Variables

Some of the more important variables are briefly described below.  You can see all variables by looking at the [defaults/main.yml](defaults/main.yml) file.

### sqitch_migrations_system_user

This is the user which is used while running the role.  The default is "nomad" - you know, because they migrate :)

```yml
sqitch_migrations_system_user: nomad
```

### sqitch_migrations_home

This is a variable defined for convenience as it is used very often.  It defines the home directory of the `sqitch_migrations_system_user`.

```yml
sqitch_migrations_home: "/home/{{ sqitch_migrations_system_user }}"
```

### sqitch_migrations_system_dependencies

This variable defines a list packages that should be installed as part of setting up the role.

```yml
# system-wide dependencies
sqitch_migrations_system_dependencies:
  - git
```

### sqitch_migrations_log_path

This is the directory in which logs will be stored.

```yml
sqitch_migrations_log_path: "/var/log/sqitch_migrations"
```

### sqitch_migrations_repositories

A list of git repositories to clone, the database migrations we want to run are located here.

```yml
sqitch_migrations_repositories:
  - url: git@github.com:onaio/data-solutions.git
    destination: "{{ sqitch_migrations_home }}/data-solutions"
    version: master
```

### sqitch_migrations_db_connections

An object containing database connections to use when running the role.

```yml
sqitch_migrations_db_connections:
  db_admin_user:  # used to identify this particular connection
    user: db_username
    password: hunter2
    host: localhost
    port: 5432
    database: dbname
  db_other_user:  # used to identify this particular connection
    user: othe_ruser
    password: hunter2
    host: localhost
    port: 5432
    database: dbname2
```

### sqitch_migrations_sql_for_setup

An object describing files containing SQL queries that should be run BEFORE database migrations.

```yml
sqitch_migrations_sql_for_setup:
  db_admin_user:  # used to identify the db connection to use
    - absolute-path-to-file-containing-sql
    - /tmp/some-file.sql
```

### sqitch_migrations_sqitch_plans

An object containing the actual sqitch database migrations to be run.

```yml
sqitch_migrations_sqitch_plans:
  db_admin_user:  # used to identify the db connection to use
    - directory: "{{ sqitch_migrations_home }}/data-solutions/examples/sqitch_test"
    - directory: "{{ sqitch_migrations_home }}/data-solutions/1-utils"
      registry: sqitch_test
      extra_args: --verify --set schema=test
      engine: pg
```

As you can tell, it is an object/dictionary where each entry contains a list of other objects.  The structure of these inner objects is as follows:

- **directory**: absolute path to a directory that contains a sqitch plan
- **registry**: (optional) the sqitch registry to use, default is "sqitch"
- **extra_args**: (optional) extra args for sqitch deploy command, default is "--verify"
- **engine**: (optional) database engine to use, default is pg

### sqitch_migrations_sql_jobs

An object containing SQL queries that should be executed periodically.

```yml
sqitch_migrations_sql_jobs:
  db_admin_user:  # used to identify the db connection to use
    - file: "/tmp/test.sql"  # absolute path to file containing sql to be run periodically
      schema: test  # database schema for this to be run in
      minute: "*/5"  # cron config for every 5 minutes
```

As you can tell, it is an object/dictionary where each entry contains a list of other objects.  The structure of these inner objects is as follows:

- **file**: absolute path to the sql file
- **schema**: (Optional) the database schema to use, default is "public"
- **month**: (Optional) cron config for month, default is "*"
- **weekday**: (Optional) cron config for weekday, default is "*"
- **day**: (Optional) cron config for day, default is "*"
- **hour**: (Optional) cron config for hour, default is "*"
- **minute**: (Optional) cron config for minute, default is "*"

## Dependencies

- **onaio.sqitch**: for installing sqitch
- **adamyala.ansible-role-clone-repo**: for cloning git repositories

You can install these by running the following ansible command:

```sh
ansible-galaxy install -r requirements.yml
```

## Example Playbook

```yml
- hosts: servers
  roles:
    - role: ansible-sqitch-migrations
      vars:
        - sqitch_migrations_system_dependencies:
            - git
        - sqitch_migrations_repositories:
            - url: git@github.com:onaio/data-solutions.git
              destination: "{{ sqitch_migrations_home }}/data-solutions"
              version: master
            - url: git@github.com:OpenSRP/opensrp-reveal-datawarehouse.git
              destination: "{{ sqitch_migrations_home }}/reveal-datawarehouse"
              version: master
        - sqitch_migrations_db_connections:
            db_admin:
              user: admin
              password: hunter2
              host: localhost
              port: 5432
              database: productiondb
            db_user:
              user: regular
              password: hunter2
              host: localhost
              port: 5432
              database: productiondb
        - sqitch_migrations_sql_for_setup:
            db_admin:
              - "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/setup/extensions.sql"
        - sqitch_migrations_sqitch_plans:
            db_admin:
              - directory: "{{ sqitch_migrations_home }}/data-solutions/examples/sqitch_test"
            db_user:
              - directory: "{{ sqitch_migrations_home }}/data-solutions/1-utils"
                registry: sqitch_test
                extra_args: --verify --set schema=test
                engine: pg
              - directory: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/1-common-migrations/1-transaction-tables"
                registry: sqitch_test
                extra_args: --verify --set schema=test
              - directory: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/1-common-migrations/2-raw-tables"
                registry: sqitch_test
                extra_args: --verify --set schema=test
              - directory: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/migrations/1-raw_tables"
                registry: sqitch_test
                extra_args: --verify --set schema=test
              - directory: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/migrations/2-transaction_tables"
                registry: sqitch_test
                extra_args: --verify --set schema=test
              - directory: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/migrations/3-views"
                registry: sqitch_test
                extra_args: --verify --set schema=test
              - directory: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/migrations/4-FI/1-Thailand-2019"
                registry: sqitch_test
                extra_args: --verify --set schema=test
              - directory: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/migrations/5-IRS/1-generic"
                registry: sqitch_test
                extra_args: --verify --set schema=test
              - directory: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/migrations/5-IRS/2-Zambia-2019"
                registry: sqitch_test
                extra_args: --verify --set schema=test
              - directory: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/migrations/5-IRS/3-Namibia-2019"
                registry: sqitch_test
                extra_args: --verify --set schema=test
        - sqitch_migrations_sql_jobs:
            db_user:
              - file: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/jobs/materialized-views/refresh_jurisdictions_materialized_view.sql"
                schema: test
                minute: "*"
              - file: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/jobs/materialized-views/refresh_plans_materialzied_view.sql"
                schema: test
                minute: "0"
                hour: "*/2"
            db_admin:
              - file: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/jobs/materialized-views/refresh_reporting_lag.sql"
                schema: test
                hour: "*/1"
              - file: "{{ sqitch_migrations_home }}/data-solutions/OpenSRP/2-reveal/jobs/materialized-views/refresh_reporting_time.sql"
                schema: test
```

## Testing

This project uses molecule for testing.

Start by creating a virtual environment and installing python packages

```sh
pip install -r requirements.txt
```

Then to run the full test sequence:

```sh
tox
```

## License

Apache 2

## Authors

[Ona Engineering](https://ona.io)
