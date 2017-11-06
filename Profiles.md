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

## ProfileRetrieveAPIView

## ProfileDoesNotExist

## Получаем объект с помощью Postman

## Обновляем UserRetrieveUpdateAPIView

## Обновляем UserSerializer

## Что будем делать дальше?

Следующей темой для рассмотрения будет основа нашего приложения - статьи. Неважно будете ли Вы читать или писать их, они являются самой важной частью нашего приложения. Без статей пользователи вообще никак не смогут использовать наше приложение.

В следующей главе мы добавим модель и сериализатор для работы со статьями. Мы рассмотрим новое понятие - набор представлений и добавим новый сигнал в наше API. До встречи!
