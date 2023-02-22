## Procedure for Development

------

### How to add items to the side navigation menu?
- Purpose
In order to connect to development page from the menu, it requires adding corresponding items to the menu on the left side of the screen before developing new pages

- Brief introduction of configuration of the navigation
Items on the menu can be divided into Section and Module. Module is an item linked to each page and it will use Section to classify. In the figure below, the items added by following steps will show in the block above the blue line and which in the block below the blue line are hard-coded and fixed.
![nav_intro](./images/nav_intro.png)

- Steps
	1. Click ‘Navigation’ under SYSTEM in the menu, it will appear Section and Module. Choose one of them depending on needs. Here takes Module as an example.
	
	1. After clicking ‘Module’, click ‘+ Add new’ on the page, then will enter adding page.
        ![addModule](./images/addModule.png)
	1. Fill in relevant information to add successfully. Please note that the Link field must to be identical to the settings in urls.py.
        ![module_info](./images/module_info.png)
	
		Take Single Product for example. If we want to build the page on the menu with the path ‘Product Operation > Product > Single Product’, we should enter ‘/product/single’ in the Link field, and add ‘path('single', .......)’ to the urls.py of product.
		
### Current approach
Since the present Navigation module is hard-coded, and the permission of the Navigation module needs to be identical to django.auth.models.Permission id to work properly, we currently use the DB table built by Airy to set.

Please use the data in 4 tables in db.zip: systemsetting_navigationmodule, systemsetting_navigationmodule_permission, systemsetting_navigationsection, auth_permission to replace data in your own DB. Also, set required permissions to see navigation you want to the user who wants to use the navigation (or log in directly by using superuser, then you can all navigations directly).

However, the current design shows that there is a flaw in the configuration of navigation. That is, template render of navigation is stored into session at main.views.MainView.home_page(), and the navigation can only be displayed through this view function. Therefore, to display correctly, in addition to preparing above data well, it’s also necessary to access the URL ‘/systemadmin’ to show navigation after login.

In summary, there are three steps to display navigation properly:

1. Use the data in 4 tables in db.zip: systemsetting_navigationmodule, systemsetting_navigationmodule_permission, systemsetting_navigationsection, auth_permission.
1. Set required permissions to the user who wants to show the navigation (or log in by using superuser).
1. Access the URL ‘/systemadmin’ after login, and the navigation will be displayed.

## Usage of API

------

### Usage of Category Service
The following five APIs are provided currently:

1. ```getCategories(type=None)-> list```：
	If no type is specified, it will return all catogories by default; otherwise, it will return all categories of the type.
	
1. ```getCategoryById(id)-> Category```：Use id to get the category item.

1. ```getAttributesByCategory(category)-> dict```：
	Use specified catogory to get catogory attributes of the category. It will return in the format of dict. The key of the dict is the name of the CategoryAttribute and the value is the CategoryAttribute itself.

1. ```getAttributeValues(instance, attributeValueModel)-> dict```：
	Use specified entity, such as EvSKU, Product, etc., objects with their own catogory, and AttributeValue class of the object to get all AttributesValues of the object.

	It will return in the format of dict. The key of the dict is the name of the CategoryAttribute and the value is the value of the AttributeValue itself. 

	For example, if I have a Product objects called p, it has two 		CategoryAttributes and the (name, value) are ('attr1', 1)、('attr2', 'attr 2 val') respectively. The return value of using ```getAttributeValues(p, ProductAttributeValue)``` will be a dict whose value is ```{'attr1': 1, 'attr2': 'attr 2 val'} ```.

1. ```saveAttributeValues(instance, attributeValueModel, instanceValueDict)-> object```: 
	A value dict which uses specified entity, AttributeValue class and value of the entity. It returns the entity. 

	For example, there is a Product objects called p, it has two CategoryAttributes which are 'attr1' and 'attr2' respectively. Now receive a request to set the following values to p: ```{'attr1': 1, 'attr2': 'attr 2 val'}```, then build a dictionary ```d = {'attr1': 1, 'attr2': 'attr 2 val', 'editor': request.user} ``` with dict as its value. **Please pay attention to set editor!**
	
	Next, use ```saveAttributeValues(p, ProductAttributeValue, d)``` to complete the update. The return value is p.

If you have any doubts about usage, you can refer to TestCategoryService.testCategoryService() in category/tests.py. It is a test of CategoryService operated with the APIs which can be regarded as an example of using these APIs.

If you still have some questions, or feel that there are some problems when using above APIs, please contact with Jacky.

### Usage of Category Attribute Value Form
Since many steps will be repeated when adding/modifying an instance with category, including the table should display different category attributes when modifying category, storage of category attributes values, etc.

To make all of these easier to use, category provides relatetd functions currently, so that these duplicate steps can be completed directly.

#### Related component
1. category.views.CategoryViews: CategoryAttributeValuesFormMixin, CategoryAttributeValuesCreateView, CategoryAttributeValuesUpdateView
1. category.views.forms.CategoryForms: BaseCategoryAttributeValueForm
1. category.templatetags.categoryTags: renderCategoryAttributeValuesForm

#### Usage
1. Let the views that you use to add/modify instances inherit CategoryAttributeValuesCreateView and CategoryAttributeValuesUpdateView respectively.
1. Continuing, override get_form_kwargs() in these two views, please refer to the explanation for the reason, and the content is as follows:
   ```console
   def get_form_kwargs(self):
       kwargs = super().get_form_kwargs()
       kwargs['attributeValueModel'] = <Your Attribute Value Model>
       <... All kwargs you want to add in ...>
       return kwargs
   ```
1. Continuing from the above two points, set attribute in the view: ```categoryType = <Your category type> ```. This category type can be obtained from constant in CategoryService, and which categoryType to use depends on what objects the view is used to add/modify. For example, product.views.ProductView.AddNew used to add Product will set ``` categoryType = CategoryService.PRODUCT_CATEGORY ```.
1. Add an instance form inherits BaseCategoryAttributeValueForm, and set form_class of adding/modifying views to this form.
1. In template, those who call the API do not need to write category and fields of categoryAttributes by themselves, just add ```{% load categoryTags %}``` to the file header directly and add ```{% renderCategoryAttributeValuesForm form categoryType %} ``` to where needs to have category and fields of categoryAttributes. **(Notice: The form parameter to access is the form whose view uses this template tag, and categoryType can be directly typed in categoryType. The CategoryAttributeValuesMixin has already processed this context_data.)**
1. Lastly, when you want to save the instance, you only need to call ```form.save()``` directly, which will save the instance itself and attributeValues.

#### Explanation
1. Both of CategoryAttributeValuesCreateView and CategoryAttributeValuesUpdateView inherit CategoryAttributeValuesFormMixin, and ```get_form_kwargs()``` in CategoryAttributeValuesFormMixin defines some kwargs that need to be passed into form and will affect operation of form.
1. Continuing, ```CategoryService.saveAttributeValues()``` is used in the steps of ```form.save()```, and instanceAttributeValueModel is necessary to operate properly. But it annot be defined in ```CategoryAttributeValuesFormMixin.get_form_kwargs()``` in advance, users have to override ```get_form_kwargs()``` in CategoryAttributeValuesCreate/UpdateView and add ```{'attributeValueModel': <attributeValueModel>} ``` to operate properly on their owns. If they don't  do that, an exception will be raised in ```BaseCategoryAttributeValueForm.__init__()``` to ask for overriding.
1. If you still have some questions about using, you can review the code in 'Related component' or ask Jacky.
1. If you feel that there are some problems when using, you can refer to product.views.ProductView.AddNew.

### Frame of template
- Since the functions of each module are mainly CRUD, only the architecture of the models is somewhat different, in order to shorten development time, you can use the shared moduleBase.html template (for example, you can refer to the SKU module).
    - If you want to create a new List page, you can directly inherit the utils/moduleBase.html template and just design table in the moduleContent block inside.
      ![list](./images/list.jpg)
    - If you want to create a new form, (because the for fields between modules are slightly different) you need to make a formBase template that inherits moduleBase first, design form of modules in the moduleContent block, then create another template that inherits the formBase.
      ![form](./images/form.jpg)
      ![add](./images/add.jpg)

### EvcoViews
There are also shered EvcoCreateView and shared EvcoUpdateView in utils/views/UtilsViews.py (for example, you can refer to the SKU module).
There is a formValid function in these two views. The way to use it is to call formValid() in form_valid of general views, enter form, function to be executed and redirect url as parameters, then formValid will use the form to execute the function and return the url.
**Notion**: In formValid(), the return value of function parameter must be a subclass object of the BaseModel. Sometimes it may be misused as a return value of QuerySet or others, so that this view cannot be used properly, and make fool-proofing program cause errors.

#### Instruction of parameter form, fKwargsDict
- In the case of only providing form parameter, formValid will directly access the whole form.cleaned_data to execute the function so form.fields in the template need to be identical to the kwargs in the function.
- If there are some inconsistencies, such as 'form.fields and kwargs of function's name are different', 'there are some fields have to be dealed with  separately', etc., please save the fields that the function will be used to deal with as dict() and use fKwargsDict parameter.
  ![skuCreateView](./images/skuCreateView.jpg)
  ![evcoCreateView](./images/evcoCreateView.jpg)

#### Notice
- Both CategoryAttributeValuesCreate/UpdateView have already inherited these two views. If there is inheritance, there is no need to deal with these two views.

## How to save files into database

### Settings
Set in settings.py: ```DEFAULT_FILE_STORAGE = 'evcoos.storage.EvcoFileStorage'```
EvcoFileStorage in evcoos.storage.py is a DB Storage we developed by ourselves to deal with bugs ralated to suites.
Then, add to urls.py: ```url(r'^files/', include('db_file_storage.urls')),```

### Model Fields
Every FileField requires setting a additional model which is used to access infomation of FileField. The following takes VehicleFile for example:
1. Create a VehicleFileModel to access information of FileField firstly:
   ```console
   class VehicleFileModel(models.Model):
       bytes = models.TextField()
       filename = models.CharField(max_length=255)
       mimetype = models.CharField(max_length=50)
   ```
1. Next, set upload_to in FileField. The format is [name of accessed model]\bytes\filename\mimetype:
   ```console
   class VehicleFile(BaseModel):
       description = models.TextField(default="")
       vehicleFile = models.FileField(upload_to='asset.VehicleFileModel\\bytes\\filename\\mimetype')
       asset = models.ForeignKey('Asset', on_delete=models.SET_NULL, null=True)
   ```
    
### How to read files in database

#### Download
```console
<a href='{% url "db_file_storage.download_file" %}?name={{ object.vehicleFile }}'>
     <i>Click here to download the picture</i>
</a>
```

#### Browse(image)
```console
<img src="{% url 'db_file_storage.get_file' %}?name={{ object.vehicleFile }}" />
```