### Docker

#### Setting backup configuration options for Docker
The `backup.config` file is dynamically populated at runtime with all `GHE_` environment variables that are part of the run command or Docker environment:

```
$ docker run -it -e "GHE_HOSTNAME=hostname" \
-e "GHE_DATA_DIR=/data" \
-e "GHE_EXTRA_SSH_OPTS=-i /ghe-ssh/id_rsa -o UserKnownHostsFile=/ghe-ssh/known_hosts \
-e "GHE_NUM_SNAPSHOTS=15" \
-v ghe-backup-data:/data \
-v "$HOME/.ssh/known_hosts:/ghe-ssh/known_hosts" \
-v "$HOME/.ssh/id_rsa:/ghe-ssh/id_rsa" \
--rm \
backup-utils ghe-backup
```

It is also possible to specify a `-e GHE_BACKUP_CONFIG` flag and volume mount in a local `backup.config` file rather than specify the variables individually at run time:

```
$ docker run -it  -e "GHE_BACKUP_CONFIG=/mnt/backup.config" \
-v ghe-backup-data:/data \
-v "$HOME/.ssh/known_hosts:/ghe-ssh/known_hosts" \
-v "$HOME/.ssh/id_rsa:/ghe-ssh/id_rsa" \
-v $HOME/backup-utils/backup.config:/mnt/backup.config \
--rm \
backup-utils ghe-backup
```

#### SSH Keys with Docker

A SSH private key that has been added to the GitHub Enterprise [Management Console for administrative SSH access](https://help.github.com/enterprise/admin/guides/installation/administrative-shell-ssh-access/) needs to be mounted into the container from the host system. It is also recommended to mount a SSH `.ssh/known_hosts` file into the container.

```
$ docker run -it -e "GHE_HOSTNAME=hostname" \
-e "GHE_DATA_DIR=/data" \
-e "GHE_EXTRA_SSH_OPTS=-i /ghe-ssh/id_rsa -o UserKnownHostsFile=/ghe-ssh/known_hosts \
-v ghe-backup-data:/data \
-v "$HOME/.ssh/known_hosts:/ghe-ssh/known_hosts" \
-v "$HOME/.ssh/id_rsa:/ghe-ssh/id_rsa" \
--rm \
backup-utils ghe-backup
```

#### Managing backup data with Docker

Data persistence is achieved by using [Docker volumes](https://docs.docker.com/engine/admin/volumes/volumes/), which are managed with [`docker volume` commands](https://docs.docker.com/engine/reference/commandline/volume/). Prior to running the container for the first time, a volume can be created if you need to specify additional options. The named volume will be automatically created at runtime if it does not exist:

```
docker volume create ghe-backup-data
```

The named Docker volume can be mounted and accessed from other containers, using any image you like:

```
# Accessing backups using the backup-utils image:

$ docker run -it -v ghe-backup-data:/data --rm backup-utils ls -l /data/
total 8
drwxr-xr-x 11 root root 4096 Oct 24 19:46 20171024T194650
drwxr-xr-x 11 root root 4096 Oct 24 19:49 20171024T194921
lrwxrwxrwx  1 root root   15 Oct 24 19:49 current -> 20171024T194921

# Accessing backups using the busybox library image:

$ docker run --rm -v ghe-backup-data:/data busybox ls -l /data
total 8
drwxr-xr-x   11 root     root          4096 Oct 24 19:46 20171024T194650
drwxr-xr-x   11 root     root          4096 Oct 24 19:49 20171024T194921
lrwxrwxrwx    1 root     root            15 Oct 24 19:49 current -> 20171024T194921
```

The volume's filesystem must support hard links. This works fine on Docker for Mac with the default local driver in my testing so far.

Bind mounting a volume is supported, as long as the Docker host supports them and allows hardlinks.

#### Scheduling backups using backup-utils in Docker

Designed to be a "one shot" type container, scheduling backup runs with the Docker image is similar to the non-Docker scheduling. Run the container with all the same variables options and volume mounts on `crontab`. This avoids needing to run `crond` or an init system inside the container, and allows for the container to be disposable (enabling the use of Docker's `--rm` flag).

To schedule hourly backup snapshots with verbose informational output written to a log file and errors generating an email:

```
MAILTO=admin@example.com

0 * * * * /usr/local/bin/docker run -i -e "GHE_HOSTNAME=hostname" -e "GHE_DATA_DIR=/data" -e "GHE_EXTRA_SSH_OPTS=-i /ghe-ssh/ghelocal -o UserKnownHostsFile=/ghe-ssh/known_hosts" -v ghe-backup-data:/data -v "$HOME/.ssh/ghelocal:/ghe-ssh/ghelocal" -v "$HOME/.ssh/known_hosts:/ghe-ssh/known_hosts" --rm backup-utils ghe-backup -v 1>>/opt/backup-utils/backup.log 2>&1
```

To schedule nightly backup snapshots instead, use:

```
MAILTO=admin@example.com

0 0 * * * /usr/local/bin/docker run -i -e "GHE_HOSTNAME=hostname" -e "GHE_DATA_DIR=/data" -e "GHE_EXTRA_SSH_OPTS=-i /ghe-ssh/ghelocal -o UserKnownHostsFile=/ghe-ssh/known_hosts" -v ghe-backup-data:/data -v "$HOME/.ssh/ghelocal:/ghe-ssh/ghelocal" -v "$HOME/.ssh/known_hosts:/ghe-ssh/known_hosts" --rm backup-utils ghe-backup -v 1>>/opt/backup-utils/backup.log 2>&1
```