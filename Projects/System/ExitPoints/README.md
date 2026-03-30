# Exit points

## QIBM_QZDA_INIT and DDMACC exit points

I got the suggestion to create a program which limits database connections after a couple of abuses on PUB400. For instance a user initiated dozens of xDBC connections, leading the system to show several Ineligible QZDASOINIT jobs in QUSRWRK subsystem, which I do not remember seeing this behavior in my past experience! Or, another user is connecting to PUB400 with tens and tens of DRDA sessions, as showed below.

|DATETIME|AUTHORIZATION_NAME|REMOTE_ADDRESS|LOCAL_PORT_NAME|ACTIVE_CONNECTIONS|
|--------|------------------|--------------|---------------|------------------|
|2026-03-30 16:16:02.280251|FLIMFLAM|75.42.37.39|as-database|10|
|2026-03-30 16:16:02.280251|QUSER|34.170.200.155|ddm-ssl|373|
|2026-03-30 16:16:02.280251|TANAWAT|13.212.151.138|as-database|4|

Check out the online IBM documentation at these [Use server exit program](https://www.ibm.com/docs/en/i/7.5.0?topic=administration-use-server-exit-programs) and [DRDA and DDM server access control using user exit programs](https://www.ibm.com/docs/en/i/7.5.0?topic=security-drda-ddm-server-access-control-using-user-exit-programs) pages.

Disclaimer: we have no authority to play with exit points on PUB400, therefore, this program was never tested as an exit program. The name of this program is [DBSSNLMT](dbssnlmt.pgm.sqlrpgle). This is an SQL RPG ILE program. Note that it makes use of a *caller activation group as it is expected to be called by the real [DBSSNLMT1](dbssnlmt1.pgm.clle) and [DBSSNLTM2](dbssnlmt2.pgm.clle) exit programs, which have, on their side a \*new activation group. The reason for that is that we want all the SQL resources that it creates to be properly closed when it completes and do not disturb the calling programs (i.e. system programs).

### DBSSNLMT1 behavior

As the program is to be set as an exit program of DDMACC network attribute exit point, it must have two parameters:

1. A 1 character variable as ouput parameter to contain:
    - '0' to tell the exit point to reject the request
    - '1' to tell the exit point to accept the request
2. A 2095 characters variable as input paramater which is filled according to DDMACC exit point format.

Firstly, it retrieves the library it resides in, then call the main program DBSSNLMT. I decided to use a dynamic call in place of a subprocedure within a module or a service program, in order to be free enough to change the main program without impacting the exit point.

Note that, with this setup, we do not take care of the content of the input parameter.

### DBSSNLMT2 behavior

As the program is to be set as an exit program of QIBM_QZDA_INIT exit point, it must have two parameters:

1. A 1 character variable as ouput parameter to contain:
    - '0' to tell the exit point to reject the request
    - '1' to tell the exit point to accept the request
2. A 285 characters variable as input paramater which is filled according to ZDAI0100 format.

Firstly, it retrieves the library it resides in, then call the main program DBSSNLMT. I decided to use a dynamic call in place of a subprocedure within a module or a service program, in order to be free enough to change the main program without impacting the exit point.

Note that, with this setup, we do not take care of the content of the input parameter.

### DBSSNLMT behavior

As the program is to be set as an exit program of QIBM_QZDA_INIT exit point, it must have one parameter:

1. A 1 character variable as ouput parameter to contain:
    - '0' to tell the exit point to reject the request
    - '1' to tell the exit point to accept the request

The first operation of the program is to gather its configuration from environment variables if they exist at job level. Indeed, the way to provide some configuration to this program is to create/update up to four environment variables at the system level. Do not forget that system level environment variables are duplicated to the job level at initiation time of each job. The program expects up to four environment variables, and gather this information through [qsys2.environment_variable_info SQL view](https://www.ibm.com/docs/en/i/7.5.0?topic=services-environment-variable-info-view).

- DATABASE_LIMIT_CONCURRENTSESSIONS is the number of allowed concurrent connections. If it does not exist or contain an invalid value, the default is 5 and set in the program. Valid values are numeric integer values.
- DATABASE_LIMIT_SERVICENAMES is the list of database service names. If it does not exist, the default is as-database,as-database-s,drda,ddm,ddm-ssl and set in the program. Valid values are a comma separated values list without spaces. There is no control in the program about the validity of any service, which means that the program does not control that a service exists in system service table. If the environment variable contains values which are not a real service, it means that all connections will be accepted.
- DATABASE_LIMIT_ALLWAYSACCEPTQ defines whether or not we accept sessions from a Q* user profile regardless any count of existing sessions. If it does not exist or contains an invalid value, the default is Y which means that we accept the connections regardless the count of active connections (this is to avoid potential issues for connections on Localhost for user profiles such as QWEBADMIN used by IBM Navigator for i), and is set in the program.  Valid values are Y for Yes and N for No.
- DATABASE_LIMIT_INCLUDELOCALHOST defines whether or not, we include local sessions from and to localhost in the count of sessions. If it does not exist or contains an invalid value, the default is N (which means that we do not check for localhost sessions) and is set in the program. Valid values are Y for Yes and N for No.

The following operation is to retrieve the current user profile of the current job. I preferred to use an SQL statement for that rather than the *PSDS data structure, as I am not sure that its user profile subfield is really updated for database server jobs.

The following operation is to check if the situation is the case of a Q* user profile trying to connect and if DATABASE_LIMIT_ALLWAYSACCEPTQ configuration item is Y for Yes. If this is the case, we send a CPI3701 information message to the joblog and set the program output parameter to '1' as required by the exit point, and the program completes.

The last operation is to count the number of active sessions for the current user profile. We use the [qsys2.netstat_job_info SQL view](https://www.ibm.com/docs/en/i/7.5.0?topic=services-netstat-job-info-view) for that.
If the count is lower that the DATABASE_LIMIT_CONCURRENTSESSIONS envirnment variable value, we send a CPI3701 information message to the joblog and set the program output parameter to '1' as required by the exit point, and the program completes.
If the count is equal or higher, we send another CPI3701 information message to the joblog, and a CPD436F information message to QSYSOPR message queue and system log and set the program output to '0' as required by the exit point, and the program completes.

When the exit point receives a flag set to '0', it rejects the connection request. When the exit point receives a flag set to '1', it accepts the connection request.
The goal of sending this CPD436F message to the system log is to provide some basic analysis capability of rejected requests if needed.

### Programs installation

One requirement I had when writing this program was that it should need as less as possible any other object and provide default setting by itself. For instance, as opposite to other tools provided in this repository, it does not use any common file import.

Therefore, to install these programs, it is required to:

1. Upload the sources in a directory of your choice to IFS stream files (with a CCSID set to 1252) named
    - dbssnlmt.pgm.sqlrpgle
    - dbssnlmt1.pgm.clle
    - dbssnlmt2.pgm.clle
2. Compile the programs in the library of your choice, but the same for all the three, using the command below:
    - CRTSQLRPGI OBJ(yourlib/DBSSNTLMT) SRCSTMF('/yourdirectory/dbssnlmt.pgm.sqlrpgle') OBJTYPE(*PGM)
    - CRTBNDCL PGM(yourlib/DBSSNLMT1) SRCSTMF('/yourdirectory/dbssnlmt1.pgm.clle')
    - CRTBNDCL PGM(yourlib/DBSSNLMT2) SRCSTMF('/yourdirectory/dbssnlmt2.pgm.clle')
3. Add it to the ZDA exit point with the command below:
    ADDEXITPGM EXITPNT(QIBM_QZDA_INIT) FORMAT(ZDAI0100) PGMNBR(*LOW) PGM(yourlib/DBSSNLMT2) TEXT('Limit DB connections per user')
4. Change DDMACC network attribute with the command below:
    CHGNETA DDMACC(yourlib/DBSSNLMT1)
5. If default settings do not comply with your needs, add the required environment variable with the required value at the system level with a command similar as below:
    ADDENVVAR ENVVAR(DATABASE_LIMIT_CONCURRENTSESSIONS) VALUE(3) LEVEL(*SYS)
