### 后台执行创建发票逻辑

![](/assets/invoice_create.png)

1、后台处理，点击确认需要做一些分组等耗时间的逻辑，所有在此处就用execute\_job执行后台任务，结束前台等待

```py
session = ConnectorSession(self.env.cr, self.env.uid, context=self.env.context)
execute_job.delay(session, 'stock.move.invoice.line', 'delay_create_invoice',
                                     self.is_combine_invoice, active_ids)
```



