# S3 backup

An Ansible role to backup MySQL or MariaDB databases to an S3 bucket or SFTP server.

## Requirements

* The `s3cmd` command must be installed and executable by the user running the role.
* The `mysql` command must be installed and executable by the user running the role.
* The `scp` command must be installed and executable by the user running the role.
* The `ssh` command must be installed and executable by the user running the role.

Please see [s3cmd's documentation](https://s3tools.org/download) on how to install the command.

## Dependencies

The following collections must be installed:

* cloud.common
* amazon.aws
* community.general
* community.mysql

## Role Variables

This role requires one dictionary as configuration, `mysql_backup`:

```yaml
    mysql_backup:
      s3cmd: "/usr/local/bin/s3cmd"
      debug: true
      stopOnFailure: false
      sources: {}
      remotes: {}
      backups: []
```

Where:
* `s3cmd` is the full path to the `s3cmd` executable. Optional, defaults to `s3cmd`.
* `debug` is `true` to enable debugging output. Optional, defaults to `false`.
* `stopOnFailure` is `true` to stop the entire role if any one backup fails. Optional, defaults to `false`.
* `sources` is a dictionary of sites and environments. Required.
* `remotes` is a dictionary of remote upload locations. Required.
* `backups` is a list of backups to perform. Required.

### Specifying Sources

In this role, "sources" specify the source from which to download backups. Each must have a unique key which is later used in the `mysql_backup.backups` list.

```yaml
mysql_backup:
  sources:
    my-prod-db:
      host: "mysql.example.com"
      port: 3306
      usernameFile: "/path/to/username.txt"
      passwordFile: "/path/to/password.txt"
      tlsCertFile: "/path/to/tls.cert"
      tlsKeyFile: "/path/to/tls.key"
      retryCount: 3
      retryDelay: 30
```

Where, in each entry:
* `host` is the hostname of the database server. 
* `port` is the port with which to connect to the database.
* `usernameFile` is the path to a file containing the username. Optional if `username` is specified.
* `username` is the username with which to connect to the database. Ignored if `usernameFile` is specified.
* `passwordFile` is the path to a file containing the password. Optional if `password` is specified.
* `password` is the password with which to connect to the database. Ignored if `passwordFile` is specified.
* `tlsCertFile` is the path to the TLS certificate necessary to connect to the database. Optional.
* `tlsKeyFile` is the path to the TLS certificate key necessary to connect to the database. Optional.
* `retryCount` is the number of time to retry `platform` commands if they fail. Optional, defaults to `3`.
* `retryDelay` is the time in seconds to wait before retrying a failed `platform` command. Optional, defaults to `30`.

### Specifying remotes

In this role "remotes" are upload destinations for backups. This role supports S3 or SFTP as remotes. Each remote must have a unique key which is later used in the `mysql_backup.backups` list.

```yaml
    - hosts: servers
      vars:
        mysql_backup:
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              provider: "AWS"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              hostBucket: "my-example-bucket.s3.example.com"
              s3Url: "https://my-example-bucket.s3.example.com"
              region: "us-east-1"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
```

For `s3` type remotes:
* `bucket` is the name of the S3 bucket. 
* `provider` is the S3 provider. [See rclone's S3 documenation on `--s3-provider`](https://rclone.org/s3/#s3-provider) for possible values. Optional, defaults to `AWS`.
* `accessKeyFile` is the path to a file containing the access key. Optional if `accessKey` is specified.
* `accessKey` is the value of the access key necessary to access the bucket. Ignored if `accessKeyFile` is specified.
* `secretKeyFile` is the path to a file containing the secret key. Optional if `secretKey` is specified.
* `secretKey` is the value of the access key necessary to secret the bucket. Ignored if `secretKeyFile` is specified.
* `region` is the AWS region in which the bucket resides. Required if using AWS S3, may be optional for other providers.
* `endpoint` is the S3 endpoint to use. Optional if using AWS, required for other providers.

For `sftp` type remotes:
* `host` is the hostname of the SFTP server. Required.
* `user` is the username necessary to login to the SFTP server. Required.
* `keyFile` is the path to a file containing the SSH private key. Required.
* `pubKeyFile` si the path to a file containing the SSH public key. Required for database backups, ignored for file backups.

### Specifying backups

The `mysql_backup.backups` list specifies the database backups perform, referencing the `mysql_backup.sources` and `mysql_backup.remotes` sections for connectivity details.

```yaml
mysql_backup:
  backups:
    - name: "example.com database"
      source: "my-prod-db"
      database: "my-site-live"
      path: "path/in/source/bucket"
      disabled: false
      targets: []
```

Where:
* `name` is the display name of the backup. Optional, but makes the logs easier.
* `source` is the name of the key under `mysql_backups.sources` from which to generate the backup. Required.
* `database` is the name of the database to backup. Required.
* `path` is the path in the source bucket from which to get files. Optional.
* `disabled` is `true` to disable (skip) the backup. Optional, defaults to `false`.
* `targets` is a list of remotes and additional destination information about where to upload backups. Required.

### Backup targets

Backup targets reference a key in `mysql_backup.remotes`, and combine that with additional information used to upload this specific backup.

```yaml
mysql_backup:
  backups:
    - name: "example.com database"
      source: "my-prod-db"
      database: "my-site-live"
      path: "path/in/target/bucket"
      disabled: false
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          disabled: false
```

Where:
* `remote` is the key under `mysql_backup.remotes` to use when uploading the backup. Required.
* `path` is the path on the remote to upload the backup. Optional.
* `disabled` is `true` to skip uploading to the specifed `remote`. Optional, defaults to `false`.

### Ping URL on completion

When a backup completes, you have the option to ping an URL via HTTP:

```yaml
mysql_backup:
  backups:
    - name: "example.com database"
      source: "my-prod-db"
      database: "my-site-live"
      path: "path/in/target/bucket"
      healthcheckUrl: "https://pings.example.com/path/to/service"
      disabled: false
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          disabled: false
```

Where:
* `healthcheckUrl` is the URL to ping when the backup completes successfully. Optional.

### Rotating backups

Backups are uploaded to the remote with the `&lt;database_name&gt;.&lt;host&gt;.&lt;port&gt;-0.sql.gz`. Often, you'll want to retain previous backups in the case an older backup can aid in research or recovery. This role supports retaining and rotating multiple backups using the `retainCount` key.

```yaml
mysql_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      database: "my-site-live"
      element: "database"
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          retainCount: 3
          disabled: true
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          retainCount: 3
          disabled: false
```

Where:
* `retainCount` is the total number of backups to retain in the directory. Optional. Defaults to `1`, or no rotation.

During a backup, if `retainCount` is set:
1. The backup with the ending `&lt;retainCount - 1&gt;.tar.gz` is deleted.
2. Starting with `&lt;retainCount - 2&gt;.tar.gz`, each backup is renamed incremending the ending index.
3. The new backup is uploaded with a `0` index as `&lt;database_name&gt;.&lt;host&gt;.&lt;port&gt;-0.sql.gz`.

This feature works both in S3 and SFTP.

### Upload retries

Sometimes an upload will fail due to transient network issues. You can specify how to control the backup using the `retries` and `retryDelay` keys on each target:

```yaml
mysql_backup:
  backups:
    - name: "example.com database"
      source: "example.com"
      database: "my-site-live"
      element: "database"
      targets:
        - remote: "example-s3-bucket"
          path: "example.com/database"
          retries: 3
          retryDelay: 30
        - remote: "sftp.example.com"
          path: "backups/example.com/database"
          retries: 3
          retryDelay: 30
```

Where:
* `retries` is the total number of retries to perform if the upload should fail. Defaults to no retries.
* `retryDelay` the number of seconds to wait between retries. Defaults to no delay.

## Example Playbook

```yaml
    - hosts: servers
      vars:
        mysql_backup:
          sources:
            my-prod-db:
              host: "mysql.example.com"
              port: 3306
              usernameFile: "/path/to/username.txt"
              passwordFile: "/path/to/password.txt"
              tlsCertFile: "/path/to/tls.cert"
              tlsKeyFile: "/path/to/tls.key"
              retryCount: 3
              retryDelay: 30
          remotes:
            example-s3-bucket:
              type: "s3"
              bucket: "my-s3-bucket"
              provider: "AWS"
              accessKeyFile: "/path/to/aws-s3-key.txt"
              secretKeyFile: "/path/to/aws-s3-secret.txt"
              hostBucket: "my-example-bucket.s3.example.com"
              s3Url: "https://my-example-bucket.s3.example.com"
              region: "us-east-1"
            sftp.example.com:
              type: "sftp"
              host: "sftp.example.com"
              user: "example_user"
              keyFile: "/config/id_example_sftp"
              pubKeyFile: "/config/id_example_sftp.pub"
            backups:
              - name: "example.com database"
                source: "my-prod-db"
                database: "my-site-live"
                path: "path/in/target/bucket"
                healthcheckUrl: "https://pings.example.com/path/to/service"
                disabled: false
                targets:
                  - remote: "example-s3-bucket"
                    path: "example.com/database"
                    retainCount: 3
                    disabled: true
                  - remote: "sftp.example.com"
                    path: "backups/example.com/database"
                    retainCount: 3
                    disabled: false
      roles:
         - { role: ten7.mysql_backup }
```

## License

GPL v3

## Author Information

This role was created by [TEN7](https://ten7.com/).
