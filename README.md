# zabbix-restic

This template is designed to analize restic backup json results file and check if backup is properly done.

There are some zabbix restic templates, but usualy requires external tools like resticprofile or shells to actively send data to zabbix from host.
This template is designed to work without requiring any external tool or shell and you can automatize your backups using crontab and still check
if they are completed succesfully.

The template export file includes two restic backup templates; one for passive mode and the other for active mode. They are equal, and the active template allows to push the log file to the zabbix server when required. you'll need some shell script to get the restic log and push it to server using zabbix_send

Tested on:

* RHEL 10.0
* Zabbix 7.x

## Authors

* Vicente Salvador Cubedo

## Installation

### On zabbix server

* import templates (xml file)
* link restic template to host where the restic backup log is placed.

You can import simple templates or multibackup templates. The method you should use to execute backup changes depending of the template you're using.

Note: I don't delete the simple versión of templates to keep compatibility with users already using this method.

### On monitored server (where you execute the restic backups)

* install and configure zabbix-agent
    ```sh
    dnf install -y zabbix-agent
    ```
    or install zabbix agent 2 if you're using this new version:
    ```sh
    dnf install -y zabbix-agent2 zabbix-agent2-plugin-*
    ```

* install required dependencies for shell commands
   ```sh
   rpm -q jq      || dnf install jq
   ```
   
* reboot zabbix-agent service
    ```sh
    systemctl restart zabbix-agent
    ```

### Configure automatic restic backup using crontab scheduling

## Using simple templates

* you can automatize your restic backup using crontab commands:
    ```
    45 02  * * *  bash -l -c "restic --json --quiet backup /folder1 | jq -c '. + {profile: \"maindata\"}' > /var/log/restic_backup.log"
    00 13  * * *  bash -l -c "restic --json --quiet backup /folder2 | jq -c '. + {profile: \"auxdata\"}' >> /var/log/restic_backup.log"
    ```
  you can add to log file the results of several backup commands. Just keep in mind you should provide different profile names.

## Using multiple templates

This template is not still available, but we document the method to use it in the future.

* you can automatize your restic backup using crontab commands:
    ```
    40 02  * * *  bash -l -c "[ -f /var/log/restic_backup.log ] || echo '{}' > /var/log/restic_backup.log"
    45 02  * * *  bash -l -c "restic --json --quiet backup /folder1 | jq -c --slurpfile input /dev/stdin '. + {maindata: $input[0]}' /var/log/restic_backup.log > /tmp/$$.restic.log && mv -f /tmp/$$.restic.log /var/log/restic_backup.log"
    00 13  * * *  bash -l -c "restic --json --quiet backup /folder2 | jq -c --slurpfile input /dev/stdin '. + {auxdata:  $input[0]}' /var/log/restic_backup.log > /tmp/$$.restic.log && mv -f /tmp/$$.restic.log /var/log/restic_backup.log"
    ```
  this method keeps always last result of each backup execution in the json object stored in /var/log/restic_backup.log keeping this results in a diferent property for each backup executed.
  
# Template restic backup

1. Create a new host to contact zabbix agent on restic backup host
2. Link the template to the host created earlier

## Macros used

|Name|Description|Default|
|----|-----------|----|
|{$BACKUP_STATUS_FILE}|Restic status file |/var/log/restic_backup.log |
|{$MAX_HOURS_BETWEEN}|How many hours between backups to alert. |26 |

## Items

|Name|Description|Type|Key and additional info|
|----|-----------|----|----|
|Restic status file|<p>Get content of JSON Array file where backup logs are placed</p>|`Zabbix agent`|vfs.file.contents[{$BACKUP_STATUS_FILE}]<p>Update: 1h</p>|

## Discovery rules

|Name|Description|Type|Key and additional info|
|----|-----------|----|----|
|Restic log profiles|<p>Discovery of JSON profiles stored in log file.</p>|`Dependent`|backup.results|

## Item prototypes for Restic log profiles discovery

|Name|Description|Type|Key and additional info|
|----|-----------|----|----|
|Restic profile {#PROFILENAME} data added|<p>Data added to backup</p>|`Dependent`|restic.data_added[{#PROFILENAME}]|

## Trigger prototypes for Restic log profiles discovery

|Name|Description|Expression|Priority|
|----|-----------|----------|--------|
|xxx|<p>-</p>|<p>**Expression**: last(/xx/xx)&lt;15m</p><p>**Recovery expression**: </p>|warning|
