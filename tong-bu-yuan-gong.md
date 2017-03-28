### 同步员工

```
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


@artemis
class ArtemisEmployeeExporter(ArtemisExporter):
    _model_name = ['hrp.connector.artemis.hr.employee']


@artemis
class ArtemisEmployeeDeleter(HrpConnectorArtemisDeleter):
    _model_name = ['hrp.connector.artemis.hr.employee']


@on_record_create(model_names='hr.employee')
def create_hr_employee(session, model_name, record_id, vals):
    artemis_connect.creat_artemis_rec(session, 'hrp.connector.artemis.hr.employee', record_id)


@on_record_write(model_names='hr.employee')
def write_hr_employee(session, model_name, record_id, vals):
    if any(f in vals for f in ['department_id', 'mobile_phone', 'name', 'work_email','employee_number']):
        artemis_connect.artemis_export_delay(session, 'hrp.connector.artemis.hr.employee', record_id)


@on_record_create(model_names='hrp.connector.artemis.hr.employee')
def delay_export_hr_employee(session, model_name, record_id, vals):
    if not session.context.get('initialize_syn', False) and session.context.get('from_backend', False):
        export_record.delay(session, model_name, record_id)


@on_record_unlink(model_names='hr.employee')
def delay_unlink(session, model_name, record_id):
    artemis_connect.delay_unlink(session, 'hrp.connector.artemis.hr.employee', record_id)

```



