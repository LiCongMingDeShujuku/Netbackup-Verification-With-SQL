![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# SQL中的Netbackup验证
#### Netbackup Verification With SQL

![#](images/Netbackup-Verification-With-SQL-01.png?raw=true "#")

## Contents

- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


![#](images/Netbackup-Verification-With-SQL-01.png?raw=true "#")

 
You can get a list of all the files that Netbackup has successfully copied by using the BPLIST utility. Ordinarilly this is run from Powershell or Command prompt, but it can be run using SQL, and the results can be pulled into a table, and you can build a process around it if needed. Thats exactly what I did here.

你可以使用BPLIST实用程序获取Netbackup已成功复制的所有文件的列表。通常，这是用Powershell或Command提示符运行的，但它可以使用SQL运行，结果可以拉入表中，如果需要，你可以围绕它构建一个程序。 这正是我在这里所做的。

![#](images/Netbackup-Verification-With-SQL-02.png?raw=true "#")
 
1	exec master..xp_cmdshell 'C:\NetBackupList\bplist -R 99 -C MyServerName.MyDomain.com -s 06/23/2018 00:00:00 -e 07/24/2018 00:00:00 -I "F:\BACKUPS"'
In order for this backup process to work you’ll need to copy the Netbackup BPLIST utility to a root folder. In this example I am using the root of C:. I created a folder called NetBackupList, and there is where I copied the BPLIST utility and all the corresponding DLL’s.
You can see the BPLIST.exe utility (in red) along with all the DLL’s (in blue). Again; you can use any folder you want, but it would be much easier if it’s a root folder, and the folder name has no spaced. This probably shouldn’t be in Drive C:, but in this example it works well.
In this example; the BPLIST utility is found here under:
C:\Program Files\VERITAS\NetBackup\bin

为了使此备份过程正常工作，需要将Netbackup BPLIST实用程序复制到根文件夹。 在这个例子中，我使用的是C：的根。 我创建了一个名为NetBackupList的文件夹，我在那里复制了BPLIST实用程序和所有相应的DLL。
你可以看到BPLIST.exe实用程序（红色）以及所有DLL（蓝色）。你可以使用任何你想要的文件夹，但如果它是一个根文件夹，并且文件夹名称没有间隔会更容易。 这可能不应该在Drive C：中，但在这个例子中效果很好。
在这个例子中，在这里可以找到BPLIST实用程序：
C:\Program Files\VERITAS\NetBackup\bin

![#](images/Netbackup-Verification-With-SQL-03.png?raw=true "#")

 
Here’s all the files you’ll need to copy in order for the BPLIST.exe utility to work.

以下是你需要复制的所有文件，以便BPLIST.exe实用程序正常工作。

bplist.exe
libnbbase.dll
libnbclient.dll
vrtsLogFormatMsg_3.dll
vrtslogread_3.dll
vxACE_6.dll
vxcPBX.dll
vxicudata_6.dll
vxicui18n_6.dll
vxicuuc_6.dll
vxlis_3.dll
vxul_3.dll

![#](images/Netbackup-Verification-With-SQL-04.png?raw=true "#")
 
When writing this process the firs thing you need to do is parameterize the BPLIST statement by creating variables that you can plug into it. You can see how I’m doing that below. The next thing is create a series of lists. A list of all backups that have occurred through SQL internal tables which is captured from BackupMediaFamily (splitting the path name from the file name), and using the path to run the BPLIST utility against every known backup path, and then taking the results of that and comparing the path list that BPLIST shows you against the database backups you are holding locally. Next you concatinate a Delete statement to run against only the files that were confirmed (using the BPLIST output), and you have a cleanup script which removes only the files that have already been backed up by Netbackup.

在编写此过程时，你需要做的第一件事是通过创建可插入其中的变量来参数化BPLIST语句。你可以在下面看到我是怎么做的。接下来是创建一系列的列表。通过从BackupMediaFamily捕获的SQL内部表生成一个所有备份的列表。使用路径运行BPLIST实用程序，与所有已知的
路径作对比。然后获取结果 并将BPLIST显示的路径列表与你在本地保存的数据库备份进行比较。接下来，你通过一个Delete语句仅运行已确认的文件（使用BPLIST输出），并且你有一个清除脚本，该脚本仅删除已由Netbackup备份过的文件。



---
## Logic
```SQL
use master;
set nocount on
 
declare     @fqdn_server_name   varchar(255)
declare     @tomorrow       varchar(255)
declare     @30_days        varchar(255)
declare     @file_removal       table ([del_statement] varchar(555))
set     @fqdn_server_name   = (select cast(serverproperty('machinename') as varchar) + '.' + lower(default_domain()) + '.com')
set     @tomorrow       = (select convert(varchar, getdate() + 1, 101)  + ' 00:00:00')
set     @30_days        = (select convert(varchar, getdate() - 30, 101) + ' 00:00:00')
--create temp table to capture list of all confirmed netbackup file copies.
--创建临时表以捕获所有已确认的netbackup文件副本的列表。
if object_id('tempdb..##netbackup_confirmed') is not null
    drop table  ##netbackup_confirmed
create table    ##netbackup_confirmed ([file_name] varchar(255))
 
--find all backup path and backup files for the last 30 days.
--查找过去30天的所有备份路径和备份文件
declare     @duration   datetime
set     @duration   = (select getdate() - 30)
declare     @backup_history table ([location] varchar(255), [backup_file] varchar(255))
insert into @backup_history
select
    'location'          = reverse(right(reverse(upper(bmf.physical_device_name)), len(bmf.physical_device_name) - charindex('\',reverse(bmf.physical_device_name),1) + 1))
,   'backup_file'       = right(bmf.physical_device_name, charindex('\', reverse('\' + bmf.physical_device_name)) - 1)
from        msdb.dbo.backupset bs join msdb.dbo.backupmediafamily bmf on bs.media_set_id = bmf.media_set_id
where       bs.backup_finish_date > @duration
        and bmf.[device_type] not in ('7')
group by    bs.database_name,   bs.backup_finish_date, bmf.physical_device_name, bs.type, bmf.device_type
order by    bs.database_name,   bs.backup_finish_date desc
 
--build Netbackup BPLIST logic for all known backup paths, and corresponding backup files.
--为所有已知备份路径和相应的备份文件构建Netbackup BPLIST逻辑。
declare @bplist_all     table ([bp_commands] varchar(max))
insert into @bplist_all
select distinct
'insert into ##netbackup_confirmed exec master..xp_cmdshell ''C:\NetBackupList\bplist -R 99 -C ' + @fqdn_server_name + ' -s ' + @30_days + ' -e ' + @tomorrow + ' -I "' + reverse(stuff(reverse([location]), 1, 1, ''))  + '"'''
from @backup_history
 
--execute Netbackup BPLIST utility across all unique backup paths gathering all confirmed files copied to enterprise storage and place them into table ##netbackup_confirmed.
--在所有唯一备份路径上执行Netbackup BPLIST实用程序，收集复制到企业存储的所有已确认文件，并将它们放入表## netbackup_confirmed中。
declare @populate_nbc       varchar(max)
set @populate_nbc       = ''
select  @populate_nbc       = @populate_nbc + 
[bp_commands]
from    @bplist_all
exec    (@populate_nbc)
 
-- get manual confirmation.  run query against @bplist_all and get the BPLIST command for any drive on the server.
-- run the following command with the target drive letter you're looking for.
--手动确认。 对@bplist_all运行查询并获取服务器上任何驱动器的BPLIST命令。
--使用你要查找的目标驱动器号运行以下命令。

select replace([bp_commands], 'insert into ##netbackup_confirmed ', '') 
from @bplist_all where [bp_commands] like '%y:\%'

 
--created file removal process for only files that have successfully backed up to the enterprise.
--为已成功备份到企业的文件创建文件删除程序。
insert into @file_removal
select 'exec master..xp_cmdshell ''DEL "' + [location] + [backup_file] + '"'';' from @backup_history where [location] + [backup_file] in (select [file_name] from ##netbackup_confirmed)
order by [backup_file] desc
 
--remove backups that thave already been moved to enterprise storage.
--删除已移至企业存储的备份。
declare @delete_files   varchar(max)
set @delete_files = ''
select  @delete_files = @delete_files + 
[del_statement] + char(10) 
from    @file_removal
exec    (@delete_files)


```



[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

