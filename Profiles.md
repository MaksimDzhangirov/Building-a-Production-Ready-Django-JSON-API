# Профили пользователей

В конце прошлой главы мы кратко коснулись различия между пользователя и их профилями, но я хотел бы немного подробнее разобрать эту тему, прежде чем мы начнём работать с профилями.

При разработке программного обеспечения существует такое понятие как [Принцип Единой Ответственности] (https://en.wikipedia.org/wiki/Single_responsibility_principle). Идея заключается в том, что каждый класс должен делать что-то одно, но делать это очень хорошо. Как Принцип Единой Ответственности связан с нашей темой? Он является причиной, из-за которой мы разделяем пользователей и их профили.

Мы используем модель пользователя для аутентификации и авторизации (прав доступа). Задача модели `User` заключается в том, чтобы убедиться, что пользователю разрешен доступ к тем ресурсам, которые они хотят получить. Например, пользователю может быть разрешено редактировать свою электронную почту и пароль. Но в то же время у них не должно быть возможности изменить электронную почту и пароль другого пользователя.

Наоборот, модель `Profile` предназначена для отображения информации о пользователе в пользовательском интерфейсе. В нашем клиенте будет страница с профилем для каждого пользователя откуда и происходит название модели `Profile`. Сейчас мы перенесём кое-что из модели пользователя, поскольку существует непосредственная связь между моделями "Профиль" и "Пользователь" и наша цель свести дублирующуюся информацию в них к минимуму.

Теперь мы готовы к созданию модели `Profile`.

## Создание модели Profile

Создайте `conduit/apps/profiles/models.py` и добавьте следующий код:

```python
from django.db import models

class Profile(models.Model):
    # Существует непосредственная связь между моделью Profile и
    # User. Создавая связь один-к-одному между ними, мы формализуем эту связь.
    # Каждый пользователь будет иметь одну и только одну связанную с ним модель Profile.
    user = models.OneToOneField(
        'authentication.User', on_delete=models.CASCADE
    )

    # У каждого профиля пользователя будет поле, с помощью  other users
    # которого они могут рассказать что-то о себе. Это поле будет пустым в
    # момент создания пользователем своей учетной записи, поэтому 
    # мы укажем это следующим образом blank=True.
    bio = models.TextField(blank=True)

    # Кроме поля `bio`, каждый пользовател может загрузить картинку для профиля или
    # аватар. Это поле не будет обязательным и может быть пустым.
    image = models.URLField(blank=True)

    # Временная метка, указывающая, когда был создан этот объект.
    created_at = models.DateTimeField(auto_now_add=True)

    # Временная метка, указывающая, когда был в последний раз обновлен этот объект.
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.user.username
```

Вы наверное заметили, что и модель User, и Profile содержат поля. Эти поля будут использоваться во всех нащих моделях, поэтому почему бы не потратить несколько минут и перенести их в свою собственную модель?

## Модель для временных меток

Создайте `conduit/apps/core/models.py` и добавьте следующий фрагмент кода:

```python
from django.db import models

class TimestampedModel(models.Model):
    # Временная метка, показывающая, когда был создан этот объект.
    created_at = models.DateTimeField(auto_now_add=True)

    # Временная метка, показывающая, когда был в последний раз обновлен этот объект.
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

        # По умолчанию, любая модель, которая наследуется от `TimestampedModel`
        # должна быть упорядочена в обратном хронологическом порядке. Мы можем переопределить это 
        # для каждой модели в случае необходимости, но обратный хронологический
        #  порядок хороший выбор по умолчанию для большинства моделей.
        ordering = ['-created_at', '-updated_at’]
```
А теперь измените `conduit/apps/profiles/models.py` следующим образом:

```python
from django.db import models

+from conduit.apps.core.models import TimestampedModel


-class Profile(models.Model):
+class Profile(TimestampedModel):
    # Как было сказано выше, существует непосредственная связь между моделью Profile и
    # User. Создавая связь один-к-одному между ними, мы формализуем эту связь.
    # Каждый пользователь будет иметь одну и только одну связанную с ним модель Profile.
    user = models.OneToOneField(
        'authentication.User', on_delete=models.CASCADE
    )

    # У каждого профиля пользователя будет поле, с помощью  other users
    # которого они могут рассказать что-то о себе. Это поле будет пустым в
    # момент создания пользователем своей учетной записи, поэтому 
    # мы укажем это следующим образом `blank=True`.
    bio = models.TextField(blank=True)

    # Кроме поля `bio`, каждый пользовател может загрузить картинку для профиля или
    # аватар. Это поле не будет обязательным и может быть пустым.
    image = models.URLField(blank=True)

-    # Временная метка, указывающая, когда был создан этот объект.
-    created_at = models.DateTimeField(auto_now_add=True)
-
-    # Временная метка, указывающая, когда был в последний раз обновлен этот объект.
-    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.user.username
```
Поскольку мы хотим также использовать модель для временных меток в модели `User`, нам нужно сделать пару изменений в ней.

Откройте `conduit/apps/authentication/models.py` и внесите следующие изменения:

```python
import jwt

from datetime import datetime, timedelta

from django.conf import settings
from django.contrib.auth.models import (
    AbstractBaseUser, BaseUserManager, PermissionsMixin
)
from django.db import models

+from conduit.apps.core.models import TimestampedModel

# …

-class User(AbstractBaseUser, PermissionsMixin):
+class User(AbstractBaseUser, PermissionsMixin, TimestampedModel):
    # У каждого `User` должен быть уникальный человеко-понятный идентификатор,
    # который мы можем использовать для представления `User` в UI. Мы хотим
    # проиндексировать этот столбец в базе данных для ускорения поиска. 
    username = models.CharField(db_index=True, max_length=255, unique=True)

    # Нам также нужно каким-то образом связываться с пользователем
    # и способ идентификации пользователя при входе в систему. Поскольку нам в
    # любом случае необходим адрес электронной почты для связи с пользователем, 
    # мы будем также использовать email для входа в систему, поскольку он 
    # наиболее часто используется в качестве логина на момент написания учебного 
    # пособия.
    email = models.EmailField(db_index=True, unique=True)

    # Когда пользователь больше не захочет использовать нашу платформу,     
    # он может захотеть удалить свою учетную запись. Для нас это будет проблемой, 
    # поскольку собранные о пользователе данные ценны для нас и мы не хотим удалять их. 
    # Мы просто предложим пользователям отключить их учетную запись вместо её удаления.
    # Таким образом, они больше не будут отображаться на сайте, но мы сможем продолжать
    # анализировать собранные данные.   
    is_active = models.BooleanField(default=True)

    # Флаг `is_staff` используется Django, чтобы определить кто может, 
    # а кто - нет входить в систему администрирования Django. Для большинства пользователей
    # значение этого флага всегда будет равно false.
    is_staff = models.BooleanField(default=False)

-    # Временная метка, показывающая когда был создан этот объект.
-    created_at = models.DateTimeField(auto_now_add=True)
-
-    # Временная метка, показывающая, когда в последний раз обновлялся этот объект.
-    updated_at = models.DateTimeField(auto_now=True)

    # При использовании нестандартной, пользовательской модели пользователя необходимо
    # определить дополнительные поля, требуемые Django.

    # Свойство `USERNAME_FIELD` указывает какое поле будет использоваться для входа в систему.
    # Здесь мы хотим использовать поле email.
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['username']

        # …
```

## Отношение один-к-одному и использование сигнального фреймворка Django

В модели `Profile` мы создали отношение один-к-одному между `User` и `Profile`. Было бы неплохо, если бы этого было бы достаточно и мы могли бы на этом закончить, но нам все равно необходимо сообщить Django, что мы хотим создавать `Profile` каждый раз, когда мы создаём `User`.

Для этого мы будем использовать [сигнальный](https://docs.djangoproject.com/en/1.9/topics/signals/) фреймворк Django. А именно, мы будем использовать сигнал [post_save](https://docs.djangoproject.com/en/1.9/ref/signals/#django.db.models.signals.post_save), чтобы создать экземпляр модели `Profile` после экземпляра модели `User`.

Начнём открыв `conduit/apps/authentication/signals.py` и добавив в файл следующий код:

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

from conduit.apps.profiles.models import Profile

from .models import User

@receiver(post_save, sender=User)
def create_related_profile(sender, instance, created, *args, **kwargs):
    # Обратите внимание, что здесь мы проверяем поле `created`. Мы делаем это только 
    # при создании экземляра `User`. Если метод save, вызвавший этот сигнал,
    # был запущен при обновлении экземпляра модели, то очевидно, что у пользователя
    # уже есть профиль.
    if instance and created:
        instance.profile = Profile.objects.create(user=instance)
```
Это сигнал, который создаст объект профиля, но Django не будет запускать его по умолчанию. Вместо этого нам необходимо создать свой пользовательский класс `AppConfig` для приложения `authentication` и зарегистрировать его в Django.

Откройте `conduit/apps/authentication/__init__.py` и добавьте следующий код:

```python
from django.apps import AppConfig


class AuthenticationAppConfig(AppConfig):
    name = 'conduit.apps.authentication'
    label = 'authentication'
    verbose_name = 'Authentication'

    def ready(self):
        import conduit.apps.authentication.signals

# Вот как мы регистрируем свою, пользовательскую конфигурацию приложения в Django. Django достаточно умён, 
# чтобы найти свойство `default_app_config` для каждого зарегистрированного приложения и 
# и использовать правильную конфигурацию приложения на основе этого значения.
default_app_config = 'conduit.apps.authentication.AuthenticationAppConfig'
```

Теперь после создания нового пользователя для этого пользователя должен создаваться профиль. Давайте протестируем это, чтобы убедиться, что всё работает правильно.

## Тестируем created_related_profile

Первое, что нам нужно сделать - это удалить нашу существующую базу данных. Ни у одного из наших пользователей нет профиля, поэтому Django попросит ввести значение по умолчанию. Проблема заключается в том, что вводить значение по умолчанию нужно будет для каждой новой записи, которую мы создадим в будущем, чего мы конечно не хотим.

Чтобы удалить базу данных, удалите файл `db.sqlite3` в корневом каталоге Вашего проекта.

После этого мы хотим создать новые миграции для приложения `profiles`. Поскольку мы делаем это в первый раз для `profiles`, нам надо явно указать, что эти миграции относятся к приложению `profiles`.

```
~ python manage.py makemigrations profiles
```

> ЗАМЕЧАНИЕ: В приведенном выше фрагменте не нужно вводить символ ~ вместе с остальной командой. Мы использовали его, чтобы указать, что эту команду нужно запускать из командной строки.

После создания новых миграций запустите следующую команду, чтобы применить новые миграции и создать новую базу данных:

```
~ python manage.py migrate
```

Теперь Вы можете использовать Postman для отправки запроса на регистрацию и создания нового пользователя с профилем. Давайте отправим этот запрос прямо сейчас.

Теперь нам нужно проверить, что профиль был успешно создан. Выполните следующую команду из командной строки, чтобы открыть новую оболочку:

```
~ python manage.py shell_plus
```

Внутри оболочки все что нам нужно сделать - это извлечь пользователя, которого мы только что создали, и убедиться, что у него есть профиль:

```
>>> u = User.objects.first()
>>> u.profile
<Profile: james>
```

Результат выполнения команды `u.profile` будет отличаться именем созданного Вами пользователя. Если `u.profile` возвращает экземпляр модели `Profile`, то можно переходить к следующему разделу.
 
## Сериализуем объекты Profile

Создайте `conduit/apps/profiles/serializers.py` со следующим содержимым:

```python
from rest_framework import serializers

from .models import Profile

class ProfileSerializer(serializers.ModelSerializer):
    username = serializers.CharField(source='user.username')
    bio = serializers.CharField(allow_blank=True, required=False)
    image = serializers.SerializerMethodField()

    class Meta:
        model = Profile
        fields = ('username', 'bio', 'image',)
        read_only_fields = ('username',)

    def get_image(self, obj):
        if obj.image:
            return obj.image

        return 'https://static.productionready.io/images/smiley-cyrus.jpg'
```

В этом фрагменте кода нет ничего, чего бы мы не рассматривали ранее. Сериализатор очень похож на `UserSerializer`, который мы создали в прошлой главе. Можно переходить к следующему разделу.

## Отображаем объекты Profile

Поскольку мы знаем, что столкнемся с той же проблемой, которая у нас уже была, когда данные о пользователе выдавались не в пространстве имен "user", давайте не будем медлить и создадим `ProfileJSONRenderer`. Этот формирователь ответа от сервера будет очень похож на `UserJSONRenderer`, поэтому мы создадим `ConduitJSONRenderer`, от которого могут наследоваться как `UserJSONRenderer`, так и `ProfileJSONRenderer`. Это позволит нам абстрагировать некоторые части кода и избежать их дублирования.

Добавьте следующее в `conduit/apps/core/renderers.py`:

```python
import json

from rest_framework.renderers import JSONRenderer


class ConduitJSONRenderer(JSONRenderer):
    charset = 'utf-8'
    object_label = 'object'

    def render(self, data, media_type=None, renderer_context=None):
        # Если представление генерирует ошибку (например, пользователь не может быть аутентифицирован)
        # `data` будет содержать ключ `errors`. Мы хотим, чтобы 
        # используемый по умолчанию JSONRenderer обрабатывал ошибки, поэтому нам нужно
        # проверить этот случай.
        errors = data.get('errors', None)

        if errors is not None:
            # Как было сказано выше, мы позволяем используемому по умолчанию JSONRenderer
            # обрабатывать ошибки.
            return super(ConduitJSONRenderer, self).render(data)

        return json.dumps({
            self.object_label: data
        })
```

Существует два отличия между `ConduitJSONRenderer` и `UserJSONRenderer`, которые мы создали:

1. В `UserJSONRenderer` мы не указывали свойство `object_label`. Причина заключалась в том, что мы знали, что объектом для `UserJSONRenderer` будет `user`. В этом случае объект (или пространство имен) будет меняться в зависимости от того какой класс будет наследоваться от `ConduitJSONRenderer`. Поэтому мы позволяем задавать свойство `object_label` динамически и по молчанию принимаем его равным `object`.

2. `UserJSONRenderer` нужно осуществлять декодирование JWT, если он является частью запроса. Это декодирование нужно осуществлять только в `UserJSONRenderer` и оно не будет использоваться в каком-либо другом формирователе ответа от сервера. Не имеет смысла добавлять его в `ConduitJSONRenderer`. Скоро мы обновим `UserJSONRenderer`, чтобы учесть эту особенность.

Теперь добавьте следующее в `conduit/apps/profiles/renderers.py`:

```python
from conduit.apps.core.renderers import ConduitJSONRenderer


class ProfileJSONRenderer(ConduitJSONRenderer):
    object_label = 'profile'
```

В этом фрагменте кода нет ничего нового, поскольку `ProfileJSONRenderer` не отличается по своему функционалу от `UserJSONRenderer`.

Откройте `conduit/apps/authentication/renderers.py` и внесите следующие изменения:

```python
-import json
-
-from rest_framework.renderers import JSONRenderer
+from conduit.apps.core.renderers import ConduitJSONRenderer


-class UserJSONRenderer(JSONRenderer):
+class UserJSONRenderer(ConduitJSONRenderer):
-    charset = 'utf-8'
+    object_label = ‘user’

    def render(self, data, media_type=None, renderer_context=None):
-        # Если представление генерирует ошибку (например пользователь не может быть аутентифицирован
-        # или подобную), `data` будут содержать ключ `errors`. Мы хотим, чтобы используемый
-        # по умолчанию JSONRenderer обрабатывал ошибки, поэтому необходимо
-        # проверить наличие этого ключа в `data`.
-        errors = data.get('errors', None)
-
        # Если был передан ключ `token` в запросе, то он будет байтовым объектом.
        # Байтовые объекты плохо сериализуются, поэтому нам надо его декодировать
        # прежде чем выдавать объект User.
        token = data.get('token', None)

-        if errors is not None:
-            # As mentioned above, we will let the default JSONRenderer handle
-            # rendering errors.
-            return super(UserJSONRenderer, self).render(data)

        if token is not None and isinstance(token, bytes):
            # Как было сказано выше, мы декодируем `token` только в том случае,
            # если он является байтовым объектом.
            data['token'] = token.decode('utf-8')

-        # Наконец мы можем выдать наши данные в пространстве имен "user".
-        return json.dumps({
-            'user': data
-        })
+        return super(UserJSONRenderer, self).render(data)
```

В основном всё что мы сделали здесь - это удалили те части, которые теперь находятся в `ConduitJSONRenderer`.

Для `UserJSONRenderer` всё должно работать точно так же как работало ранее. Чтобы проверить это, выполните запрос "Current User" в Postman.

## ProfileRetrieveAPIView

Давайте добавим конечную точку для получения информации о конкретном пользователе.

```python
from rest_framework import status
from rest_framework.generics import RetrieveAPIView
from rest_framework.permissions import AllowAny
from rest_framework.response import Response

from .models import Profile
from .renderers import ProfileJSONRenderer
from .serializers import ProfileSerializer


class ProfileRetrieveAPIView(RetrieveAPIView):
    permission_classes = (AllowAny,)
    renderer_classes = (ProfileJSONRenderer,)
    serializer_class = ProfileSerializer

    def retrieve(self, request, username, *args, **kwargs):
        # Пытаемся извлечь запрошенный профиль и генерируем исключение, если 
        # профиль не найден.
        try:
            # Мы используем метод `select_related`, чтобы избежать ненужных запросов к 
            # базе данных.
            profile = Profile.objects.select_related('user').get(
                user__username=username
            )
        except Profile.DoesNotExist:
            raise

        serializer = self.serializer_class(profile)

        return Response(serializer.data, status=status.HTTP_200_OK)
```

В вышеприведенном коде мы учитываем случай, когда запрошенного профиля не существует, но делаем это не совсем верно. В частности, мы никак не контролируем сообщение об ошибке, которое получит клиент. Давайте попытаемся исправить этот недочёт. 

## ProfileDoesNotExist

Создайте файл с названием `conduit/apps/profiles/exceptions.py` и добавьте в него следующий код:

```python
from rest_framework.exceptions import APIException


class ProfileDoesNotExist(APIException):
    status_code = 400
    default_detail = 'The requested profile does not exist.'
```

Это простое исключение. В Django REST фреймворк каждый раз, когда Вы хотите создать своё собственное, пользовательское исключение Вы наследуете его от `APIException`. Всё что Вам нужно сделать затем - это указать свойства `default_detail` и `status_code`. Параметры по умолчанию для этого исключения можно переопределить для каждого конкретного случая, если Вы посчитаете, что так стоит сделать.

Внесите следующие изменения в функцию `core_exception_handler` из файла `conduit/apps/core/exceptions.py`:

```python
def core_exception_handler(exc, context):
    # Если возникает исключение, которое мы здесь явно не обрабатываем, мы хотим,
    # поручить его обработку стандартному DRF обработчику. Если мы хотим обработать данный тип исключения,
    # то нам всё равно нужен доступ к ответу, генерируемому DRF,
    # поэтому в первую очередь необходимо получить его.
    response = exception_handler(exc, context)
    handlers = {
+        'ProfileDoesNotExist': _handle_generic_error,
        'ValidationError': _handle_generic_error
    }
    # В строке кода после этого комментария видно как мы определяем
    # тип текущего исключения. Затем мы используем его, чтобы понять должны ли мы обрабатывать это
    # исключение или можно позволить Django REST фреймворку сделать это за нас.    
    exception_class = exc.__class__.__name__

    if exception_class in handlers:
        # Если это исключение одно из тех, что мы хотим обрабатывать, то обрабатываем его. В противном случае,
        # возвращаем ответ, сгенерированный ранее стандартным обработчиком исключений.
        return handlers[exception_class](exc, context, response)

    return response
```

Мы будем обрабатывать наше пользовательское исключение так же, как и `ValidationError`, но теперь мы можем корректировать информацию об  ошибке, которую увидит клиент. Не останавливаясь на полпути, добавим `ProfileDoesNotExist`  в наше представление.

Откройте `conduit/apps/profiles/views.py` и добавьте следующие изменение:

```python
from rest_framework import status
from rest_framework.generics import RetrieveAPIView
from rest_framework.permissions import AllowAny
from rest_framework.response import Response

+from .exceptions import ProfileDoesNotExist
from .models import Profile
from .renderers import ProfileJSONRenderer
from .serializers import ProfileSerializer


class ProfileRetrieveAPIView(RetrieveAPIView):
    permission_classes = (AllowAny,)
    renderer_classes = (ProfileJSONRenderer,)
    serializer_class = ProfileSerializer

    def retrieve(self, request, username, *args, **kwargs):
        # Пытаемся извлечь запрошенный профиль и генерируем ошибку, если
        # профиль не может быть найден.
        try:
            # Мы используем метод `select_related`, чтобы избежать ненужных
            # запросов к базе данных.
            profile = Profile.objects.select_related('user').get(
                user__username=username
            )
        except Profile.DoesNotExist:
-            raise
+            raise ProfileDoesNotExist

        serializer = self.serializer_class(profile)

        return Response(serializer.data, status=status.HTTP_200_OK)
```

Проблема решена! Давайте добавим url-адрес для `ProfileRetrieveAPIView` в наш файл urls.py.

Создайте `conduit/apps/profiles/urls.py` со следующим содержимым:

```python
from django.conf.urls import url

from .views import ProfileRetrieveAPIView

urlpatterns = [
    url(r'^profiles/(?P<username>\w+)/?$', ProfileRetrieveAPIView.as_view()),
]
```

Как и в случае с `conduit/apps/authentication/urls.py` нам надо зарегистрировать этот новый файл с url  в переменной `urlpatterns` из файла `conduit/urls.py`.

Откройте `conduit/urls.py` и внесите следующее изменение:

```python
urlpatterns = [
    url(r'^admin/', admin.site.urls),

    url(r'^api/', include('conduit.apps.authentication.urls', namespace='authentication')),
+    url(r'^api/', include('conduit.apps.profiles.urls', namespace='profiles')),
]
```

## Получаем профиль с помощью Postman

Если вы откроете Postman и загляните в каталог "Profiles", то увидите там запрос под названием "Profile."  Пошлите запрос на сервер, чтобы проверить, что всё что мы сделали до этого работает. Предполагая, что всё прошло хорошо, мы можем двигаться дальше и обновить представление `UserRetrieveUpdateAPIView`.

## Обновляем UserRetrieveUpdateAPIView

Откройте `conduit/apps/authentication/views.py` и добавьте следующие изменения в метод `update`:

```python
def update(self, request, *args, **kwargs):
-    serializer_data = request.data.get('user', {})
+    user_data = request.data.get('user', {})
+
+    serializer_data = {
+        ’username': user_data.get('username', request.user.username),
+        ’email': user_data.get('email', request.user.email),
+
+        ’profile': {
+            ’bio': user_data.get('bio', request.user.profile.bio),
+            ’image': user_data.get('image', request.user.profile.image)
+        }
+    }

    # Вот где используется последовательность сериализации, 
    # проверки, сохранения, о которой мы говорили ранее.
    serializer = self.serializer_class(
        request.user, data=serializer_data, partial=True
    )
    serializer.is_valid(raise_exception=True)
    serializer.save()

    return Response(serializer.data, status=status.HTTP_200_OK)
```

Эти изменения позволят нам использовать одну и ту же конечную точку для обновления электронной почты, пароля, биографии и аватара пользователя.

Нам также необходимо обновить `UserSerializer`, чтобы метод `update` работал с профилями.

## Обновляем UserSerializer

Откройте `conduit/apps/authentication/serializers.py` и обновите импорты следующим образом:

```python
from django.contrib.auth import authenticate

from rest_framework import serializers

+from conduit.apps.profiles.serializers import ProfileSerializer
+from .models import User
```

Затем мы можем обновить `UserSerializer` следующим образом:

```python
class UserSerializer(serializers.ModelSerializer):
    Handles serialization and deserialization of User objects."""

    # Длина пароля должна быть не менее 8 символов, но не более 128 
    # символов. Эти значения по умолчанию заданы в Django. Мы могли бы изменить их, но это бы
    # дополнительных усилий, не давая никаких преимуществ, поэтому давайте будем использовать
    # значения по умолчанию.
    password = serializers.CharField(
        max_length=128,
        min_length=8,
        write_only=True
    )
+
+    # Когда поле должно обрабатываться как сериализатор, мы должны явно указать это.
+    # Кроме того, `UserSerializer` никогда не должен выдавать информацию о профиле,
+    # поэтому мы установим свойство `write_only=True`.
+    profile = ProfileSerializer(write_only=True)
+
+    # Мы хотим получить поля `bio` и `image` из соответствующей модели 
+    # Profile.
+    bio = serializers.CharField(source='profile.bio', read_only=True)
+    image = serializers.CharField(source='profile.image', read_only=True)

    class Meta:
        model = User
-        fields = (‘email’, ‘username’, ‘password’, ‘token’,)
+        fields = (
+            'email', 'username', 'password', 'token', 'profile', 'bio',
+            'image',
+        )

    # …
```

Наконец изменим метод `update` `UserSerializer`, добавив обработку данных профиля.

```python
def update(self, instance, validated_data):
    """Метод осуществляет обновление модели User."""

    # Для паролей не должен использоваться метод `setattr`, в отличие от других полей.
    # Это связано с тем, что Django предоставляет функцию, которая осуществляет хэширование  
    # и добавление солей к паролям, что важно для безопасности приложения. Это означает, что мы должны
    # удалить поле password из словаря `validated_data`, прежде чем обработать данные, хранящиеся в нём.    
    password = validated_data.pop('password', None)

+    # Как и пароли, мы должны обрабатывать профили отдельно от других полей. Для этого
+    # мы удаляем данные профиля из словаря `validated_data`.
+    profile_data = validated_data.pop('profile', {})
+
    for (key, value) in validated_data.items():
        # Для ключей, оставшихся в `validated_data`, мы присвоим их значения атрибутам 
        # текущего экземпляра `User`.
        setattr(instance, key, value)

    if password is not None:
        # Метод `.set_password()` был упомянут выше. Он осуществляет все необходимые операции 
        # для безопасного сохранения пароля, освобождая нас от необходимости заниматься этим.
        instance.set_password(password)

    # Наконец, после обновления всех полей, мы должны явно сохранить 
    # модель. Стоит отметить, что метод `.set_password()` не сохраняет
    # модель.
    instance.save()

+    for (key, value) in profile_data.items():
+        # Мы делаем то же чамое, что делили выше, но в этот раз мы вносим
+        # изменения в модель Profile.
+        setattr(instance.profile, key, value)
+
+    # Сохраняем профиль так же как до этого сохранили пользователя.
+    instance.profile.save()
+
    return instance
```

Со всеми этими изменениями мы готовы предоставить наш новый функционал - профили - нашим пользователям! Для проверки его работоспособности, вернитесь в Postman и осуществите запросы "Current User" и "Update User" в каталоге "Auth", чтобы убедиться, что мы ничего не нарушили во время рефакторинга.

## Что будем делать дальше?

Следующей темой для рассмотрения будет основа нашего приложения - статьи. Неважно будете ли Вы читать или писать их, они являются самой важной частью нашего приложения. Без статей пользователи вообще никак не смогут использовать наше приложение.

В следующей главе мы добавим модель и сериализатор для работы со статьями. Мы рассмотрим новое понятие - набор представлений и добавим новый сигнал в наше API. До встречи!
