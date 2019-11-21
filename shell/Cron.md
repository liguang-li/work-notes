# Cron任务在Linux中计划和自动化任务

## Cron任务用于“自动执行的任务”，有助于简单重复的任务的执行。Cron是一个守护进程，可让你安排这些任务，然后按指定的时间间隔执行这些任务。

* 系统范围的Crontab文件

  /etc/crontab,只能由root用户访问和编辑。通常用于配置系统范围的守护进程。

  ~~~
  [root@centos-8 ~]# cat /etc/crontab
  SHELL=/bin/bash
  PATH=/sbin:/bin:/usr/sbin:/usr/bin
  MAILTO=root
  # For details see man 4 crontabs
  
  # Example of job definition:
  # .-------------------------- minute (0 - 59)
  # |  .----------------------- hour (0 - 23)
  # |  |  .-------------------- day of month (1 - 31)
  # |  |  |  .----------------- month (1 -12) or jan,feb,mar,...
  # |  |  |  |  .-------------- day of week (0 - 6) or sun,mon,tue,wed,thu, fri,sat
  # *	 *	*  *  *  user-name  command to be executed
  ~~~

* 用户crontab文件

  Linux用户还可以在crontab命令的帮助下创建自己的cron任务，创建的cron任务将以创建它们的用户身份运行。

  cron守护进程在后台静默的检测/etc/crontab, /var/spool/cron及/etc/cron.d/*目录。

  crontab命令用于编辑cron文件。

  ## crontab

  ### 管理cron任务

  ~~~
  crontab -e	 #edit the /etc/crontab file
  crontab -u username -e
  crontab -l 	 #list 
  crontab -r   #del all the cron tasks
  ~~~

  \*: 匹配所有记录。

  @hourly	 == 0 * * * * command

  @daily		== 0 0 * * * command

  @weekly	== 0 0 1 * mon command

  @monthly	== 0 0 1 * *

  @yearly 		== 0 0 1 1 *

* 限制crontab

  控制谁有权使用crontab命令。

  /etc/cron.deny

  /etc/cron.allow

* 检查cron日志

  /var/log/cron