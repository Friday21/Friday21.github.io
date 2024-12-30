---
layout:     post
title:      xadmin中的view
date:       2018-03-23
author:     "FridayLi"
catalog: true
tags:
  - Python
  - Django
---

这篇文章主要是想弄明白xadmin中几个重要页面的加载流程， 每个页面对应的template和view， 以及view之间的继承关系。会涉及到以下几点： 

*  view 的继承关系
*  每个view对应的template以及每个template的大概内容
*  列表页、详情页、新增页、更新页的页面加载逻辑
*  每个页面对应的权限控制

这篇文章不会讲但在xadmin中也很重要的几点：

*  plugin
*  widget
*  form layout

## 1. 继承关系
![描述](/img/old-post/7d2f5c5a8e89c96b51a328212dc573a16582.PNG)


## 2. 几个基础view的属性和方法

### 1. BaseAdminObject
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
get_view | func | 获取view
get_model_view | func | 获取model view
get_admin_url  |  func | 根据view的name获取url
get_model_url | func | 根据model和对应view的name获取url
get_model_perm | func | 根据name获取对应的modle权限名称
has_model_perm | func | 检查当前user是否拥有对应name的权限， name:add、delete...
message_user | func | 完成某项操作后， 用来在页面上推送提示信息
template_response | func | 根据指定模板和context， 返回渲染结果
render_response | func | 返回content内容
get_form_params | func | 将url参数组装成form参数
get_query_string | func | 拼接url参数

### 2. BaseAdminView(不再重复写父类的属性和方法)
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
base_template | attr | 模板， 默认值为base.html, 内容很少
need_site_permission | attr | 查看该view是否需要权限
as_view | func | dispatch http request
init_request | func | request 初始化
init_plugin | func | 插件初始化
get_context | func | 获取用来渲染模板的context
media | attr | view 页面中的js、 css资源
get_media | func | 获取media


### 3. CommAdminView
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
base_template | attr | 模板， 默认值是base_site.html, 带有menu和nav_bar
menu_template | attr | 菜单栏模板
site_title | attr | 网站标题, 默认Django Xadmin
site_footer | attr | 页脚， powered by (my company.inc)
global_models_icon | attr | 全局model的icon设置， 是个字典， 可以指定某个model的icon
default_model_icon | attr | 默认的model的icon
apps_lable_title | attr | app对应的title
apps_icons | attr | 字典， 用来指定某个app的icon
get_site_menu | func | 获取网站菜单
get_nav_menu | func | 获取网站导航的菜单， 比较重要， 下面会讲
get_context | func | context新添加了一些菜单、页脚、标题等内容
get_model_icon | func | 获取model的icon
get_breadcrumb | func | 获取面板栏的标题、url等

CommAdminView的模板是base_site.html, 这个页面会有菜单栏，view中通过get_nav_menu从site的_registry中获取注册的model的ListAdminView页面url，组装成一个列表，返回给template. 列表的格式如下：
```python
[{
        'title': 'Material',  # app 名称
        'menus': [{   #app下的menu
                'title': '相框',  # 一个model的admin view名称
                'url': '/admin/material/album/',  # 该model的列表页 url
                'icon': None,
                'perm': 'material.view_album ',  # 查看该页面需要的权限
                'order ': 5}, 
                {'title': '贴纸'  # 另一个model的admin View
                'url': '/ admin / material / sticker / ', 
                icon ': None, '
                perm 'material.view_sticker',
                'order': 6
            }],
        'first_url': '/admin/material/album/'  # 这个app对应的首页
    }, {
        'title': '管理',  # 另一个app
        'menus': {
            'title': '日志',
            'url': '/admin/xadmin/log/',
            'icon': 'fa fa-cog',
            'perm': 'xadmin.view_log',
            'order': 8
        }
        'first_icon': 'fa fa-cog',
        'first_url': '/admin/xadmin/log/'
    },
```

### 4.  ModelAdminView
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
fields | attr | form展示的字段
exclude | attr | form 不展示的字段
ordering | attr | 排序的字段
model | attr | 对应的model
remove_permissions | attr | [], 去掉指定的权限
get_context | func | 新添加了model的名字、icon等信息
get_breadcrumb | func | title 用model的verbose_name_plural
get_object | func | 根据id获取对应model的object
get_object_url | func| 获取某个object的详情页或更新页的权限
model_admin_url | func | 根据name获取model相对应的url
get_model_perms | func | 返回model对应的add、view、change、delete的权限代码
get_template_list | func | 根据模板名称查找对应的路径
get_ordering | func | 获得排序字段
queryset | func | 获得model默认的query_set
has_view_permission | func | 当前user是否拥有view权限
has_chagne_permission | func | 当前user是否拥有编辑权限
has_delete_permission | func | 当前user是否拥有删除权限


## 3. 几个重要view的属性方法及其加载流程
### 1. ListAdminView
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
list_display | attr | 要展示的字段， 元组或列表
list_display_links | attr | 展示change页面链接的字段， 默认第一个字段
list_display_links_detail | attr | 要展示详情页的字段
list_select_related | attr | mysql查询时select related的字段， 外键的话可以用来减少mysql查询次数
list_per_page | attr | 每页展示多少条数据， 默认50
list_max_show_all | attr | 点击展示全部时最多展示多少条数据， 默认200
list_exclude | attr| 如果不指定list_display, 可以在这指定不展示哪些字段
search_field | aatr | 用来搜索的字段
paginator_class | attr | 分页的class， 默认是django自带的Paginator
ordering | attr| 指定用来排序的字段或方法
object_list_template | attr | 列表页的模板， 继承自base_site
init_request | func | 初始化一些属性
get_list_display | func | 获取用来展示的字段， 来源是url中的col参数或者是list_display
get_list_display_links | func | 获取用来展示link的字段， 如果不指定的话就去list_display中的第一个， 可以通过指定list_display = ['None'] 来取消展示链接
make_result_list | func | 根据页数、是否展示所有等参数来获取要展示的数据， 这里会hit db
get_result_list | func | 调用make_result_list 来获取数据， 可被plugin覆盖
post_result_list | func | 调用make_result_list 来获取数据， 可被plugin覆盖
get_list_queryset | func | 获取select related 和 ordering字段来生成queryset
_get_default_ordering | func | 获取默认排序， 取值是自定义的self.ordering 或 model meta中定义的ordering
get_ordering_field | func | 把排序字段映射到model的真实field
get_ordering | func | 从url中和 _get_default_ordering 获取排序字段， 并在最后附加上主键字段（默认排序）
get_ordering_field_columns | func | 将排序字段拼接成有序字典， key是排序字段， value是生序（ASC）或降序（DESC）
get_check_field_url | func | null
get_model_method_fields | func | 将model中自定义的字段， 用FakeMethodField 封装成model字段。 字段需要加上is_column 属性
get_context | func | 更新了results、headers、model_fields 等字段
get_response | func | 提供给plugin使用
get | func | 返回get_response（默认为None） 或  model_list.html 和 context 的渲染结果
post_response | func | 提供给plugin使用
post | func | 提供给plugin使用
get_paginator | func | 获取用来分页的class
get_page_number | func | 获取页数的html展示格式
result_header | func | 列表页头部字段
result_headers | func | 调用result_header 获取所有字段的头部
result_item | func | 展示字段value结果
result_row | func | 调用result_item， 返回一行的数据展示结果
results | func | 调用result_row , 返回所有数据的展示结果
url_for_result | func | 返回某个object的详情页或编辑页url
get_media | func | 添加了details.js 和 form.css
block_pagination | func | 展示html页码， 返回一些数据， 然后通过@inclusion_tag 来渲染到pagination.html

List 页面的加载流程
![描述](/img/old-post/704690a2d6c91745dc8eb7ef09570f681802.PNG)

### 2. DetailAdminView
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
form | attr | 用来展示model详情的form， 默认是django的forms.ModelForm
detail_layout | attr | layout， 默认None
detail_show_all | attr | 展示所有数据， 默认true
detail_template | attr | 自定义details模板， 默认None， 即使用model_detail.html
form_layout | attr | form layout, 优先级在detail layout 之后
init_request | func | 权限判断和404处理
get_form_layout | func | 获取form的layout
get_model_form | func | 获取model的form， 用modelform_factory根据指定的form和model的fields生成
get_form_helper | func | 获取form的helper并调用get_form_layout获取layout
get | func|根据self.obj 初始化form， 赋值给self.form_obj, 返回self.get_response()
get_context | func | 新增了form、object 和是否拥有编辑、删除权限
get_breadcrumb | func | title 和 url 赋值为obj的verbose_name 和 详情页url
get_media | func| 新增了 form.js 和 form.css
get_field_result | func | 获取字段的html 展示
get_response | func | 根据context和model_detail.html 返回渲染好的详情页

Detail 页面的加载流程
![描述](/img/old-post/c2be2a121efa612a010a12fa847a08421477.PNG)
### 3 ModelFormAdminView
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
form | attr | 用来新建或修改model的form, 默认是forms.ModelForm
formfield_overrides | attr| 字典 用来覆盖某些model的field
readonly_fields | attr | 元组或列表， 只读的字段
style_fields | attr | 字典， 用来设定某些字段的展示样式
exclude | attr | 元组或列表， 不显示的字段
relfield_style | attr | 外键field的style
save_as | attr | 用来展示show_save_as_new等字段， 默认不展示
save_on_top | attr | 提交按钮放在顶部， 默认false
add_form_template | attr | 添加model的template， 默认model_form.html
change_form_template | attr | 编辑model的template， 默认model_form.html
form_layout | attr | from layout, 默认None
formfield_for_dbfield | func | db field 转化成对应的form field
get_field_style | func | 根据db field 和指定的style， 获取对应的具体展示样式
get_field_attrs | func | 获取db field的style， 被formfield_for_dbfield调用
prepare_form | func | 调用get_model_form 并赋值给self.model_form
instance_forms | func | 调用get_form_datas 并初始化self.model_from, 赋值给self.form_obj
setup_forms | func | 设置form的helper
valid_forms | func | 校验表单
get_model_form| func | 根据self.form、self.fields、self.exclude等生成model form
get_form_layout | func | 生成form的layout
get_form_helper | func | 生成form的helper并绑定layout
get_readonly_fields | func | 获取只读字段
save_forms | func | 提交表单， 生成的obj赋值给self.new_obj（不commit）
change_message | func | 在页面上展示添加或编辑model成功的信息
save_models | func | 保存 self.new_obj
save_related | func | 保存model的外键obj
get | func | 实例化表单， 设置helper， 返回get_response()
post | func | 保存提交的数据, 返回get_response()
get_context | func | 新添加了form， 原obj， 以及编辑、删除的权限和按钮
get_error_list | func | 返回错误， 用来展示到表单上
get_media | func | 新添加了form.js 和 form.css

### 4. CreateAdminView
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
init_request | func | 检查权限， 调用prepare_form()
get_form_datas | func | 如果是get， 则从url中获取一些值填充在表单， 如果是post， 则返回post的data和file
get_context | func | 更新了title字段
get_breadcrumb | func | 更新面板栏的title和url
get_response | func | 根据context 和 add_form_template 返回渲染的结果
post_response | func | 展示消息和跳转到列表页或index页

新增页面的加载流程：
![描述](/img/old-post/421e5bac1d93d4c7565949deb20416c03548.PNG)

### 5. UpdateAdminView
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
init_request | func | 从url中获取object_id并用它获取org_obj, 调用prepare_form()
get_form_datas | func | 根据org_obj 返回form 的data
get_context | func| 更新title并添加obj ID
get_breadcrumb | func | 更新面板栏的title和url
get_response | func | 根据context 和 change_form_template 返回渲染结果
post | func | 如果有_saveasnew 字段则调用CreateAdminView 的post， 否则保存提交的数据, 返回get_response()
post_response | func | 展示消息和跳转到列表页或index页

更新页面的加载流程
![描述](/img/old-post/bcba42c59ca2348f4169bb5364d569167250.PNG)

## 4. xadmin site
### 1. AdminSite 中几个重要的属性和方法
字段 | 属性 or 方法 | 描述
:----------- |:------------- |:------------- 
self._registry = {} | attr |   这里记录了model注册的admin_class
self._registry_avs = {}| attr | 记录view注册的admin_class, 增强已有view的功能
self._registry_views[] | attr |  直接注册的view， 可以根据url（path）访问到对应的view
self._registry_modelviews = [] | attr |  记录model view, 这里的view会注册到每个model上
register_view | func | 将view记录到_registry_views 中
register | func | 注册model或view的admin_class, 前者记录到_registry， 后者记录到_registry_avs
unregister | func | register 的逆操作
set_loginview | func | 设置自定义的login view
admin_view | func | 在view外边包装一层， 增加了权限检查
get_view_class | func | 比较重要， 下面会讲
create_admin_view | func | 调用get_view_class 生成admin_view
create_model_admin_view | func | 调用get_view_class, 生成admin_view
get_urls | func |比较重要， 下面会讲
urls | property | 调用get_urls, 获取xadmin的urls

### 2. get_view_class()
get_view_class()
```python
    def get_view_class(self, view_class, option_class=None, **opts):
        merges = [option_class] if option_class else []
        for klass in view_class.mro():
            reg_class = self._registry_avs.get(klass)
            if reg_class:
                merges.append(reg_class)
            settings_class = self._get_settings_class(klass)
            if settings_class:
                merges.append(settings_class)
            merges.append(klass)
        new_class_name = ''.join([c.__name__ for c in merges])

        if new_class_name not in self._admin_view_cache:
            plugins = self.get_plugins(view_class, option_class)
            self._admin_view_cache[new_class_name] = MergeAdminMetaclass(
                new_class_name, tuple(merges),
                dict({'plugin_classes': plugins, 'admin_site': self}, **opts))

        return self._admin_view_cache[new_class_name]
```
当create_admin_view 调用get_view_class时， 只传入了view_class， 而get_view_class 所做的就是查找view_class所有的父类， 以及通过register()注册到这些父类上的admin_class,  所有这些类放到一个列表里， 然后用type生成一个新的类， 名称是这些类的名字的字符串拼接， 拥有这些类所有的属性和方法。属性和方法的覆盖顺序与这些类在列表中的位置一致， 即最底层的会覆盖掉它的父类


如果是create_model_admin_view 调用get_view_class, 还会传入option_class， 这个option_class其实就是我们自己定义的admin_class, 会被放到列表的第一个， 覆盖调其它类相同的属性和方法。

比如访问项目中的一个List页面， 对应的新生成的view的名字是这样的：
```
xadmin.sites.activitybusinessaccountAdminListAdminViewModelAdminViewGlobeSettingCommAdminViewBaseSettingBaseAdminViewBaseAdminObjectViewobject
```
很清楚的看到后面是ListAdminView ——> ModelAdminView ——> CommAdminView——>BaseAdminView ——> object
另外， 值得注意的是， 你自己定义的admin class的名字是无关紧要的， 因为会在register过程中替换掉：
```python
admin_class = type(str("%s%sAdmin" % (model._meta.app_label, model._meta.model_name)), (admin_class,), options or {})
```
可以看到名字被换成了"%s%sAdmin"， 第一个%s是app的名字， 第二个是model的名字

### 3. get_urls()
get_urls 里主要做了两件事：
* 一是把self._registry_views中的每个view， 通过调用create_admin_view 生成新的view并注册到对应的url上。  
* 二是把self._registry 中的每个对应的model: admin_class 与self._registry_modelviews 中的每个model_view相结合， 调用create_model_admin_view并将新生成的view注册到相应的url上。其实就是为每个model的admin_class注册ListAdminView、DetailAdminView、UpdateAdminView、CreateAdminView等。
```python
# 将self._registry_views中的view注册到对应的url上
urlpatterns += [
    url(
        path,
        wrap(self.create_admin_view(clz_or_func))
        if inspect.isclass(clz_or_func) and issubclass(clz_or_func, BaseAdminView)
        else include(clz_or_func(self)),
        name=name
    )
    for path, clz_or_func, name in self._registry_views
]

# 把self._registry 中的每个对应的model: admin_class 与self._registry_modelviews 中的每个model_view相结合， 调用create_model_admin_view并将新生成的view注册到相应的url上
for model, admin_class in iteritems(self._registry):
    view_urls = [
        url(
            path,
            wrap(self.create_model_admin_view(clz, model, admin_class)),
            name=name % (model._meta.app_label, model._meta.model_name)
        )
        for path, clz, name in self._registry_modelviews
    ]
    urlpatterns += [
        url(r'^%s/%s/' % (model._meta.app_label, model._meta.model_name), include(view_urls))
    ]
```

## 5. 项目启动时xadmin的加载流程
![描述](/img/old-post/0b22cf6da61b0b6f733ea29dbc0e0a1b5002.PNG)

加载好url后， 打开admin页面， 加载IndexView对应的页面， indexView继承自CommadminView, 里面的get_nav_menu会把菜单栏加载到网页上， 打开其中的页面， 会加载对应model的ListAdminView。 

## 6. 代码中几个不太完美的地方
*  自定义的admin_class 不是直接继承自某个父类， 所以在pycharm上写某些属性和方法时不能自动补全
*   adminView中有好几处未定义先使用的情况， 例如在CommadminView的get_nav_menu中调用了admin_site,  
 `for model, model_admin in self.admin_site._registry.items()`
而这个属性是在site的get_view_class（）中创建class时通过指定options赋值上去的。
```
    self._admin_view_cache[new_class_name] = MergeAdminMetaclass(
                new_class_name, tuple(merges),
                dict({'plugin_classes': plugins, 'admin_site': self}, **opts))
```  

*  CommadminView中的get_nav_menu 方法只取了site中的_register中的model:admin_class, 没有取_register_view, 所以通过register_view()注册的view不会显示在菜单栏里。
*   `flag = self.org_obj is None and 'create' or 'change'` 代码里有好几处这样的代码， 读起来很绕， 不如直接写成 
`flag = 'create' if self.org_obj is None else 'change'`
*   有好几处列表解析式的长度都超过了三行， 建议直接写成for循环算啦。

### 传送门
xadmin 源码地址： [https://github.com/sshwsfc/xadmin](https://github.com/sshwsfc/xadmin)
xadmin 官方文档（比较简略）： [https://xadmin.readthedocs.io/en/latest/index.html](https://xadmin.readthedocs.io/en/latest/index.html)

