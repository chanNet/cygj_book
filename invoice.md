### 后台执行创建发票逻辑

![](/assets/invoice_create.png)

1、后台处理，点击确认需要做一些分组等耗时间的逻辑，所有在此处就用execute\_job执行后台任务，结束前台等待

```py
session = ConnectorSession(self.env.cr, self.env.uid, context=self.env.context)
execute_job.delay(session, 'stock.move.invoice.line', 'delay_create_invoice',
                                     self.is_combine_invoice, active_ids)
```

```py
@api.model
def delay_create_invoice(self, is_combine_invoice, active_ids):
    flag, group_moves, category_ids = self._group_by_partner(is_combine_invoice, active_ids)
    if flag:
        journal_id = self._get_journal('purchase')
        inv_type = 'in_invoice'
    else:
        journal_id = self._get_journal('purchase_refund')
        inv_type = 'in_refund'
    self._make_invoice_delay(category_ids, group_moves, journal_id, inv_type)

@api.multi
def _make_invoice_delay(self, category_ids, group_moves, journal_id, inv_type):
    packing_obj = self.env['stock.picking']
    move_obj = self.env['stock.move']
    for key, ids in group_moves.iteritems():
        moves = move_obj.browse(ids)
        vendor_category_ids = category_ids.get(key, [])
        packing_obj._invoice_create_line(moves, journal_id.id, inv_type,
                                         vendor_category_ids=vendor_category_ids, background=True)
```

2、执行后台批量任务

```py
@api.model
def _invoice_create_line(self, moves, journal_id, inv_type='out_invoice', vendor_category_ids=[], background=False):
    self = self.with_context(no_check=True)
    session = ConnectorSession(self.env.cr, self.env.uid, context=self.env.context)
    if not moves:
        return
    product_price_unit = {}
    line_number = 1
    invoice_id = False
    todo = {}
    picking_todo = self
    status = self.env['stock.move'].get_invoice_status_dict()
    for move in moves:
        company = move.company_id
        order_id = move.origin_purchase_order_line_id.order_id
    partner, user_id, currency_id = move.vendor_name, self.env.uid, order_id.currency_id.id
    key = (partner, currency_id, company.id, user_id)
    invoice_vals = self._get_invoice_vals(key, inv_type, journal_id, move)

    vendor_category_id = order_id.vendor_category_id.id

    invoice_vals.update({'vendor_category_id': vendor_category_id,
                         'vendor_category_ids': [(6, 0, vendor_category_ids)],
                         'currency_id': currency_id,
                         'origin': False,
                         'need_process': background})

    if partner.bank_ids and partner.bank_ids[0].id:
        partner_bank_id = partner.bank_ids[0].id
        invoice_vals.update({'partner_bank_id': partner_bank_id})

    if not invoice_id:
        invoice_id = self._create_invoice_from_picking(move.picking_id, invoice_vals)
    if background:
        # 后台批量执行，先创建发票头，再批量并发创建发票行，在这里job_res_id是发票的ID，因为需要这些子任务跑完后对发票进行处理，
        # 如汇总行上的金额，生成负债分配，主要是这些逻辑需要发票真正创建完成了
        batch_execute_job.delay(session, 'stock.move', '_generate_invoice_line', moves.ids,
                                'stock.picking._process_invoice', 100, invoice_id, inv_type, partner.id,
                                job_res_id=invoice_id)
        return True
    invoice_line_vals = self._prepare_invoice_line(invoice_id, move, partner, inv_type, line_number=line_number)
    line_number += 1
    is_extra_move, extra_move_tax, default_move_tax = move._get_taxs_by_move(inv_type)
    if not is_extra_move:
        product_price_unit[invoice_line_vals['product_id'], invoice_line_vals['uos_id']] = invoice_line_vals[
            'price_unit']
    if is_extra_move and (invoice_line_vals['product_id'], invoice_line_vals['uos_id']) in product_price_unit:
        invoice_line_vals['price_unit'] = product_price_unit[
            invoice_line_vals['product_id'], invoice_line_vals['uos_id']]
    if is_extra_move:
        desc = (inv_type in (
            'out_invoice', 'out_refund') and move.product_id.product_tmpl_id.description_sale) or \
               (
                   inv_type in (
                   'in_invoice', 'in_refund') and move.product_id.product_tmpl_id.description_purchase)
        invoice_line_vals['name'] += ' ' + desc if desc else ''
        if extra_move_tax:
            invoice_line_vals['invoice_line_tax_id'] = extra_move_tax
        # the default product taxes
        elif default_move_tax:
            invoice_line_vals['invoice_line_tax_id'] = default_move_tax
    todo[move.id] = {'line_val': invoice_line_vals, 'move_val': move._get_move_val_invoice(status)}
    if move.picking_id and move.picking_id not in picking_todo:
        picking_todo += move.picking_id
    moves.create_invoice_line_for_move(todo)
    picking_todo.recompute_invoice_state()
    self._process_invoice(invoice_id, vendor_category_ids, False)
    return [invoice_id]
```

3、done\_method 的\_process\_invoice

```py
@api.model
def _process_invoice(self, invoice_id=[], vendor_category_ids=[], background=True):
    # 这里的invoice_id是通过done_method的job_res_id传值
    self = self.with_context(no_check=True).suspend_security()
    invoice_obj = self.env['account.invoice']
    invoices = invoice_obj.browse(invoice_id)
    invoice_distribution_line_obj = self.env['hrp.account.invoice.distribution.line']
    for invoice in invoices:
        if background and not vendor_category_ids:
            vendor_category_ids = invoice.vendor_category_ids.ids
        invoice.button_compute()
        if len(vendor_category_ids) > 1:
            for vci in vendor_category_ids:
                invoice_distribution_line_amount = 0
                for l in invoice.invoice_line:
                    tax_amount = l.get_tax_amount()
                    top_category_id = invoice_obj._get_top_category_id(l.product_id.categ_id)
                    if top_category_id.id == vci:
                        invoice_distribution_line_amount += l.price_subtotal + tax_amount
                invoice_distribution_line_obj.create({
                    'invoice_id': invoice.id,
                    'vendor_category_id': vci,
                    'liability_account_id': invoice.account_id.id,
                    'amount': invoice_distribution_line_amount
                })
        elif len(vendor_category_ids) == 1:
            invoice_distribution_line_obj.create({
                'invoice_id': invoice.id,
                'vendor_category_id': vendor_category_ids[0],
                'liability_account_id': invoice.account_id.id,
                'amount': invoice.check_total
            })
        if background:
            line_number = 1
            for line in invoice.invoice_line:
                self._cr.execute("UPDATE account_invoice_line set line_number = %s WHERE id = %s",
                                 (line_number, line.id))
                line_number += 1
            self._cr.execute("UPDATE account_invoice set need_process = %s WHERE id = %s",
                             (False, invoice.id))
            self.invalidate_cache()
            invoice.invoice_line.mapped('stock_move_id.picking_id').recompute_invoice_state()
    return invoices
```



