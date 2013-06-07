=====================================
利用django-crontab來佈署系統cron jobs
=====================================

安裝 django-crontab Django application
---------------------------------------------------
::

    $ pip install django-crontab

settings.py 設定
----------------
::

    CRONJOBS = [   
            ('58 23 * * *', 'family.cron.generate_trans_report_job', '>> %s 2>&1' % os.path.join(_PATH, os.path.pardir, 'family_cronjob.log')),                     
    ]

cron function 範例
------------------
  - cron job related model::

        class Cron_Job_Log(models.Model):
        STATUS_CHOICES = (
            ('1', _('cron_job_success')),
            ('2', _('cron_job_failed')),
        )
        title = models.CharField(max_length=48, default='', verbose_name=_('cron_job_title'))
        exec_time = models.DateTimeField(auto_now_add=True, verbose_name=_('cron_job_exec_time'))       
        status_code = models.CharField(default='1', max_length=1, choices=STATUS_CHOICES, verbose_name=_('cron_job_status'))
        def success(self):
            self.status_code = '1'
        def failed(self):
            self.status_code = '2'
 
  - cron job python function::

        logger = logging.getLogger('family.cronjob')
        #
        #
        def generate_trans_report_job():
            translation.activate('zh_TW')
            job = Cron_Job_Log()
            job.title = generate_trans_report_job.__name__  
            try:
                #your logic here
        
            except: 
                logger.warning("Error when perform cron job %s" % sys._getframe().f_code.co_name, exc_info=1)
                job.failed()
                # raise again
                raise
            else:       
                job.success()
            finally:
                job.save()
                translation.deactivate()
    
  - execution
    
    - add the cron jobs to the system crontab::

      $ python manage.py crontab add

    - linux command to list system crontab::

      $ crontab -u your_username -l

    - linux command to remove system crontab::

      $ crontab -u your_username -r
