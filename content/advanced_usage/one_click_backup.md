---
description: Use GoCD's administration interface or API to backup and restore GoCD server
keywords: GoCD server, GoCD backup, administration interface, backup API
title: Backup GoCD Server
---

# Backup GoCD Server

You can use GoCD's administration interface to perform an One-Click Backup of GoCD. You can also perform a backup [using the API](https://api.gocd.org/#backups).

## Steps to initiate backup

- After you are logged into GoCD, click on the _Admin > Backup_ link, and click the _Perform Backup_ button.

    > **Note:** GoCD will be unusable during the backup process.

- Backup time is proportional to the database and configuration size. We suggest you backup GoCD when the GoCD Server is idle. Users who are logged into the GoCD Dashboard will be redirected to a maintenance page during the backup. On completion of the backup they will be redirected to the page they were on.

### What is backed up?

The backup will be performed into the `${ARTIFACT_REPOSITORY_LOCATION}/serverBackups` directory. `${ARTIFACT_REPOSITORY_LOCATION}` for your server can be found as mentioned [here](../installation/configuring_server_details.html#artifact-repository-configuration).

The backup directory will be named `backup_{TIMESTAMP}` where the `${TIMESTAMP}` is the time when the backup was initiated.

- Database - This depends on the kind of database being used:

  - If you're using H2: This is in a zip called `db.zip`. The zip has a single DB file called `cruise.mv.db`.

  - If you're using PostgreSQL:
    - The database backup is a single file called `db.DATABASE_NAME`, where `DATABASE_NAME` is the name of the database. For instance, it could be called: `db.gocd`.
    - See [PostgreSQL documentation](https://www.postgresql.org/docs/current/backup.html) regarding backup and restoration.
    - Note that the configured [PostgreSQL database connection properties](../installation/configuring_database/connection-properties.html#gocd-database-extra-backup-command-arguments) can affect the output and contents of the backup.

  - If you're using MySQL:
    - The database backup is a single file called `db.DATABASE_NAME`, where `DATABASE_NAME` is the name of the database. For instance, it could be called: `db.gocd`.
    - See [MySQL documentation](https://dev.mysql.com/doc/refman/8.0/en/using-mysqldump.html) regarding backup and restore.
    - Note that the configured [MySQL database connection properties](../installation/configuring_database/connection-properties.html#gocd-database-extra-backup-command-arguments) can affect the output and contents of the backup.

- Configuration - This is in a zip called `config-dir.zip`. This zip contains the XML configuration, Jetty server configuration, Keystores and all other GoCD's internal configurations.
- Wrapper Configuration - This is in a zip called `wrapper-config-dir.zip`. This zip contains configuration files for startup properties for the server.
- XML Configuration Version Repo - This is in a zip called `config-repo.zip`. This zip contains the Git repository of the XML configuration file.
- GoCD version - This is a file called `version.txt`. This file contains the version of the GoCD server when the backup was initiated

### What is not backed up?

The following are not backed up as a part of the GoCD backup process. Please ensure that these are manually backed up regularly.

- Artifacts - Please refer to [this section](../faq/admin_out_of_disk_space.html#move-the-artifact-repository-to-a-new-larger-drive) to find out how to deal with artifacts
- Log Files
- Plugins - These are found at `${SERVER_INSTALLATION_DIR}/plugins/`. This contains both the external and bundled plugins.
- Addons - These are found at `${SERVER_INSTALLATION_DIR}/addons/`. This contains installed addons.

#### Strategy to backup Artifacts and Test Reporting Data

Artifacts and the Test Reporting Data keep getting new files and directories added to them. So, it is a good idea to use `rsync` to copy the contents of these two into a backup location.

*For Instance:* Lets say you have a copy of all the files till 12-02-2012 in a location. On 20-02-2012, you can do something like:

```shell
rsync -avzP ${ARTIFACT_LOCATION} ${BACKUP_LOCATION}
```

This makes sure that only the files and directories that got newly added will be synced to the `${BACKUP_LOCATION}` and not the entire contents.

### Restoring GoCD using backup

The restoration process is not automated and needs to be done manually. Please refer to the previous sections about the contents of the backup.

#### Steps to restore

- In order to restore the GoCD server from a backup, the server must first be stopped. Make sure the process is completely dead before starting the restoration.
- Choose the backup directory that you want to restore from.

    > **Note:** Please ensure that you are not restoring a GoCD backup onto a GoCD version that is older than the one that was used to perform the backup.

    > *For example:* A backup using GoCD version 19.3 cannot be restored onto GoCD version 19.2. See the file `version.txt` in the backup directory to know the version of GoCD that was used to perform the backup

- Database - This depends on the kind of database being used:

  - If you're using H2: Unzip `db.zip` and copy the file `cruise.mv.db` found inside it to the directory `${SERVER_INSTALLATION_DIR}/db/h2db`.

  - If you're using PostgreSQL: Use `psql` to restore using the database backup file (for instance `db.gocd`). This depends on how you connect to the database. Example:

        psql -d my_new_db -h my.db.host.com --password <db.gocd

    See [PostgreSQL documentation](https://www.postgresql.org/docs/current/backup.html) regarding backup and restoration.

  - If you're using MySQL: Use `mysql` to restore using the database backup file (for instance `db.gocd`). This depends on how you connect to the database. Example:

        mysql -D my_new_db -h my.db.host.com -u root -p <db.gocd

    See [MySQL documentation](https://dev.mysql.com/doc/refman/8.0/en/using-mysqldump.html) regarding backup and restore.

- Configuration - Unzip the `config-dir.zip` into a temp directory. Copy all the files from this directory to `${SERVER_INSTALLATION_DIR}/config` directory on Windows and Mac or `/etc/go` on Linux.
- Wrapper Configuration - Unzip the `wrapper-config-dir.zip` into a temp directory. 

  > **Note:** If you are restoring a backup onto a different GoCD version, *or* you have modified the `${SERVER_INSTALLATION_DIR}/wrapper-config/wrapper.conf` please `diff` the backed up files with the pre-installed versions for any differences. Newer GoCD versions can change the default properties required to start GoCD correctly and your old configuration file may not work correctly.
   
  After reviewing (or merging changes into a third copy), copy all the files from your temp directory to `${SERVER_INSTALLATION_DIR}/wrapper-config` directory.
- Configuration History - Unzip the `config-repo.zip` into temp directory. Recursively copy all the contents from this directory to `${SERVER_INSTALLATION_DIR}/db/config.git` .
- Make sure the ownership of all the files that are restored are the same as the user running the Go server.

    > **For example:** Make sure you run a `chown -R go:go ${SERVER_INSTALLATION_DIR}/db/h2db` after Database restoration.

- Start the GoCD server
