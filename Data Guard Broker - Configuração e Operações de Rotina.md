---
documentclass: article
lang: pt-BR

title: Data Guard Broker - Configuração e Operações de Rotina
author: Lucas Pimentel Lellis
toc-title: Sumário
***REMOVED***
---

# Data Guard Broker - Configuração e Operações de Rotina

## Ambiente de Exemplo

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

```{.default .numberLines}
[oracle@linux01 cdb001 ~]$ lsnrctl status

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 30-DEC-2022 21:00:17

Copyright (c) 1991, 2022, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=linux01)(PORT=1521)))
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
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=linux01)(PORT=1521)))
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
    (ADDRESS = (PROTOCOL = TCP)(HOST = linux01)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = cdb001.world)
    )
  )

CDB001_dg =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = linux02)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = cdb001_dg.world)
    )
  )
```

#### Criar standby redo logs

Os Standby Redo Logs (SRLs) devem ser criados na quantidade "Número de redo log groups + 1" para cada thread e com
tamanho igual ou superior ao maior logfile de cada thread.

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

```{.default .numberLines}
[oracle@linux02 cdb002 admin]$ lsnrctl status

LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 30-DEC-2022 21:13:33

Copyright (c) 1991, 2022, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=linux02)(PORT=1521)))
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
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=linux02)(PORT=1521)))
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
    (ADDRESS = (PROTOCOL = TCP)(HOST = linux01)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = cdb001.world)
    )
  )

CDB001_dg =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = linux02)(PORT = 1521))
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
mkdir -p /u02/oradata/CDB001_DG /u03/fast_recovery_area/CDB001_DG /u01/app/oracle/admin/cdb001_dg/adump
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
        set control_files='/u02/oradata/CDB001_DG/control01.ctl','/u03/fast_recovery_area/CDB001_DG/control02.ctl'
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
    StaticConnectIdentifier         = '(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=linux01)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=cdb001_DGMGRL.world)(INSTANCE_NAME=cdb001)(SERVER=DEDICATED)))'
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

### Cascaded Standby - RedoRoutes

A partir da versão 12c, é possível configurar o Redo Transport em cascata caso se tenha dois ou mais Standbys.

Dada a configuração abaixo:

```{.default .numberLines}
DGMGRL> show configuration

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001     - Primary database
    cdb001_dg  - Physical standby database
    cdb001_dg2 - Physical standby database
```

Na configuração padrão, o Redo Transport é feito a partir da base cdb001 para as bases cdb001_dg e cdb001_dg2. Com o
parâmetro `RedoRoutes`, podemos alterar o Redo Transport para ser feito da seguinte maneira:

```{.default}
CDB001 ---> CDB001_DG ---> CDB001_DG2
```

Para estabelecer essa configuração, devemos alterar o parâmetro `RedoRoutes` da seguinte maneira:

```{.default .numberLines}
edit database cdb001 set property redoroutes='(LOCAL : cdb001_dg ASYNC)';

edit database cdb001_dg set property redoroutes='(LOCAL : cdb001 ASYNC, cdb001_dg2 ASYNC)(cdb001 : cdb001_dg2 ASYNC)(cdb001_dg2 : cdb001 ASYNC)';

edit database cdb001_dg2 set property redoroutes='(LOCAL : cdb001_dg ASYNC)';
```

Resultado:

```{.default .numberLines}
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001     - Primary database
    cdb001_dg  - Physical standby database
      cdb001_dg2 - Physical standby database (receiving current redo)

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 31 seconds ago)

DGMGRL>
```

Para exemplos mais complexos, ver o documento [Best Practices for Configuring Redo Transport for Data Guard and Active Data Guard 12c](https://www.oracle.com/technetwork/database/availability/broker-12c-transport-config-2082184.pdf).

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

O arquivo deverá estar na mesma pasta do `alert_<INSTANCE>.log`.

## Role Transitions

### Observações

* Sempre conecte via TNS para executar uma operação de Role Transition

* Antes de executar uma operação de Role Transition, faça um [backup da configuração](#backup-da-configuração).

### Validação

Valide as bases primary e standby. Exemplo:

```{.default .numberLines}
[oracle@linux01 cdb001 ~]$ dgmgrl sys@cdb001
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Sat Dec 31 02:28:28 2022
Version 19.15.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Password:
Connected to "cdb001"
Connected as SYSDBA.
DGMGRL> show configuration

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Physical standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 41 seconds ago)

DGMGRL> validate database verbose cdb001          <<< PRIMARY

  Database Role:    Primary database

  Ready for Switchover:  Yes

  Flashback Database Status:
    cdb001:  On

  Capacity Information:
    Database  Instances        Threads
    cdb001    1                1

  Managed by Clusterware:
    cdb001:  NO
    Validating static connect identifier for the primary database cdb001...
    The static connect identifier allows for a connection to database "cdb001".

  Temporary Tablespace File Information:
    cdb001 TEMP Files:  5

  Data file Online Move in Progress:
    cdb001:  No

  Transport-Related Information:
    Transport On:  Yes

  Log Files Cleared:
    cdb001 Standby Redo Log Files:  Cleared

DGMGRL> validate database verbose cdb001_dg          <<< STANDBY

  Database Role:     Physical standby database
  Primary Database:  cdb001

  Ready for Switchover:  Yes
  Ready for Failover:    Yes (Primary Running)

  Flashback Database Status:
    cdb001   :  On
    cdb001_dg:  Off

  Capacity Information:
    Database   Instances        Threads
    cdb001     1                1
    cdb001_dg  1                1

  Managed by Clusterware:
    cdb001   :  NO
    cdb001_dg:  NO
    Validating static connect identifier for the primary database cdb001...
    The static connect identifier allows for a connection to database "cdb001".

  Temporary Tablespace File Information:
    cdb001 TEMP Files:     5
    cdb001_dg TEMP Files:  5

  Data file Online Move in Progress:
    cdb001:     No
    cdb001_dg:  No

  Standby Apply-Related Information:
    Apply State:      Running
    Apply Lag:        0 seconds (computed 0 seconds ago)
    Apply Delay:      0 minutes

  Transport-Related Information:
    Transport On:  Yes
    Gap Status:    No Gap
    Transport Lag:  0 seconds (computed 0 seconds ago)
    Transport Status:  Success

  Log Files Cleared:
    cdb001 Standby Redo Log Files:     Cleared
    cdb001_dg Online Redo Log Files:   Cleared
    cdb001_dg Standby Redo Log Files:  Available

  Current Log File Groups Configuration:
    Thread #  Online Redo Log Groups  Standby Redo Log Groups Status
              (cdb001)                (cdb001_dg)
    1         6                       7                       Sufficient SRLs

  Future Log File Groups Configuration:
    Thread #  Online Redo Log Groups  Standby Redo Log Groups Status
              (cdb001_dg)             (cdb001)
    1         6                       7                       Sufficient SRLs

  Current Configuration Log File Sizes:
    Thread #   Smallest Online Redo      Smallest Standby Redo
               Log File Size             Log File Size
               (cdb001)                  (cdb001_dg)
    1          200 MBytes                200 MBytes

  Future Configuration Log File Sizes:
    Thread #   Smallest Online Redo      Smallest Standby Redo
               Log File Size             Log File Size
               (cdb001_dg)               (cdb001)
    1          200 MBytes                200 MBytes

  Apply-Related Property Settings:
    Property                        cdb001 Value             cdb001_dg Value
    DelayMins                       0                        0
    ApplyParallel                   AUTO                     AUTO
    ApplyInstances                  0                        0

  Transport-Related Property Settings:
    Property                        cdb001 Value             cdb001_dg Value
    LogShipping                     ON                       ON
    LogXptMode                      ASYNC                    ASYNC
    Dependency                      <empty>                  <empty>
    DelayMins                       0                        0
    Binding                         Mandatory                Mandatory
    MaxFailure                      0                        0
    ReopenSecs                      300                      300
    NetTimeout                      30                       30
    RedoCompression                 DISABLE                  DISABLE

DGMGRL>
```

Valide as configurações de rede:

```{.default .numberLines}
DGMGRL> VALIDATE NETWORK CONFIGURATION FOR ALL;
Connecting to instance "cdb001" on database "cdb001" ...
Connected to "cdb001"
Checking connectivity from instance "cdb001" on database "cdb001 to instance "cdb001_dg" on database "cdb001_dg"...
Succeeded.
Connecting to instance "cdb001_dg" on database "cdb001_dg" ...
Connected to "cdb001_dg"
Checking connectivity from instance "cdb001_dg" on database "cdb001_dg to instance "cdb001" on database "cdb001"...
Succeeded.

Oracle Clusterware is not configured on database "cdb001".
Connecting to database "cdb001" using static connect identifier "(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=linux01)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=cdb001_DGMGRL.world)(INSTANCE_NAME=cdb001)(SERVER=DEDICATED)(STATIC_SERVICE=TRUE)))" ...
Succeeded.
The static connect identifier allows for a connection to database "cdb001".

Oracle Clusterware is not configured on database "cdb001_dg".
Connecting to database "cdb001_dg" using static connect identifier "(DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=linux02)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=cdb001_dg_DGMGRL.world)(INSTANCE_NAME=cdb001_dg)(SERVER=DEDICATED)(STATIC_SERVICE=TRUE)))" ...
Succeeded.
The static connect identifier allows for a connection to database "cdb001_dg".

DGMGRL>
```

### Switchover

Operação que transforma a base Primary em Physical Standby e a base Physical Standby em Primary. Não há perda de dados.

Sintaxe:

```{.default .numberLines}
switchover to <physical standby>;
```

Exemplo:

```{.default .numberLines}
DGMGRL> switchover to cdb001_dg;
Performing switchover NOW, please wait...
Operation requires a connection to database "cdb001_dg"
Connecting ...
Connected to "cdb001_dg"
Connected as SYSDBA.
New primary database "cdb001_dg" is opening...
Operation requires start up of instance "cdb001" on database "cdb001"
Starting instance "cdb001"...
Connected to an idle instance.
ORACLE instance started.
Connected to "cdb001"
Database mounted.
Database opened.
Connected to "cdb001"
Switchover succeeded, new primary is "cdb001_dg"
DGMGRL> show configuration

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001_dg - Primary database
    cdb001    - Physical standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 52 seconds ago)

DGMGRL> show database cdb001

Database - cdb001

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 0 seconds ago)
  Apply Lag:          0 seconds (computed 1 second ago)
  Average Apply Rate: 9.00 KByte/s
  Real Time Query:    ON
  Instance(s):
    cdb001

Database Status:
SUCCESS

DGMGRL> show database cdb001_dg

Database - cdb001_dg

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    cdb001_dg

Database Status:
SUCCESS

DGMGRL>
```

### Failover

Operação que converte a base Standby para Primary. Como pode causar perda de dados, só deve ser usada quando a base
Primary estiver inutilizada e a recuperação não puder ser feita em tempo hábil.

Sintaxes possíveis:

* Aplicando o máximo de Redo possível para minimizar a perda de dados:

  ```{.default .numberLines}
  failover to <standby>;
  ```

* Iniciando o failover imediatamente:

  ```{.default .numberLines}
  failover to <standby> immediate;
  ```

Exemplo:

```{.default .numberLines}
[oracle@linux02 cdb001_dg ~]$ dgmgrl sys@cdb001_dg
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Sat Dec 31 02:48:32 2022
Version 19.15.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Password:
Connected to "cdb001_dg"
Connected as SYSDBA.
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Physical standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 46 seconds ago)

DGMGRL> failover to cdb001_dg;
Performing failover NOW, please wait...
Failover succeeded, new primary is "cdb001_dg"
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001_dg - Primary database
    cdb001    - Physical standby database (disabled)
      ORA-16661: the standby database needs to be reinstated

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 2 seconds ago)

DGMGRL>
```

### Reinstate

Operação que converte para Physical Standby uma base Primary que sofreu failover. Para que possa ser feita de maneira
automática pelo Broker, é necessário que o Flashback Database tenha sido ativado previamente na base Primary antes do
Failover. Caso não tenha, será necessário um restore da base conforme descrito na nota
[416310.1](https://support.oracle.com/epmos/faces/DocContentDisplay?id=416310.1) do MOS.

Sintaxe:

```{.default .numberLines}
reinstate database <db_unique_name>;
```

Exemplo:

```{.default .numberLines}
[oracle@linux02 cdb001_dg ~]$ dgmgrl sys@cdb001_dg
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Sat Dec 31 03:00:59 2022
Version 19.15.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Password:
Connected to "CDB001_DG"
Connected as SYSDBA.
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001_dg - Primary database
    cdb001    - Physical standby database (disabled)
      ORA-16661: the standby database needs to be reinstated

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 2 seconds ago)

DGMGRL> reinstate database cdb001;
Reinstating database "cdb001", please wait...
Operation requires shut down of instance "cdb001" on database "cdb001"
Shutting down instance "cdb001"...
Connected to "cdb001"
ORACLE instance shut down.
Operation requires start up of instance "cdb001" on database "cdb001"
Starting instance "cdb001"...
Connected to an idle instance.
ORACLE instance started.
Connected to "cdb001"
Database mounted.
Connected to "cdb001"
Continuing to reinstate database "cdb001" ...
Reinstatement of database "cdb001" succeeded
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001_dg - Primary database
    cdb001    - Physical standby database
      Warning: ORA-16809: multiple warnings detected for the member

Fast-Start Failover:  Disabled

Configuration Status:
WARNING   (status updated 61 seconds ago)

DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001_dg - Primary database
    cdb001    - Physical standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 34 seconds ago)

DGMGRL> show database cdb001

Database - cdb001

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 0 seconds ago)
  Apply Lag:          0 seconds (computed 0 seconds ago)
  Average Apply Rate: 55.00 KByte/s
  Real Time Query:    ON
  Instance(s):
    cdb001

Database Status:
SUCCESS

DGMGRL>
```

Após o reinstate ser finalizado com sucesso, é possível fazer um switchover para que as bases voltem aos seus status
originais:

```{.default .numberLines}
DGMGRL> switchover to cdb001;
Performing switchover NOW, please wait...
Operation requires a connection to database "cdb001"
Connecting ...
Connected to "cdb001"
Connected as SYSDBA.
New primary database "cdb001" is opening...
Operation requires start up of instance "cdb001_dg" on database "cdb001_dg"
Starting instance "cdb001_dg"...
Connected to an idle instance.
ORACLE instance started.
Connected to "cdb001_dg"
Database mounted.
Database opened.
Connected to "cdb001_dg"
Switchover succeeded, new primary is "cdb001"
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Physical standby database
      Error: ORA-16541: member is not enabled

Fast-Start Failover:  Disabled

Configuration Status:
ERROR   (status updated 287 seconds ago)

DGMGRL> /

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Physical standby database
      Error: ORA-16541: member is not enabled

Fast-Start Failover:  Disabled

Configuration Status:
ERROR   (status updated 314 seconds ago)

DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Physical standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 47 seconds ago)

DGMGRL> show database cdb001_dg;

Database - cdb001_dg

  Role:               PHYSICAL STANDBY
  Intended State:     APPLY-ON
  Transport Lag:      0 seconds (computed 0 seconds ago)
  Apply Lag:          0 seconds (computed 0 seconds ago)
  Average Apply Rate: 17.00 KByte/s
  Real Time Query:    ON
  Instance(s):
    cdb001_dg

Database Status:
SUCCESS

DGMGRL>
```

### Convert to Snapshot Standby

Operação que converte uma base Physical Standby para Snapshot Standby, de modo a possibilitar temporariamente
operações de escrita e DDL (ex.: criação de índices para tuning de relatórios) na base Standby.

Sintaxe:

```{.default .numberLines}
convert database <db_unique_name> to snapshot standby;
```

Exemplo:

```{.default .numberLines}
[oracle@linux01 cdb001 trace]$ dgmgrl sys@cdb001
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Sat Dec 31 03:12:35 2022
Version 19.15.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Password:
Connected to "cdb001"
Connected as SYSDBA.
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Physical standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 42 seconds ago)

DGMGRL> convert database cdb001_dg to snapshot standby;
Converting database "cdb001_dg" to a Snapshot Standby database, please wait...
Database "cdb001_dg" converted successfully
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Snapshot standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 42 seconds ago)

DGMGRL>
```

### Convert to Physical Standby

Operação que converte uma base Snapshot Standby de volta para Physical Standby:

Sintaxe:

```{.default .numberLines}
convert database <db_unique_name> to physical standby;
```

Exemplo:

```{.default .numberLines}
[oracle@linux01 cdb001 trace]$ dgmgrl sys@cdb001
DGMGRL for Linux: Release 19.0.0.0.0 - Production on Sat Dec 31 03:16:07 2022
Version 19.15.0.0.0

Copyright (c) 1982, 2019, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Password:
Connected to "cdb001"
Connected as SYSDBA.
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Snapshot standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 51 seconds ago)

DGMGRL> convert database cdb001_dg to physical standby;
Converting database "cdb001_dg" to a Physical Standby database, please wait...
Operation requires shut down of instance "cdb001_dg" on database "cdb001_dg"
Shutting down instance "cdb001_dg"...
Connected to "cdb001_dg"
Database closed.
Database dismounted.
ORACLE instance shut down.
Operation requires start up of instance "cdb001_dg" on database "cdb001_dg"
Starting instance "cdb001_dg"...
Connected to an idle instance.
ORACLE instance started.
Connected to "cdb001_dg"
Database mounted.
Connected to "cdb001_dg"
Continuing to convert database "cdb001_dg" ...
Database "cdb001_dg" converted successfully
DGMGRL> show configuration;

Configuration - cdb001-standby

  Protection Mode: MaxPerformance
  Members:
  cdb001    - Primary database
    cdb001_dg - Physical standby database

Fast-Start Failover:  Disabled

Configuration Status:
SUCCESS   (status updated 40 seconds ago)

DGMGRL>
```

## Referências { - }

* [Data Guard Broker - 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/dgbkr/index.html)
* [ORACLE-BASE - Data Guard Physical Standby Setup Using the Data Guard Broker in Oracle Database 19c](https://oracle-base.com/articles/19c/data-guard-setup-using-broker-19c)
* [RMAN-08591: WARNING: invalid archived log deletion policy](https://oracle-dba-help.blogspot.com/2018/10/RMAN-08591-backup-failure-standby-database-backup.html)
* [Insufficient SRLs reported by DGMGRL](https://christian-gohmann.de/2019/07/19/insufficient-srls-reported-by-dgmgrl/)
* [Known issues when using "Validate database" DGMGRL command (Doc ID 2300040.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?id=2300040.1)
* [Step by Step Guide on How To Reinstate Failed Primary Database into Physical Standby (Doc ID 738642.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?id=738642.1)
* [Reinstating a Physical Standby Using Backups Instead of Flashback (Doc ID 416310.1)](https://support.oracle.com/epmos/faces/DocContentDisplay?id=416310.1)
* [Best Practices for Configuring Redo Transport for Data Guard and Active Data Guard 12c](https://www.oracle.com/technetwork/database/availability/broker-12c-transport-config-2082184.pdf)