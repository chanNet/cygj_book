### 接口

目前odoo与外部系统交互大多是基于OCA的connector模块开发，在此基础上的是hrp\_connector，主要是增加一些通用的synchronizer，基于hrp\_connector的有hrp\_connector\_ebs,hrp\_connector\_hrp等模块，根据不同的业务场景选择不同的模块。如果需要连接到对方数据库则需要在对应的backend配置数据库连接信息，会自动生成外部数据源。

在这里，以创业管家与汇联易接口为例,，汇联易是通过rpc方式，发送http请求来提供服务，给了一个[postman](https://www.getpostman.com/)数据，如果对方提供的是http请求或者web\_service形式的，都可以先通过postman手动调试下，不必急着写代码。

注：下例涉及到的代码片段都是不完整的，且不包括通用的synchronizer、Adapter

1、整合汇联易提供的postman，转成python，提供统一的调用方式

```py
# -*- coding: utf-8 -*-

import requests
import logging
from requests.auth import AuthBase, HTTPBasicAuth
from requests.utils import to_native_string
from datetime import datetime

AETEMIS_SUCCESS = {'ok': True}


class HTTPBearerAuth(AuthBase):
    def __init__(self, access_token):
        self.access_token = access_token

    def __call__(self, r):
        r.headers['Authorization'] = 'Bearer ' + to_native_string(('%s' % (self.access_token)))
        return r


class ArtemisAPI(object):
    def __init__(self, service_url, username, password):
        self.service_url = service_url
        self.username = username
        self.password = password
        self.access_token = None
        self.bearer_auth = None

    def _get_error_message(self, response):
        json = response.json()
        message = ''
        if json.get('message', False):
            message = json.get('message')
        if 'validationErrors' in json and not json['validationErrors'] and json.get('errorCode', False):
            message = json.get('errorCode') + '  validationErrors'
        if json.get('validationErrors', False) and isinstance(json['validationErrors'], list) and isinstance(
                json['validationErrors'][0], dict):
            message = json['validationErrors'][0].get('message', False)
        return {'ok': False, 'message': str(response.status_code) + ':' + str(message)}

    def _common_update_response(self, url, val):
        response = requests.put(url, json=val, auth=self.bearer_auth)
        if response.ok:
            return AETEMIS_SUCCESS
        else:
            return self._get_error_message(response)

    def _common_search_response(self, url):
        response = requests.get(url, auth=self.bearer_auth)
        if response.ok:
            return response.json()
        else:
            return self._get_error_message(response)

    def admin_login(self):
        """
        管理员登录
        :return:
        """
        url = self.service_url + "/oauth/token"
        payload = {
            'grant_type': 'client_credentials',
            'scope': 'write',
        }
        auth = HTTPBasicAuth(self.username, self.password)
        response = requests.post(url, data=payload, auth=auth)
        if response.ok:
            self.access_token = response.json().get('access_token')
            self.bearer_auth = HTTPBearerAuth(self.access_token)
            return {'ok': True}
        else:
            return self._get_error_message(response)

    def register_company(self, value):
        """
        创建公司
        :param value: 公司信息{'name':}
        :return:
        """
        url = self.service_url + '/api/implement/companies'
        response = requests.post(url, json=value, auth=self.bearer_auth)
        if response.ok:
            return {'ok': True,
                    'companyOID': response.json().get('companyOID', False)}
        else:
            return self._get_error_message(response)

    def update_company(self, value):
        """
        更新公司信息
        :param value: {'companyOID':,'name':}
        :return:
        """
        url = self.service_url + '/api/implement/companies'
        return self._common_update_response(url, value)
```

2、定义后端

```py
# -*- coding: utf-8 -*-

from openerp.addons.connector import backend

artemis = backend.Backend('artemis')
artemis1 = backend.Backend(parent=artemis, version='1.0')
```

3、定义后端模型

```py
class ArtemisBackend(models.Model):
    _name = 'hrp.connector.artemis.backend'
    _description = 'Artemis Backend'
    _inherit = 'connector.backend'

    _backend_type = 'artemis'

    @api.model
    def _select_versions(self):
        """
        可用的版本
        :return:
        """
        return [('1.0', 'Version 1.0')]

    @api.model
    def _lang_get(self):
        languages = self.env['res.lang'].sudo().search([])
        return [(language.code, language.name) for language in languages]

    @api.model
    def _tz_get(self):
        return [(tz, tz) for tz in sorted(pytz.all_timezones, key=lambda tz: tz if not tz.startswith('Etc/') else '_')]

    name = fields.Char(string='Name', help="Name", required=True)
    version = fields.Selection(selection='_select_versions', string='Version', required=True, default='1.0')
    location = fields.Char(string="Location", help='Url to artemis application', required=True)
    username = fields.Char(string='Username', help="Username", required=True)
    password = fields.Char(string='Password', help="Password", required=True)
    default_lang = fields.Selection(_lang_get, string='Default Language', help='Default Language')
    initialized = fields.Boolean(default=False, copy=False)
    tz = fields.Selection(_tz_get, string='Timezone', help='Timezone', copy=False)
```

* `_backend_type`必须在定义后端里
* 版本也必须在定义后端里

4、接口绑定

```py
class ArtemisBinding(models.AbstractModel):
    _name = 'hrp.connector.artemis.binding'
    _inherit = 'external.binding'
    _description = 'Artemis Binding (abstract)'

    backend_id = fields.Many2one(comodel_name='hrp.connector.artemis.backend', string='Artemis Backend', required=True,
                                 ondelete='restrict')
    artemis_id = fields.Char(string='ID in the Artemis Machine', index=True)
```

5、Environment，获得执行环境的env变量

```py
def get_environment(session, model_name, backend_id):
    """ Create an environment to work with. """
    backend_record = session.env['hrp.connector.artemis.backend'].sudo().browse(backend_id)
    env = ConnectorEnvironment(backend_record, session, model_name)
    lang_code = backend_record.default_lang or 'en_US'
    if lang_code == session.context.get('lang'):
        return env
    else:
        with env.session.change_context(lang=lang_code):
            return env
```

6、检查点，当有新的记录导入的时候可能会有需要检查

```py
def add_checkpoint(session, model_name, record_id, backend_id):
    return checkpoint.add_checkpoint(session, model_name, record_id, 'hrp.connector.artemis.backend', backend_id)
```

7、ConnectorUnit

**Binder**

外围系统的ID和Odoo的ID提供绑定

**Mapper**

外围系统的记录的记录转变成Odoo的记录的映射关系

**BackendAdapter**

适配器用于实现外围系统的API。通常封装为通用接口\(CRUD\)。

**Synchronizer**

用于定义同步流程，导入，导出，删除同步。

8、[同步员工到汇联易](/tong-bu-yuan-gong.md)





