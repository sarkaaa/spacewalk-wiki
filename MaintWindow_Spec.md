# Fix TZ with machine



Schema:
add to table [rhnserver](http://www.redhat.com/spacewalk/documentation/schema-doc/table-RHNSERVER.html) reference to table rhntimezone to track in which
timezone is server.
add to table RHNACTION column IN_SERVERTIME: CHAR(1) = 'N', 'Y' means that time should be treated as localtime of target machine, 'N' is meant as localtime of Spacewalk server.

During every rhn_check we will update TZ of server.
All times in action schedule will be treated as GMT. Client should sent all
request in GMT. Server have to sent all actions which should be executed at
this time or before.



|  *Task*                    |  *Size Estimate*  |  *Owner*  |  *Remaining Hours*  |  *Status*  |
| --- | --- |
|  Convert schema from DATE to TIMESTAMP with TIME ZONE  |  4h  |
|  Convert all stored data to GMT  |  8h  |
|  Fix backend to work with new format  |  8h  |
|  Fix java stack to work with new format  |  8h  |
|  update client tools to communicate with time only in GMT  |  20 h  |
|  add checkbox to all scheduled event "Run in server localtime"  |  20 h  |
|  *Total*  |  *68 h*  |
# Create new function which will predict how long it will take to deploy errata



Average errata took 7 seconds to deploy:

    SQL> select min(errata_avg), max(errata_avg), avg(errata_avg) from (select
    ERRATA_ID, avg(86400*(COMPLETION_TIME-PICKUP_TIME)) errata_avg from
    RHNACTIONERRATAUPDATE AE, RHNSERVERACTION SA WHERE AE.action_id=SA.action_id
    group by ERRATA_ID);
    MIN(ERRATA_AVG) MAX(ERRATA_AVG) AVG(ERRATA_AVG)
    --------------- --------------- ---------------
                  0         4302.78       7.2878067




It is not sane to come with some predetermined formula which will calculate
time for errata deployment based on size of errata or number of files. Biggest
factor is postcript which we could not measure untill we deploy at least one
errata.

Additionally: most system deploys in 5 sec and sometime one or several systems
appeared which has deployment time more then 1000 seconds. Reason is unknow to
me.

Some erratas which avg was high due several system with deployment time more then 1000 sec:

    RHBA-2009:1416 (samba)  36 sec
    RHEA-2008:0185 (tzdata) 2 mins
    RHSA-2010:0122 (sudo) 2.5 mins
    RHBA-2009:0245 (initscripts) 1 hour !!


I suggest to count errata deployment time as:

    select
           avg(86400*(COMPLETION_TIME-PICKUP_TIME)) errata_avg
    from
           RHNACTIONERRATAUPDATE AE,
           RHNSERVERACTION SA
    WHERE AE.action_id=SA.action_id
           and COMPLETION_TIME is not null
           and AE.ERRATA_ID= :errata_id
    group by
           AE.ERRATA_ID;
This select is pretty instant.

If no rows are returned use query average errata deployment time for errata on target machine and as last instance we can query deploy time across all erratas:

    select
           avg(86400*(COMPLETION_TIME-PICKUP_TIME)) errata_avg
    from
           RHNACTIONERRATAUPDATE AE,
           RHNSERVERACTION SA
    WHERE
           AE.action_id=SA.action_id
           and COMPLETION_TIME is not null;
This select takes ~1 sec (on sputnik-prod).

And if this return no rows too use hard coded constant 7 sec.


|  *Task*                    |  *Size Estimate*  |  *Owner*  |  *Remaining Hours*  |  *Status*  |
| --- | --- |
|  Create function for estimate prediction  |  4 h  |
# Maintance Window



Create new table:


     CREATE TABLE maintance_window  (
        ID NUMBER(38) NOT NULL,
        CRON_EXPRESSION VARCHAR2(80) NOT NULL,
        DURATION NUMBER(38) NOT NULL, -- in minutes since start of the windows
        PRIMARY KEY (ID),
     );

we will add trigger to this table, which if this record is changed then
EARLIEST_ACTION will be updated too.

add to table [RHNACTION](http://www.redhat.com/spacewalk/documentation/schema-doc/table-RHNACTION.html) new column maintance_window_id which will reference
table maintance_window and can be NULL. If it is null we will use only
EARLIEST_ACTION.

Modify backend xmlrpc call queue.get
  - recognize new version (3)
  - if client request get in version 3 send even actions which are
         a.earliest_action >= sysdate
  - send in xmlblob time when this action will occur, and whether client should execute this action or just prepare for that action
  - if we get response code from code with meaning "End of maintenance window",
    then reschedule this action to next maintenance window. Do this for all
    action for this server and for this maintenance window.

Modify client code
  - client should request queue.get in version 3 (currently it is 2)
  - if action should be executed, do it as we do it today
  - if action occure in future, download packages (probably using "yumdownloader --resolve"),
    download configuration files. Save it to some temporary place.
    Actions which are affected (just guess):
       * packages.update
       * errata.update
       * up2date_config.get
       * up2date_config.update
       * packages.delta
       * rollback.rollback
       * configfiles.deploy
       * kickstart.schedule_sync
       * activation.schedule_pkg_install
       * activation.schedule_deploy
       * solarispkgs.install
       * solarispkgs.patchInstall
       * solarispkgs.patchClusterInstall
       * script.run
       * kickstart_guest.initiate
       * kickstart_host.schedule_virt_host_pkg_install
       * kickstart_guest.schedule_virt_guest_pkg_install
  - when action is executed, check if files has been already fetched.
  - OPTIONAL: when we get information about future action, inform rhnsd when it should execute rhn_check next time (if it is earlier then next chek in).
  - if action should be done in some maintenance window, send end of this
    period and estimated time for this errata.
  - if current time + estimated time for errata > end of maintenance window send
    back to satellite new code with meaning "End of maintenance window"


|  *Task*                    |  *Size Estimate*  |  *Owner*  |  *Remaining Hours*  |  *Status*  |
| --- | --- |
|  schema change  |  4 h  |
|  modify backend  |  20 h  |
|  modify client to comunicate with new semantic of protocol  |  20 h  |
|  prefetch rpm  |  12 h  |
|  prefetch config files  |  8 h  |
|  prefetch scripts  |  4 h  |
|  prefetch kickstarts  |  20 h  |
|  *Total*  |  *88h*  |

Modify WebUI:

add to Satellite Administration new option - default maintenance window. The
code should be the same like in "Task-o-matic has a cron like ability" task.
Plus define default size of this window in minutes.
Later we can add the same option to Organization.
And last step can be to add this option to individual machines.

for scheduling action:
Add new options:
* Do during maintenance window
* Not before X and not later then Y


|  *Task*                    |  *Size Estimate*  |  *Owner*  |  *Remaining Hours*  |  *Status*  |
| --- | --- |
|  modify schema  |  4 h  |
|  enable to specify  maintenance window in admin section in webui  |  4 h  |
|  add to all scheduling form: Do during maintenance window and Not before X and not later then Y  |  20 h  |
|  enable to specify  maintenance window for organization  |  8 h  |
|  enable to specify  maintenance window for individual machines  |  8 h  |
|  * Total *  |  *44 h*  |
# Linking scripts to config deployments



Schema change:

Create new table: 

    RHNCONFIGSCRIPTS:
      CONFIG_FILE_ID: NUMBER(38)
      SCRIPT_ID: NUMBER(38)

which will reference to table [RhnConfigFile](http://www.redhat.com/spacewalk/documentation/schema-doc/table-RHNCONFIGFILE.html) and table RhnScriptLabel.

add new row to [rhnconfigfiletype](http://www.redhat.com/spacewalk/documentation/schema-doc/table-RHNCONFIGFILETYPE.html) with name and label set to "script"

Add new table:

    RHNSCRIPTLABEL:
           ID: NUMBER
           ID_CONFIG_REVISION: NUMBER(38)   -- FK to RHNCONFIGREVISION
           LABEL: VARCHAR(1024)
           POST: NUMBER(1) -- constraint to 0 or 1, 0 means pre script, 1 means
    post script
           CREATED: DATE = (sysdate)
           MODIFIED: DATE = (sysdate)
If we want to be script connected with configuration channel we can utilize
[rhnconfigfile](http://www.redhat.com/spacewalk/documentation/schema-doc/table-RHNCONFIGFILE.html) table, but if we want them to separate we can not use it due
constraint on CONFIG_CHANNEL_ID

When editing/creating revision of config file, create new text box:

    Associate this configuration file with this script:
    
    [x] Pre Script      <- when you tick this check box following form appears
    |-[o] Choose from existing   | labels of existing scripts  |v|
    |-[o] Create new:  <- radio buttons
    |    Run as user: root
    |    Run as group: root
    |    Script label: XXX
    |    Script: XXX
    |    Note: this script will be executed only if Remote Command execution on the
    |    target system is enabled by running "rhn-actions-control --enable-run"
    [ ] Post Script
    ...

When Script Label is not set, pupulate it using Javascript with first line of
script, which begins # and is neither shebang (#!) or ^#\s*$.

we will automatically set filemode to 755.

we will automatically set config_info_id to "script"

Script label will be stored to RHNSCRIPTLABEL.LABEL and content to
RHNCONFIGCONTENT.CONTENTS using RHNCONFIGREVISION as connecting table.

if config file is deployed from WebUI, we will transform asociated script from
[rhnconfigfile](http://www.redhat.com/spacewalk/documentation/schema-doc/table-RHNCONFIGFILE.html) into records in [rhnactionscript](http://www.redhat.com/spacewalk/documentation/schema-doc/table-RHNACTIONSCRIPT.html) table and prerequisite will be
action with config file deployment.

if config file is requested from rhncfg-get we will modify backend to include
new key "script" (configFilesHandler.py:format_file_results and
config_mgmt/rhn_config_management.py: management_get_file) to return id of
script and rhncfg-get should then request this script from satellite.

We can even make suggestion based on file name (probably by javascript/ajax):
 Get name of the package:

    select PN.name from rhnpackagecapability PC, rhnpackagefile PF,
    RHNPACKAGE P,RHNPACKAGENAME PN where PN.id = P.NAME_ID and P.id=PF.package_id
    and PF.capability_id = PC.id and PC.name = '/etc/httpd/conf.d/welcome.conf'
    group by PN.name; }}}
     check for latest version server has available in subscibed channels.
     if package is not in subscribed channels, but is is presented in DB, choose
    latest version of this package.
     check in rhnpackagecapability of this package for file {{{ name LIKE
    '/etc/rc.d/init.d/%' 
 if such file exist, extract value of that service name and suggest script:

     chkconfig foo on
     service foo restart

When configuration file is deployed, check whether this file belongs to some package:

{{{ 
select PN.name from rhnpackagecapability PC, rhnpackagefile PF, RHNPACKAGE P,RHNPACKAGENAME PN 
  where PN.id = P.NAME_ID and 
        P.id=PF.package_id and 
        PF.capability_id = PC.id and
        PC.name = '/etc/redhat-release'
  group by PN.name; 
}}}
and if so, check whether target machine has this package installed. If package is not present suggest user package installation:

{{{ 
[[x]] Schedule installation of package 'redhat-release' before deployment of configuration file /etc/redhat-release 
}}}

For SSM: 


         Schedule installation of package 'redhat-release' before deployment of configuration file /etc/redhat-release:
         Server:                    Install package:
         foo.redhat.com              [x]
         bar.redhat.com              [x]


In detail of configuration channel -> tab List/RemoveFiles add new button
"Asociate with script" where we can associate more files with one script.


|  *Task*                    |  *Size Estimate*  |  *Owner*  |  *Remaining Hours*  |  *Status*  |
| --- | --- |
|  schema change  |  4 h  |
|  add new tab for editing configuration files  |  8 h  |
|  create java classes for saving this data and general manipulation with scripts  |  4 h  |
|  revision handling  |  8 h  |
|  revison handling for SSM  |  16 h  |
|  suggest body of script if conf file belong to service  |  8 h  |
|  suggest installation of package, which owns this config file  |  8 h  |
|  schedule script run if config file is scheduled from webui  |  8 h  |
|  modify backend to send associate script  |  12 h  |
|  mofigy rhncfg-get to retrieve associate script  |  8 h  |
|  guess script name from content  |  4 h  |
|  * Total *  |  * 88 *  |
# Reoccurring remote commands



In Main tab Configuration put to left menu new option "Remote Scripts" so we
will have there:

 * Overview
 * Configuation Channels
 * Configuration Files
 * Remote Scripts
 * Systems

Page "Remote Scripts" will mimic "Configuration Files" page. Table will be:

|  Label  |  attached to configuration file  |
| --- | --- |
|  apache restart  |  [link](1)  |
|  magic script  |  (none)  |

This labels are taken from RHNSCRIPTLABEL table.

When you click on label you get page where you can edit this script. That page
will be copy'n'paste of current page for editing configuration files. With one
difference.
When you save changed content (or attributes).  You will get to page which say


      This script is linked to config files:
      +------------------------------------+
      | /etc/foo                           |
      | /etc/bar                           |
      | /etc/foobar                        |
      +------------------------------------+ 
      Unselect files for which you do not want the scrip changed
    
                    +--------+
                    | Modify |
                    +--------+
All items are selected by default.
If user:
 * leave all selected - new revision of script is made
 * select only some - we will show new page:


       You created new revision of remote script FOO for these configurataion files:
         /etc/foo
         /etc/bar
       Enter new name for this new revision: __________________
                    +------+
                    | Save |
                    +------+

This action will create new Remote Script with this new content and link those
selected config files to this new Remote Script.
## Scheduling Reoccurring remote commands



In SystemDetail ->  Remote Command provide drop down menu which allows you to select existing remote script, but keep option to write new command in text box. The same for SSM.


|  *Task*                    |  *Size Estimate*  |  *Owner*  |  *Remaining Hours*  |  *Status*  |
| --- | --- |
|  implement list of remote scripts  |  3 h  |
|  webui for edit of remote scripts  |  12 h  |
|  handle conflitcs  |  8 h  |
|  add drop down box to "Run Remote Command"  |  4 h  |
|  dtto for SSM  |  4 h  |
|  *Total*  |  *31 h*  |
# API

## Java API

 


Edit existing call:

 * configchannel.getDetails() - provide info if script is linked to this channel.

Crete new api calls:

 * remotecommand.create()
 * remotecommand.list()
 * remotecommand.getDetail()
 * remotecommand.delete()
 * remotecommand.linkToConfigFile()


|  *Task*                    |  *Size Estimate*  |  *Owner*  |  *Remaining Hours*  |  *Status*  |
| --- | --- |
|  api calls  |  20 h  |
## Backend API & actions



If config files is scheduled to deploy check if remote script is linked with
this config and create new record in [RHNACTION](http://www.redhat.com/spacewalk/documentation/schema-doc/table-RHNACTION.html) and [RHNACTIONSCRIPT](http://www.redhat.com/spacewalk/documentation/schema-doc/table-RHNACTIONSCRIPT.html) and
correctly set PREREQUISITE in RHNACTION. So if pre script fails config file is
not deployed.

No need to modify rhn_check as it only pick up scheduled actions and such case
RHNACTIONSCRIPT is populated by Server and rhn_check can already handle this.

We need to modify rhncfg-client. Its mode "get" deploy relevant configuration
files. We need to get and execute associate remote scripts as well.

No need to modify rhncfg-manager as its mode "get" outputs to stdout only.

We will need to create new backend xmlrpc call which allow rhncfg-client fetch
content of remote script for given configuration file.


|  *Task*                    |  *Size Estimate*  |  *Owner*  |  *Remaining Hours*  |  *Status*  |
| --- |
|  none needed, everything was used previously'  |

