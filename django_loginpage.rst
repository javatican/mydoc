===================
Django 登入畫面範例
===================

settings.py
-------------
::
 
    LOGIN_URL = reverse_lazy('login')
    LOGIN_REDIRECT_URL=reverse_lazy('after_login_main_page')

備註: 
    1. LOGIN_REDIRECT_URL設定登入成功後要redirect過去的url(若沒有指定next參數時)
    2. LOGIN_URL及LOGIN_REDIRECT_URL在Django1.5版之後, 可以使用named URL patterns

urls.py
---------
- 可以利用django auth module所提供的login, logout view functions
  
  - login()可以傳入template_name, authentication_form等
    
    - template_name: 指定要回傳的html template
    - authentication_form: login template所要搭配使用的form class, 預設使用 **AuthenticationForm**

  - logout()可以傳入next_page, template_name 等參數
    
    - next_page: 指定要redirect去的url (higher priority than 'template_name')
    - template_name: 指定要回傳的html template

  ::

    from django.contrib.auth.views import login, logout
    url(r'^login/$', login, {'template_name':'family/admin_login.html',}, name='login'),
    url(r'^logout/$', logout,  {'next_page': '/family/login/'}, name='logout'),
 
- AuthenticationForm, 包含username, password兩個fields

View functions need login protection
-------------------------------------
- use **@login_required** decorator, 來標示view functions 需要先登入才能被呼叫::

    @login_required
    def update_member_adv_class_setting(request):
        pass

- 若使用者尚未登入, 
 
  - 則會redirect到settings.py中所設定的登入畫面網址, 亦即 **LOGIN_URL**
  - 並會在url後加上 **next** 參數, 作為登入後要redirect去的url
  - login()會讀入next參數, 並將其寫入context中, 供template取得該參數
  
Login template 範例
-------------------
- 記得<form>的action屬性要使用 **next={{next}}** , 以傳入登入成功後要執行的view function
- username,password的驗證實際是在form.clean()中進行, 因此當登入失敗時, login()會將錯誤訊息寫入到 **form.non_field_errors** 中

::

    {% if form.non_field_errors  %}
      {{ form.non_field_errors }}
    {% endif %}

    <form method="post" id="admin_login_form" action="{% url login %}?next={{next}}">
	<div align="center">{% csrf_token %}</div>
        <table>
            <tr><td/><td colspan="3"><span class="function_title">{% trans "請登入:" %}</span></td></tr>
            <tr>
            <td width="80"  >&nbsp;</td>
            <td width="100"  >{{ form.username.label_tag }} :</td>
              <td width="400">{{ form.username }}</td>
              <td width="210">{{ form.username.errors }}</td>
            </tr>
            <tr>
            <td >&nbsp;</td>
            <td >{{ form.password.label_tag }} :</td>
            <td>{{ form.password }}</td>
            <td>{{ form.password.errors }}</td>
            </tr>
            <tr>
            <td>&nbsp;</td>
            <td>&nbsp;</td>
              <td height="50"><button type="submit" id="admin_login_action" >{% trans "登入" %}</button></td>
              <td>&nbsp;</td>
            </tr>
        </table>
    </form>
