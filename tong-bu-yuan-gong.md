### 同步员工

generic.py

```py
from .unit.delete_synchronizer import export_delete_record
from .unit.export_synchronizer import export_record

# 通用的创建时同步方法
def creat_artemis_rec(session, artemis_model, openerp_id):
    backends = session.env['hrp.connector.artemis.backend'].suspend_security().search([])
    for backend in backends:
        if backend.initialized:
            session.env[artemis_model].suspend_security().with_context(from_backend=True).create({
                'backend_id': backend.id,
                'openerp_id': openerp_id
            })

# 通用的修改时同步方法
def artemis_export_delay(session, artemis_model, record_id):
    artemis_recs = session.env[artemis_model].suspend_security().search([('openerp_id', '=', record_id)])
    for rec in artemis_recs:
        export_record.delay(session, artemis_model, rec.id)

# 通用的删除时同步方法
def delay_unlink(session, artemis_model, record_id):
    """ Delay a job which delete a record on artemis.

    Called on binding records."""
    for rec in session.env[artemis_model].suspend_security().search([('openerp_id', '=', record_id)]):
        export_delete_record.delay(session, artemis_model,
                                   rec.backend_id.id, rec.artemis_id)
```

```py
# -*- coding: utf-8 -*-

from openerp.addons.connector.event import (on_record_write,
                                            on_record_create,
                                            on_record_unlink
                                            )
from openerp.addons.connector.unit.mapper import (mapping,
                                                  ExportMapper)
from openerp import models, fields, api, _
from openerp.exceptions import ValidationError
import generic as artemis_connect
from .backend import artemis
from .unit.backend_adapter import GenericAdapter
from .unit.export_synchronizer import export_record, ArtemisExporter
from .unit.delete_synchronizer import HrpConnectorArtemisDeleter

# 声明员工接口绑定，类似中间表
class ArtemisHrEmployee(models.Model):
    _name = 'hrp.connector.artemis.hr.employee'
    _inherit = 'hrp.connector.artemis.binding'
    _inherits = {'hr.employee': 'openerp_id'}

    openerp_id = fields.Many2one(comodel_name='hr.employee', string='Employee', required=True,
                                 ondelete='cascade')

    @api.one
    @api.constrains('backend_id', 'openerp_id')
    def _constrains_unique_backend(self):
        if self.suspend_security().search_count(
                [('backend_id', '=', self.backend_id.id), ('openerp_id', '=', self.openerp_id.id)]) > 1:
            raise ValidationError(_('A Artemis binding for this employee already exists.'))


class HrpHrEmployee(models.Model):
    _inherit = 'hr.employee'

    artemis_bind_ids = fields.One2many(comodel_name='hrp.connector.artemis.hr.employee', inverse_name='openerp_id',
                                       string='Artemis Bindings')
    department_id = fields.Many2one(comodel_name='hr.department', ondelete='restrict')

# 声明员工专用adapter，在create、write、delete调用artemis_api
@artemis
class HrEmployeeAdapter(GenericAdapter):
    _model_name = 'hrp.connector.artemis.hr.employee'
    _artemis_model = 'artemis_employee'

    def create(self, data):
        res = self.artemis.register_user(data)
        if res.get('ok'):
            return res.get('userOID')
        else:
            raise Exception(res.get('message'))

    def write(self, external_id, data):
        data.pop('password')
        data['userOID'] = external_id
        res = self.artemis.update_user(data)
        if not res.get('ok'):
            raise Exception(res.get('message'))

    def delete(self, external_id):
        res = self.artemis.unlink_user(external_id)
        if not res.get('ok'):
            raise Exception(res.get('message'))

# 声明导出时的mapper, direct是直接的映射，左边是odoo，右边是汇联易，如需要逻辑处理再映射可用mapping修饰器
@artemis
class ArtemisDepartmentExportMapper(ExportMapper):
    _model_name = 'hrp.connector.artemis.hr.employee'

    direct = [('name', 'fullName'),
              ('mobile_phone', 'mobile'),
              ('mobile_phone', 'password'),
              ('work_email', 'email')]

    @mapping
    def company(self, record):
        return {'companyOID': self.backend_record.artemis_company}

    @mapping
    def employee_number(self, record):
        return {'employeeID': record.employee_number or ''}

    @mapping
    def department(self, record):
        department_id = self.env['hrp.connector.artemis.hr.department'].suspend_security().search(
            [('openerp_id', '=', record.department_id.id), ('backend_id', '=', self.backend_record.id)], limit=1)
        return {'departmentOID': department_id.artemis_id or ''}

# 声明exporter
@artemis
class ArtemisEmployeeExporter(ArtemisExporter):
    _model_name = ['hrp.connector.artemis.hr.employee']

# 声明deleter
@artemis
class ArtemisEmployeeDeleter(HrpConnectorArtemisDeleter):
    _model_name = ['hrp.connector.artemis.hr.employee']

# 创建时触发，一般是在里面触发中间表的创建
@on_record_create(model_names='hr.employee')
def create_hr_employee(session, model_name, record_id, vals):
    artemis_connect.creat_artemis_rec(session, 'hrp.connector.artemis.hr.employee', record_id)

# 修改时触发，connector自带
@on_record_write(model_names='hr.employee')
def write_hr_employee(session, model_name, record_id, vals):
    if any(f in vals for f in ['department_id', 'mobile_phone', 'name', 'work_email','employee_number']):
        artemis_connect.artemis_export_delay(session, 'hrp.connector.artemis.hr.employee', record_id)

# 由员工表的创建触发
@on_record_create(model_names='hrp.connector.artemis.hr.employee')
def delay_export_hr_employee(session, model_name, record_id, vals):
    if not session.context.get('initialize_syn', False) and session.context.get('from_backend', False):
        export_record.delay(session, model_name, record_id)

# 删除时触发，connector自带
@on_record_unlink(model_names='hr.employee')
def delay_unlink(session, model_name, record_id):
    artemis_connect.delay_unlink(session, 'hrp.connector.artemis.hr.employee', record_id)
```



