---
documentclass: article
lang: pt-BR

title: Data Guard Broker - Configuração e Operações de Rotina
author: Lucas Pimentel Lellis
---

# Data Guard Broker - Configuração e Operações de Rotina

## Ambiente

| Informação     | Primary  | Standby   |
|----------------|----------|-----------|
| db_name        | CDB001   | CDB001    |
| db_unique_name | CDB001   | CDB001_dg |
| db_domain      | world    | world     |
| Listener       | LISTENER | LISTENER  |
| Hostname       | linux01  | linux02   |
| Versão         | 19.15    | 19.15     |

## Pré-Requisitos

### Primary

#### Listener - Entradas estáticas

Incluir no arquivo `$ORACLE_HOME/network/admin/listener.ora`:

```{.default .numberLines}
SID_LIST_LISTENER=
      (
        SID_LIST=
          (
            SID_DESC=
              (SID_NAME=cdb001)
              (GLOBAL_DBNAME=cdb001_DGMGRL.world)
              (ORACLE_HOME=/u01/app/oracle/product/19.0.0/db_1)
              (ENVS="TNS_ADMIN=/u01/app/oracle/product/19.0.0/db_1/network/admin")
          )
          (
            SID_DESC=
              (SID_NAME=cdb001)
              (GLOBAL_DBNAME=cdb001.world)
              (ORACLE_HOME=/u01/app/oracle/product/19.0.0/db_1)
              (ENVS="TNS_ADMIN=/u01/app/oracle/product/19.0.0/db_1/network/admin")
          )
      )
```

Execute um stop/start do listener:

```{.bash .numberLines}
lsnrctl stop LISTENER && lsnrctl start LISTENER

sqlplus / as sysdba <<EOF
alter system register;
exit
EOF

```

Verifique que os serviços estáticos estão disponíveis:

```{.bash .numberLines}
[oracle@linux01 cdb001 ~]$ lsnrctl status

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 30-DEC-2022 21:00:17

Copyright (c) 1991, 2022, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=linux01.gcp.goldopet.win)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     listener
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                30-DEC-2022 20:59:17
Uptime                    0 days 0 hr. 1 min. 0 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/19.0.0/db_1/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/linux01/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=linux01.gcp.goldopet.win)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
Services Summary...
Service "cdb001.world" has 2 instance(s).
  Instance "cdb001", status UNKNOWN, has 1 handler(s) for this service...       <<< SERVIÇO ESTÁTICO
  Instance "cdb001", status READY, has 1 handler(s) for this service...
Service "cdb001XDB.world" has 1 instance(s).
  Instance "cdb001", status READY, has 1 handler(s) for this service...
Service "cdb001_DGMGRL.world" has 1 instance(s).
  Instance "cdb001", status UNKNOWN, has 1 handler(s) for this service...       <<< SERVIÇO ESTÁTICO
Service "efbbb315a8f7344ae0530b31a8c0e2f0.world" has 1 instance(s).
  Instance "cdb001", status READY, has 1 handler(s) for this service...
Service "efbc4fde254a5135e0530b31a8c0e81c.world" has 1 instance(s).
  Instance "cdb001", status READY, has 1 handler(s) for this service...
Service "efcbf70b67fe3819e0530b31a8c0d96e.world" has 1 instance(s).
  Instance "cdb001", status READY, has 1 handler(s) for this service...
Service "efcbf70b68003819e0530b31a8c0d96e.world" has 1 instance(s).
  Instance "cdb001", status READY, has 1 handler(s) for this service...
Service "orcl001.world" has 1 instance(s).
  Instance "orcl001", status READY, has 1 handler(s) for this service...
Service "orcl001XDB.world" has 1 instance(s).
  Instance "orcl001", status READY, has 1 handler(s) for this service...
Service "pdb001.world" has 1 instance(s).
  Instance "cdb001", status READY, has 1 handler(s) for this service...
Service "pdb002.world" has 1 instance(s).
  Instance "cdb001", status READY, has 1 handler(s) for this service...
Service "pdb003.world" has 1 instance(s).
  Instance "cdb001", status READY, has 1 handler(s) for this service...
The command completed successfully
[oracle@linux01 cdb001 ~]$
```

#### Tnsnames

Incluir no arquivo `$ORACLE_HOME/network/admin/tnsnames.ora`:

```{.default .numberLines}
CDB001 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = linux01.gcp.goldopet.win)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = cdb001.world)
    )
  )

CDB001_dg =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = linux02.gcp.goldopet.win)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = cdb001_dg.world)
    )
  )
```

#### Criar standby redo logs - Número de redo log groups + 1

```{.sql .numberLines}
alter database add standby logfile thread 1 group 7 size 200m;
alter database add standby logfile thread 1 group 8 size 200m;
alter database add standby logfile thread 1 group 9 size 200m;
alter database add standby logfile thread 1 group 10 size 200m;
alter database add standby logfile thread 1 group 11 size 200m;
alter database add standby logfile thread 1 group 12 size 200m;
alter database add standby logfile thread 1 group 13 size 200m;
```

#### Colocar banco em flashback - Opcional

```{.sql .numberLines}
alter database flashback on;
```

#### Habilitar Force Logging

```{.sql .numberLines}
alter database force logging;

alter system archive log current;
```

### Standby

#### Listener - Entradas estáticas

Incluir no arquivo `$ORACLE_HOME/network/admin/listener.ora`:

```{.default .numberLines}
SID_LIST_LISTENER=
      (
        SID_LIST=
          (
            SID_DESC=
              (SID_NAME=cdb001_dg)
              (GLOBAL_DBNAME=cdb001_dg_DGMGRL.world)
              (ORACLE_HOME=/u01/app/oracle/product/19.0.0/db_1)
              (ENVS="TNS_ADMIN=/u01/app/oracle/product/19.0.0/db_1/network/admin")
          )
          (
            SID_DESC=
              (SID_NAME=cdb001_dg)
              (GLOBAL_DBNAME=cdb001_dg.world)
              (ORACLE_HOME=/u01/app/oracle/product/19.0.0/db_1)
              (ENVS="TNS_ADMIN=/u01/app/oracle/product/19.0.0/db_1/network/admin")
          )
      )
```

Execute um stop/start do listener:

```{.bash .numberLines}
lsnrctl stop LISTENER && lsnrctl start LISTENER
```

Verifique que os serviços estáticos estão disponíveis:

```{.bash .numberLines}
[oracle@linux02 cdb002 admin]$ lsnrctl status

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 30-DEC-2022 21:13:33

Copyright (c) 1991, 2022, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=linux02.gcp.goldopet.win)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
Start Date                30-DEC-2022 21:13:23
Uptime                    0 days 0 hr. 0 min. 9 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/19.0.0/db_1/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/linux02/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=linux02.gcp.goldopet.win)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
Services Summary...
Service "cdb001_dg.world" has 1 instance(s).
  Instance "cdb001", status UNKNOWN, has 1 handler(s) for this service...   <<< SERVIÇO ESTÁTICO
Service "cdb001_dg_DGMGRL.world" has 1 instance(s).
  Instance "cdb001", status UNKNOWN, has 1 handler(s) for this service...   <<< SERVIÇO ESTÁTICO
The command completed successfully
[oracle@linux02 cdb002 admin]$
```

#### Tnsnames

Incluir no arquivo `$ORACLE_HOME/network/admin/tnsnames.ora`:

```{.default .numberLines}
CDB001 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = linux01.gcp.goldopet.win)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = cdb001.world)
    )
  )

CDB001_dg =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = linux02.gcp.goldopet.win)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = cdb001_dg.world)
    )
  )
```

#### Oratab

```{.bash .numberLines}
echo "cdb001_dg:$ORACLE_HOME:Y" >> /etc/oratab
```

#### Pfile inicial

```{.bash .numberLines}
cat > $ORACLE_HOME/dbs/initcdb001_dg.ora <<EOF
db_name=cdb001
db_unique_name=cdb001_dg
db_domain=world
sga_max_size=4G
sga_target=4G
EOF
```

#### Criar diretórios

```{.bash .numberLines}
mkdir -p /u02/oradata /u03/fast_recovery_area /u01/app/oracle/admin/cdb001_dg/adump
```

#### Copiar password file da primary

```{.bash .numberLines}
scp linux01:$ORACLE_HOME/dbs/orapwcdb001 $ORACLE_HOME/dbs/orapwcdb001_dg
```

#### Startup Nomount

```{.bash .numberLines}
export ORACLE_SID=cdb001_dg
export ORAENV_ASK=NO
. oraenv

sqlplus / as sysdba <<EOF
startup nomount
exit
EOF
```

#### Executar Duplicate

```{.default .numberLines}
connect target sys/<SENHA>@cdb001
connect auxiliary sys/<SENHA>@cdb001_dg

run {
    allocate channel ch1 device type disk;
    allocate channel ch2 device type disk;
    allocate channel ch3 device type disk;
    allocate channel ch4 device type disk;

    allocate auxiliary channel aux1 device type disk;
    allocate auxiliary channel aux2 device type disk;
    allocate auxiliary channel aux3 device type disk;
    allocate auxiliary channel aux4 device type disk;

    set newname for database to new;

    duplicate target database
      for standby
      from active database section size 200m
      dorecover
      spfile
        set db_unique_name='cdb001_dg'
        set audit_file_dest='/u01/app/oracle/admin/cdb001_dg/adump'
        set job_queue_processes='0'
      nofilenamecheck;
}
```

## Configuração do Broker

### Ativar o DMON (Processo do DG Broker) - Primary e Standby

```{.sql .numberLines}
alter system set dg_broker_start=true;
```

### Configurar o Data Guard via Broker - Primary

Abrir o DGMGRL (conectar via TNS):

```{.bash .numberLines}
dgmgrl sys@cdb001
```

Criar configuração:

```{.default .numberLines}
CREATE CONFIGURATION 'cdb001-standby' AS
    PRIMARY DATABASE IS 'cdb001'
    CONNECT IDENTIFIER IS cdb001;
```

Resultado esperado:

```{.default .numberLines}
Configuration "cdb001-standby" created with primary database "cdb001"
```

Adicionar o banco standby:

```{.default .numberLines}
ADD DATABASE 'cdb001_dg' AS CONNECT IDENTIFIER IS cdb001_dg;
```

Resultado esperado:

```{.default .numberLines}
Database "cdb001_dg" added
```

### Iniciar a Replicação - Primary

Abrir o DGMGRL (conectar via TNS):

```{.bash .numberLines}
dgmgrl sys@cdb001
```

Habilite a configuração:

```{.default .numberLines}
ENABLE CONFIGURATION;
```

No SQL*Plus, force um log switch:

```{.default .numberLines}
alter system archive log current;
```

Volte ao DGMGRL e ajuste os parâmetros abaixo:

```{.default .numberLines}
EDIT DATABASE CDB001 SET PROPERTY 'StandbyFileManagement'='AUTO';
EDIT DATABASE CDB001_DG SET PROPERTY 'StandbyFileManagement'='AUTO';

EDIT DATABASE CDB001 SET PROPERTY 'StandbyArchiveLocation'='USE_DB_RECOVERY_FILE_DEST';
EDIT DATABASE CDB001_DG SET PROPERTY 'StandbyArchiveLocation'='USE_DB_RECOVERY_FILE_DEST';

EDIT DATABASE CDB001 SET PROPERTY 'ApplyLagThreshold'='300';
EDIT DATABASE CDB001_DG SET PROPERTY 'ApplyLagThreshold'='300';

EDIT DATABASE CDB001 SET PROPERTY 'TransportLagThreshold'='300';
EDIT DATABASE CDB001_DG SET PROPERTY 'TransportLagThreshold'='300';

EDIT DATABASE CDB001 SET PROPERTY 'Binding'='Mandatory';
EDIT DATABASE CDB001_DG SET PROPERTY 'Binding'='Mandatory';
```

### Validar o Data Guard

Abrir o DGMGRL (conectar via TNS ou /):

```{.bash .numberLines}
dgmgrl sys@cdb001
```

Validar configuração:

```{.default .numberLines}
show configuration verbose
```

Resultado esperado:

```{.default .numberLines}
DGMGRL> show configuration verbose

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Physical standby database

  Properties:
    FastStartFailoverThreshold      = '30'
    OperationTimeout                = '30'
    TraceLevel                      = 'USER'
    FastStartFailoverLagLimit       = '30'
    CommunicationTimeout            = '180'
    ObserverReconnect               = '0'
    FastStartFailoverAutoReinstate  = 'TRUE'
    FastStartFailoverPmyShutdown    = 'TRUE'
    BystandersFollowRoleChange      = 'ALL'
    ObserverOverride                = 'FALSE'
    ExternalDestination1            = ''
    ExternalDestination2            = ''
    PrimaryLostWriteAction          = 'CONTINUE'
    ConfigurationWideServiceName    = 'cdb001_CFG'

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS
```

Validar primary:

```{.default .numberLines}
show database verbose cdb001
```

Resultado esperado:

```{.default .numberLines}
DGMGRL> show database verbose cdb001

Database - cdb001

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    cdb001

  Properties:
    DGConnectIdentifier             = 'cdb001'
    ObserverConnectIdentifier       = ''
    FastStartFailoverTarget         = ''
    PreferredObserverHosts          = ''
    LogShipping                     = 'ON'
    RedoRoutes                      = ''
    LogXptMode                      = 'ASYNC'
    DelayMins                       = '0'
    Binding                         = 'Mandatory'
    MaxFailure                      = '0'
    ReopenSecs                      = '300'
    NetTimeout                      = '30'
    RedoCompression                 = 'DISABLE'
    PreferredApplyInstance          = ''
    ApplyInstanceTimeout            = '0'
    ApplyLagThreshold               = '300'
    TransportLagThreshold           = '300'
    TransportDisconnectedThreshold  = '30'
    ApplyParallel                   = 'AUTO'
    ApplyInstances                  = '0'
    StandbyFileManagement           = ''
    ArchiveLagTarget                = '0'
    LogArchiveMaxProcesses          = '0'
    LogArchiveMinSucceedDest        = '0'
    DataGuardSyncLatency            = '0'
    LogArchiveTrace                 = '0'
    LogArchiveFormat                = ''
    DbFileNameConvert               = ''
    LogFileNameConvert              = ''
    ArchiveLocation                 = ''
    AlternateLocation               = ''
    StandbyArchiveLocation          = 'USE_DB_RECOVERY_FILE_DEST'
    StandbyAlternateLocation        = ''
    InconsistentProperties          = '(monitor)'
    InconsistentLogXptProps         = '(monitor)'
    LogXptStatus                    = '(monitor)'
    SendQEntries                    = '(monitor)'
    RecvQEntries                    = '(monitor)'
    HostName                        = 'linux01'
    StaticConnectIdentifier         = '(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=linux01.gcp.goldopet.win)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=cdb001_DGMGRL.world)(INSTANCE_NAME=cdb001)(SERVER=DEDICATED)))'
    TopWaitEvents                   = '(monitor)'
    SidName                         = '(monitor)'

  Log file locations:
    Alert log               : /u01/app/oracle/diag/rdbms/cdb001/cdb001/trace/alert_cdb001.log
    Data Guard Broker log   : /u01/app/oracle/diag/rdbms/cdb001/cdb001/trace/drccdb001.log

Database Status:
SUCCESS

DGMGRL>
```

Validar standby

```{.default .numberLines}
show database verbose cdb001_dg
```

Resultado esperado:

```{.default .numberLines}
DGMGRL> show database verbose cdb001_dg

Database - cdb001_dg

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 0 seconds ago)
  Apply Lag:          0 seconds (computed 0 seconds ago)
  Average Apply Rate: 47.00 KByte/s
  Active Apply Rate:  2.30 MByte/s
  Maximum Apply Rate: 5.51 MByte/s
  Real Time Query:    OFF
  Instance(s):
    cdb001_dg

  Properties:
    DGConnectIdentifier             = 'cdb001_dg'
    ObserverConnectIdentifier       = ''
    FastStartFailoverTarget         = ''
    PreferredObserverHosts          = ''
    LogShipping                     = 'ON'
    RedoRoutes                      = ''
    LogXptMode                      = 'ASYNC'
    DelayMins                       = '0'
    Binding                         = 'Mandatory'
    MaxFailure                      = '0'
    ReopenSecs                      = '300'
    NetTimeout                      = '30'
    RedoCompression                 = 'DISABLE'
    PreferredApplyInstance          = ''
    ApplyInstanceTimeout            = '0'
    ApplyLagThreshold               = '300'
    TransportLagThreshold           = '300'
    TransportDisconnectedThreshold  = '30'
    ApplyParallel                   = 'AUTO'
    ApplyInstances                  = '0'
    StandbyFileManagement           = ''
    ArchiveLagTarget                = '0'
    LogArchiveMaxProcesses          = '0'
    LogArchiveMinSucceedDest        = '0'
    DataGuardSyncLatency            = '0'
    LogArchiveTrace                 = '0'
    LogArchiveFormat                = ''
    DbFileNameConvert               = ''
    LogFileNameConvert              = ''
    ArchiveLocation                 = ''
    AlternateLocation               = ''
    StandbyArchiveLocation          = 'USE_DB_RECOVERY_FILE_DEST'
    StandbyAlternateLocation        = ''
    InconsistentProperties          = '(monitor)'
    InconsistentLogXptProps         = '(monitor)'
    LogXptStatus                    = '(monitor)'
    SendQEntries                    = '(monitor)'
    RecvQEntries                    = '(monitor)'
    HostName                        = 'linux02'
    StaticConnectIdentifier         = '(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=linux02)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=cdb001_dg_DGMGRL.world)(INSTANCE_NAME=cdb001_dg)(SERVER=DEDICATED)))'
    TopWaitEvents                   = '(monitor)'
    SidName                         = '(monitor)'

  Log file locations:
    Alert log               : /u01/app/oracle/diag/rdbms/cdb001_dg/cdb001_dg/trace/alert_cdb001_dg.log
    Data Guard Broker log   : /u01/app/oracle/diag/rdbms/cdb001_dg/cdb001_dg/trace/drccdb001_dg.log

Database Status:
SUCCESS

DGMGRL>
```

### Configurações do RMAN - Standby

Caso deseje usar a base standby para offload dos backups, é obrigatório o uso de um catálogo acessível tanto pela Primary
quanto pela Standby.

Configuração do `ARCHIVELOG DELETION POLICY` - com uso de catálogo:

```{.default .numberLines}
CONFIGURE ARCHIVELOG DELETION POLICY TO APPLIED ON STANDBY FOR db_unique_name 'cdb001_dg';
```

Configuração do `ARCHIVELOG DELETION POLICY` - sem uso de catálogo:

```{.default .numberLines}
CONFIGURE ARCHIVELOG DELETION POLICY TO APPLIED ON STANDBY;
```

Essa configuração permite que os archivelogs na base Standby sejam gerenciados pela Fast Recovery Area, sem necessidade
de limpeza manual ou via crontab.

### Configurações do RMAN - Primary

* Caso deseje usar a base standby para offload dos backups, é obrigatório o uso de um catálogo acessível tanto pela Primary
  quanto pela Standby.

* Reter na Primary Archives não recebidos na Standby:

  * Configuração do `ARCHIVELOG DELETION POLICY` - com uso de catálogo:

    ```{.default .numberLines}
    CONFIGURE ARCHIVELOG DELETION POLICY TO SHIPPED TO STANDBY BACKED UP 1 TIMES TO DEVICE TYPE DISK FOR db_unique_name 'cdb001';
    ```

  * Configuração do `ARCHIVELOG DELETION POLICY` - sem uso de catálogo:

    ```{.default .numberLines}
    CONFIGURE ARCHIVELOG DELETION POLICY TO SHIPPED TO STANDBY BACKED UP 1 TIMES TO DEVICE TYPE DISK;
    ```

* Reter na Primary Archives não aplicados na Standby:

  * Configuração do `ARCHIVELOG DELETION POLICY` - com uso de catálogo:

    ```{.default .numberLines}
    CONFIGURE ARCHIVELOG DELETION POLICY TO APPLIED ON STANDBY BACKED UP 1 TIMES TO DEVICE TYPE DISK FOR db_unique_name 'cdb001';
    ```

  * Configuração do `ARCHIVELOG DELETION POLICY` - sem uso de catálogo:

    ```{.default .numberLines}
    CONFIGURE ARCHIVELOG DELETION POLICY TO APPLIED ON STANDBY BACKED UP 1 TIMES TO DEVICE TYPE DISK;
    ```

* Não reter archives na Primary:

  * Configuração do `ARCHIVELOG DELETION POLICY` - com uso de catálogo:

    ```{.default .numberLines}
    CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 1 TIMES TO DEVICE TYPE DISK FOR db_unique_name 'cdb001';
    ```

  * Configuração do `ARCHIVELOG DELETION POLICY` - sem uso de catálogo:

    ```{.default .numberLines}
    CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 1 TIMES TO DEVICE TYPE DISK;
    ```

## Operações de Rotina

### Validar se o Broker está sendo usado

No SQL*Plus

```{.sql .numberLines}
show parameter dg_broker_start
```

Resultado esperado:

```{.default .numberLines}
SQL> show parameter dg_broker_start

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
dg_broker_start                      boolean     TRUE
SQL>
```

### Abrir o DGMGRL

Abrir o DGMGRL (conectar via TNS ou /):

```{.bash .numberLines}
dgmgrl sys@cdb001
```

### Validação Básica

Validar configuração:

```{.default .numberLines}
show configuration
```

Resultado esperado:

```{.default .numberLines}
DGMGRL> show configuration

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Physical standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 52 seconds ago)

DGMGRL>
```

Validar primary:

```{.default .numberLines}
show database cdb001
```

Resultado esperado:

```{.default .numberLines}
DGMGRL> show database cdb001

Database - cdb001

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    cdb001

Database Status:
SUCCESS

DGMGRL>
```

Validar standby:

```{.default .numberLines}
show database cdb001_dg
```

Resultado esperado:

```{.default .numberLines}
DGMGRL> show database cdb001_dg

Database - cdb001_dg

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 1 second ago)
  Apply Lag:          0 seconds (computed 1 second ago)
  Average Apply Rate: 2.00 KByte/s
  Real Time Query:    OFF
  Instance(s):
    cdb001_dg

Database Status:
SUCCESS

DGMGRL>
```

### Interromper Redo Apply

```{.default .numberLines}
edit database cdb001_dg set state='APPLY-OFF';
```

### Reiniciar Redo Apply

```{.default .numberLines}
edit database cdb001_dg set state='APPLY-ON';
```

### Interromper Envio de Archives

```{.default .numberLines}
edit database cdb001 set state='TRANSPORT-OFF';
```

### Reiniciar Envio de Archives

```{.default .numberLines}
edit database cdb001 set state='TRANSPORT-ON';
```

### Backup da configuração

```{.default .numberLines}
EXPORT CONFIGURATION TO 'backup_config_cdb001.txt';
```

O arquivo sempre será gerado na mesma pasta do `alert_<INSTANCE>.log`.

### Restore da configuração

```{.default .numberLines}
IMPORT CONFIGURATION FROM 'backup_config_cdb001.txt';
```

O arquivo sdeverá estar na mesma pasta do `alert_<INSTANCE>.log`.

## Role Transitions

### Switchover

### Failover

### Reinstate

### Convert to Snapshot Standby

### Convert to Physical Standby

## Referências { - }

* [Data Guard Broker - 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/dgbkr/index.html)
* [ORACLE-BASE - Data Guard Physical Standby Setup Using the Data Guard Broker in Oracle Database 19c](https://oracle-base.com/articles/19c/data-guard-setup-using-broker-19c)
* [RMAN-08591: WARNING: invalid archived log deletion policy](https://oracle-dba-help.blogspot.com/2018/10/RMAN-08591-backup-failure-standby-database-backup.html)
* [Insufficient SRLs reported by DGMGRL](https://christian-gohmann.de/2019/07/19/insufficient-srls-reported-by-dgmgrl/)
* [Known issues when using "Validate database" DGMGRL command (Doc ID 2300040.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?id=2300040.1)
