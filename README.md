# ossec-server

An ossec-server image with the ability to separate the ossec configuration/data from the container, meaning easy container replacements. This image is designed to be as turn key as possible, supporting out of the box:

1. Automatic enrollment for agents, using ossec-authd
2. Remote syslog forwarding for the ossec server messages
3. SMTP notifications _(requires no-auth SMTP server)_


The following directories are externalized under `/var/ossec/data` which allow the container to be replaced without configuration or data loss: `logs`, `etc`, `stats`,`rules`, and `queue`. In addition to those directories, the `bin/.process_list` file is symlink'ed to `process_list` in the data volume.

## Quick Start

To get an up and running ossec server that supports auto-enrollment and sends HIDS notifications a syslog server, use.

```
 docker run --name ossec-server -d -p 1514:1514/udp -p 1515:1515\
  -e SYSLOG_FORWADING_ENABLED=true -e SYSLOG_FORWARDING_SERVER_IP=X.X.X.X\
  -v /somepath/ossec_mnt:/var/ossec/data xetusoss/ossec-server
```

Once the system starts up, you can execute the standard ossec commands using docker. For example, to list active agents.

```
docker exec -ti ossec-server /var/ossec/bin/list_agents -a
```

## Available Configuration Parameters

* __AUTO_ENROLLMENT_ENABLED__: Specifies whether or not to enable auto-enrollment via ossec-authd. Defaults to `true`;
* __AUTHD_OPTIONS__: Options to passed ossec-authd, other than -p and -g. Defaults to empty;
* __SMTP_ENABLED__: Whether or not to enable SMTP notifications. Defaults to `true` if ALERTS_TO_EMAIL is specified, otherwise `false`
* __SMTP_RELAY_HOST__: The relay host for SMTP messages, required for SMTP notifications. This host must support non-authenticated SMTP ([see this thread](https://ossec.uservoice.com/forums/18254-general/suggestions/803659-allow-full-confirguration-of-smtp-service-in-ossec)). No default.
* __ALERTS_FROM_EMAIL__: The email address the alerts should come from. Defaults to `ossec@$HOSTNAME`.
* __ALERTS_TO_EMAIL__: The destination email address for SMTP notifications, required for SMTP notifications. No default.
* __SYSLOG_FORWADING_ENABLED__: Specify whether syslog forwarding is enabled or not. Defaults to `false`.
* __SYSLOG_FORWARDING_SERVER_IP__: The IP for the syslog server to send messagse to, required for syslog fowarding. No default.
* __SYSLOG_FORWARDING_SERVER_PORT__: The destination port for syslog messages. Default is `514`.
* __SYSLOG_FORWARDING_FORMAT__: The syslog message format to use. Default is `default`.

**Please note**: All the SMTP and SYSLOG configuration variables are only applicable to the first time setup. Once the container's data volume has been initialized, all the configuration options for OSSEC can be changed.



## Agentless monitoring

After you installed OSSEC, you need to enable the agentless monitoring:

```
/var/ossec/bin/ossec-control enable agentless
```
#### Connection with public key authentication

```
sudo -u ossec ssh-keygen
```
It will create the public keys inside `/var/ossec/.ssh`. After that, just scp the public key to the remote box and your password less connection should work.

```
ssh-copy-id -i ~/.ssh/id_rsa.pub user@10.0.0.1
```
Add the endpoint by running the following command on the OSSEC server:

```
/var/ossec/agentless/register_host.sh add user@10.0.0.1 NOPASS
```

The command output must be similar to the following:

`*Host user@test.com added.`

#### List connected endpoints
Use the following command to display the connected endpoints:

```
/var/ossec/agentless/register_host.sh list
```
Output:
`*Available hosts:`
`user@10.0.0.1`

#### Configuring agentless
Add the setting below to the `/data/etc/ossec.conf` configuration file of the Wazuh server to monitor the /bin and /etc directories

Example of configuring of linux file integrity monitoring with common "ssh_integrity_check_linux" script:

```
<agentless>
    <type>ssh_integrity_check_linux</type>
    <frequency>3600</frequency>
    <host>user@10.0.0.1</host>
    <state>periodic</state>
    <arguments>/bin /etc</arguments>
</agentless>
```
See more about scripts: https://www.ossec.net/docs/docs/manual/agent/agentless-scripts.html


#### Running the completed setup
Once the configuration is completed, you can restart OSSEC. You should see something like “Started ossec-agentlessd” in the output. Before each agentless connection is started, OSSEC will do a configuration check to make sure everything is fine. Look at `/data/logs/ossec.log` for any error. You should see after restart:

`2022/12/12 15:24:12 ossec-agentlessd: INFO: Test passed for 'ssh_integrity_check_bsd'.'`

When it connects to the remote agentless host, you will also see:

`2022/12/12 15:25:19 ossec-agentlessd: INFO: ssh_integrity_check_bsd: user@10.0.0.1: Starting.`
`2022/12/12 15:25:46 ossec-agentlessd: INFO: ssh_integrity_check_bsd: user@10.0.0.1: Finished.`

## Alerts

By default, OSSEC writes alerts to a log file located along the path: `/data/logs/alerts/alerts.log`

See more about alerts output options: https://www.ossec.net/docs/docs/manual/output/index.html




