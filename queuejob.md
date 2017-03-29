### 后台执行

在odoo里，有些逻辑大批量数据时很慢，前台浏览器无法及时收到服务器的反馈导致超时，这时用queue.job的方式后台并发执行，既能避免前台超时，也能并发执行。目前，这些后台执行的代码是写在hrp\_connector中，下面的代码和例子均在HRP 3.0-PUTH分支

```py
# -*- coding: utf-8 -*-

from openerp.addons.connector.queue.job import job


@job(default_channel='root')
def batch_execute_job(session, model_name, method, record_ids, done_method=None, job_size=100, *args, **kwargs):
    """
    批量执行任务
    :param session: 会话
    :param model_name: 模型名
    :param method: 执行方法
    :param record_ids: 记录ids
    :param done_method: 待执行方法
    :param job_size:
    :param args:
    :param kwargs:
    :return:
    """
    queue_job_obj = session.env['queue.job']
    job_uuids = []
    val = {}
    if kwargs.get('job_res_id', False):
        # 可以为done_method指定参数
        val['res_id'] = kwargs.pop('job_res_id')
    if len(record_ids) > job_size:
        index = 0
        while index < len(record_ids):
            next = index + job_size
            ids = (record_ids[index:next])
            job_uuids += [execute_job.delay(session, model_name, method, ids, *args, **kwargs)]
            index = next
    else:
        job_uuids += [execute_job.delay(session, model_name, method, record_ids, *args, **kwargs)]
    # 建立job的父子关系
    queue_jobs = queue_job_obj.suspend_security().search([('uuid', 'in', job_uuids)])
    batch_job = queue_job_obj.suspend_security().search([('uuid', '=', session.env.context.get('job_uuid'))])
    if done_method:
        val['done_method'] = done_method
    if val:
        batch_job.suspend_security().write(val)
    queue_jobs.suspend_security().write({'parent_id': batch_job.id})


@job(default_channel='root')
def execute_job(session, model_name, method, record_ids, *args, **kwargs):
    model = session.pool[model_name]
    getattr(model, method)(session.cr, session.uid, record_ids, *args, **kwargs)
```

上面的**done\_method**会在批量任务的所有子任务完成后执行，定义了一个安排的动作ir\_cron，一分钟一次

![](/assets/ir_cron.png)

相关代码如下

```py
@api.model
def execute_job_done_method(self):
    job_obj = self.env['queue.job']
    undone_jobs = self.search(
        [('state', '=', 'done'), ('is_done', '=', False), ('parent_id', '=', False), ('child_ids', '!=', False)])
    done_jobs = job_obj
    child_jobs = job_obj
    for job in undone_jobs:
        if all(child.state in ['done', 'failed'] for child in job.child_ids):
            if job.done_method:
                model_name, func_name = job.done_method.rsplit('.', 1)
                model = self.env[model_name].with_context(batch_job_id=job.id)
                try:
                    with self.env.cr.savepoint():
                        if job.res_id:
                            getattr(model, func_name)(job.res_id)
                        else:
                            getattr(model, func_name)
                        done_jobs += job
                        child_jobs += job.child_ids
                except Exception, e:
                    _logger.error(e)
            else:
                done_jobs += job
                child_jobs += job.child_ids
    done_jobs.write({'is_done': True})
```

例子：[发票匹配](/invoice.md)

