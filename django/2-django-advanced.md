# Django avancé

--------------------------------------------------------------------------------

# Introduction aux tests automatisés

--------------------------------------------------------------------------------

## Arborescence

Une architecture possible pour organiser les tests d'une application consiste à
créer, dans un dossier ``tests``, un fichier de tests
(``test_views.py``, ``test_models.py``, ...) par fichier de
l'application (``views.py``, ``models.py``, ...).

    !console
    ├── library
    │   ├── __init__.py
    │   ├── forms.py
    │   ├── models.py
    │   ├── views.py
    │   ├── tests
    │   │   ├── __init__.py
    │   │   ├── test_forms.py
    │   │   ├── test_models.py
    │   │   └── test_views.py

Les tests automatisés sont les premières briques indispensables pour garantir
une application fiable et éviter les régressions au fil du temps.

--------------------------------------------------------------------------------

## Tester un modèle - Le modèle

Voici un modèle très basique :

    !python
    # models.py

    class Author(models.Model):
        firstname = models.CharField(max_length=100, null=True, blank=True)
        lastname = models.CharField(max_length=100)

        def __str__(self):
            return '%s %s' % (self.firstname, self.lastname)

L'idée n'est pas de tester Django (création d'instance, vérification que les 
différents fonctionnent, ...) mais bien de tester notre code personnel. Ici,
seule la fonction ``__str__`` est donc à tester.

Il faut prendre soin de tester les différents cas possibles d'exécution ( en
l'occurrence, la présence d'un ``firstname`` ou non).


--------------------------------------------------------------------------------

## Tester un modèle - Le test

    !python
    # tests/test_models.py
    from django.test import TestCase
    from library.models import Author


    class AuthorAsStringTest(TestCase):

        def test_with_first_name(self):

            author = Author.objects.create(firstname='René',
                                           lastname='Descartes')
            self.assertEqual(str(author), 'René Descartes')

        def test_without_first_name(self):

            author = Author.objects.create(lastname='Platon')
            self.assertEqual(str(author), 'Platon')

--------------------------------------------------------------------------------

## Exécution des test

    !shell
    $ ./manage.py test library
    Creating test database for alias 'default'...
    .F
    ======================================================================
    FAIL: test_without_first_name (library.tests.test_models.AuthorAsStringTest)
    ----------------------------------------------------------------------
    Traceback (most recent call last):
      File "/home/al/makina-slides/django/example_projects/libraryproject/library/tests/test_models.py", line 16, in test_without_first_name
        self.assertEqual(str(author), 'Platon')
    AssertionError: 'None Platon' != 'Platon'
    - None Platon
    + Platon

    ----------------------------------------------------------------------
    Ran 2 tests in 0.002s

    FAILED (failures=1)
    Destroying test database for alias 'default'...

--------------------------------------------------------------------------------

## Tester un modèle - Le modèle corrigé

    !python
    class Author(models.Model):
        firstname = models.CharField(max_length=100, null=True, blank=True)
        lastname = models.CharField(max_length=100)

        def __str__(self):
            if self.firstname:
                return '%s %s' % (self.firstname, self.lastname)
            else:
                return self.lastname

Exécution des tests :

    !shell
    $ ./manage.py test library.tests.test_models
    Creating test database for alias 'default'...
    ..
    ----------------------------------------------------------------------
    Ran 2 tests in 0.001s

    OK
    Destroying test database for alias 'default'...

--------------------------------------------------------------------------------

## Tester une vue - Le test

    !python
    from datetime import date
    from django.test import TestCase
    from library.models import Book


    class BookViewTest(TestCase):

        def test_recent_books_view(self):
            recent_date = date.today()
            recent_book = Book.objects.create(title='Titre du livre récent',
                                              published=recent_date)

            old_date = recent_date.replace(year=recent_date.year - 1)
            old_book = Book.objects.create(title='Titre du vieux livre',
                                           published=old_date)

            response = self.client.get("/library/recent/")

            self.assertNotContains(response, old_book.title)
            self.assertContains(response, recent_book.title)

--------------------------------------------------------------------------------

## Tester une vue - Le modèle

    !python
    class Book(models.Model):
        title = models.CharField(max_length=200)
        published = models.DateField(null=True, blank=True)

--------------------------------------------------------------------------------

## Tester une vue - La vue

    !python
    from django.shortcuts import render
    from datetime import date
    from .models import Book


    def recent_books(request):
        today = date.today()
        threshold = today.replace(year=today.year - 1)

        results = Book.objects.filter(published__gt=threshold)

        return render(request, 'library/book_list.html', {
            'results': results
        })

On a vérifié :

1. que la vue fonctionne correctement (pas d'erreur 500);
2. que la requête renvoit les résultats attendus.


--------------------------------------------------------------------------------

## Tester un formulaire - Le formulaire

    !python
    # forms.py
    from django import forms

    class PeriodForm(forms.Form):
        begin = forms.DateField()
        end = forms.DateField()

        def __init__(self, *args, **kwargs):
            super(PeriodForm, self).__init__(*args, **kwargs)

            begin = self.initial.get('begin', None)
            if begin:
                self.initial['end'] = begin.replace(month=begin.month + 1)


Comme pour le modèle, l'idée n'est pas de tester ce qui est du ressort de Django.
Ici, il est simplement nécessaire de s'assurer que la fonction ``__init__`` 
fonctionne correctement dans les différents cas possibles (présence ou non d'une valeur
initiale pour le champ ``begin``).

--------------------------------------------------------------------------------

## Tester un formulaire - Le test

    !python
    # tests/test_forms.py
    from datetime import date
    from django.test import TestCase
    from library.forms import PeriodForm


    class PeriodFormTest(TestCase):

        def test_init_without_begin(self):
            f = PeriodForm()
            self.assertIsNone(f.initial.get('end'))

        def test_init_with_begin(self):
            initial = {'begin': date(2014, 1, 1)}
            f = PeriodForm(initial=initial)
            self.assertEqual(f.initial.get('begin'), date(2014, 1, 1))
            self.assertEqual(f.initial.get('end'), date(2014, 2, 1))

--------------------------------------------------------------------------------

## Rapport de couverture

Révèle quelles parties du code sont couvertes par les tests

    !shell
    $ pip install coverage
    $ coverage run --branch --source=library ./manage.py test
    $ coverage report
    Name                                 Stmts   Miss Branch BrPart  Cover
    ----------------------------------------------------------------------
    library/__init__.py                      0      0      0      0   100%
    library/admin.py                         1      0      0      0   100%
    library/apps.py                          3      0      0      0   100%
    library/forms.py                         9      0      2      0   100%
    library/migrations/0001_initial.py       6      0      0      0   100%
    library/migrations/__init__.py           0      0      0      0   100%
    library/models.py                       11      0      2      0   100%
    library/tests/__init__.py                0      0      0      0   100%
    library/tests/test_forms.py             12      0      0      0   100%
    library/tests/test_models.py             9      0      0      0   100%
    library/tests/test_views.py             12      0      0      0   100%
    library/urls.py                          3      0      0      0   100%
    library/views.py                         8      0      0      0   100%
    ----------------------------------------------------------------------
    TOTAL                                   74      0      4      0   100%


--------------------------------------------------------------------------------

# Gestion des utilisateurs 

--------------------------------------------------------------------------------

## Principaux concepts

La gestion des utilisateurs Django est principalement gérée par le module
``django.contrib.auth``. Ce module introduit plusieurs concepts :

* ``User`` : classe représentant un utilisateur Django ;
* ``Permission`` : classe d'assigner à un utilisateur le droit de faire une certaine action ou non (booléen);
* ``Group`` : classe permettant d'associer plusieurs permissions à un sous-ensemble d'utilisateurs ;
* des vues spécifiques (connexion, déconnexion, ...) ;
* des formulaires (CRUD, connexion, changement de mot de passe, ...) ;
* un *backend* d'authentification souple et personnalisable.

Pour disposer de cette fonctionnalité, le module ``django.contrib.auth`` doit
être présent dans les ``INSTALLED_APPS`` du projet (cf ``settings.py``).

--------------------------------------------------------------------------------

## Les utilisateurs

La classe ``User`` est le coeur du système d'authentification Django. Une instance
de ``User`` représente un utilisateur, une personne qui interagit avec le site. Elle
permet plusieurs choses comme :

* gérer des restriction d'accès ;
* personnaliser des profils utilisateurs ;
* associer des contenus à leur créateur.

### Quelques propriétés

La classe ``User`` fournit quelques propriétés de base comme ``first_name``, ``last_name``,
``username``, ``password``, ``email``. D'autres propriétés, plus *fonctionnelles*,
sont à connaitre :

* ``is_active`` : booléen précisant si le compte est actif ;
* ``is_staff`` : booléen précisant si l'utilisateur peut accéder à l'interface d'administration ;
* ``is_superuser`` : booleén spécifiant si l'utilisateur est un super-utilisateur.

--------------------------------------------------------------------------------

## Les permissions

Django fournit un système de permissions assez simple. Il consiste à assigner des
permissions particulières à des utilisateurs ou/et à des groupes.

L'interface d'administration peut notamment utiliser les permissions génériques
``add``, ``change`` et ``delete`` sur chaque modèle existant dans le projet Django.

Pour le modèle ``my_model`` de l'application ``my_app``, les permissions suivantes
pourront être créées par un ``./manage.py syncdb`` :

* 'my_app.add_my_model'
* 'my_app.change_my_model'
* 'my_app.delete_my_model'

Il est aussi possible de créer ses propres permissions.

Quelques fonctions de la classe ``User`` permettent de travailler avec ces permissions,
notamment :

* ``user.get_all_permissions()``
* ``user.has_perm(perm)``

--------------------------------------------------------------------------------

## Les groupes

L'objectif des groupes et de catégoriser des sous-ensembles d'utilisateurs
afin de leur assigner une liste commune de permissions. Exemple :

* *Rédacteur* : permissions d'ajout/modification/suppression d'articles
* *Administrateur* : permissions d'ajout/modification/suppression d'utilisateurs
* ...

--------------------------------------------------------------------------------

## Les vues

Django apporte nativement quelques vues facilitant l'authentification et la gestion
du mot de passe des utilisateurs, principalement :

* ``login``
* ``logout``
* ``logout_then_login``
* ``password_change``
* ``password_reset``
* ...

Quelques settings permettent aussi de simplifier cette gestion :

* ``LOGIN_URL`` : URL vers la vue de connexion
* ``LOGIN_REDIRECT_URL`` : URL de redirection après l'authentification de l'utilisateur
* ``LOGOUT_URL`` : URL de la vue de déconnexion

--------------------------------------------------------------------------------

## Les formulaires

Sans utiliser directement les vues prêtes à l'emploi, il est aussi possible de baser
des vues personnalisées sur des formulaires présents dans la bibliothèque
``django.contrib.auth.forms``.

Ces formulaires réalisent de base certaines vérifications très utiles (unicité du nom 
d'utilisateur, vérification de la ressaisie du mot de passe, ...).

* ``AuthenticationForm`` : formulaire d'authentification
* ``UserChangeForm`` : formulaire d'édition du compte utilisateur
* ``PasswordChangeForm`` : formulaire de changement de mot de passe
* ``PasswordResetForm`` : formulaire de réinitialisation de mot de passe
* ``SetPasswordForm`` : formulaire de création de mot de passe

--------------------------------------------------------------------------------

## Backend d'authentification

La bibliothèque ``django.contrib.auth.backends`` apporte un système de *backend*
d'authentification relativement simple et très souple.

Deux *backends* par défaut sont disponibles:

* ``ModelBackend`` : *backend* d'authentification par défaut utilisant le nom d'utilisateur / mot de passe de l'utilisateur
* ``RemoteUserBackend`` : permet de gérer une authentification depuis une source externe via les entête HTTP

### Écrire un backend personnalisé

Il est assez facile d'écrire son propre *backend* pour personnaliser l'authentification
des utilisateurs en écrivant une simple classe qui implémente certaines fonctions comme :

* ``authenticate``
* ``get_user``
* ...

--------------------------------------------------------------------------------

# Tutoriel : Mettre en place la (dé)connexion et créer un groupe avec le droit d'administrer les tâches

.fx: alternate

--------------------------------------------------------------------------------

# Aller plus loin avec les modèles

--------------------------------------------------------------------------------

# Le concept ``Queryset``

### Rappel

Un ``Queryset`` représente une collection d'objets provenant de la base de données. Cette collection peut être filtrée, limitée, ordonnée, ... grâce à des méthodes qui correspondent à des clauses SQL.

Il est donc possible de construire des requêtes en base de données via ce QuerySet.

### Exemple

    !python
    >>> Book.objects.filter(title__icontains='django') \
                    .exclude(relase__lte=date('2014-01-01')) \
                    .order_by('price')

--------------------------------------------------------------------------------

## ``QuerySet`` avancés - `values()`

Cette méthode retourne un ``ValuesQuerySet`` qui liste des dictionnaires plutôt que des instances du modèle. Chaque dictionnaire représente une instance ; ses clés correspondent aux attributs de l'instance. Il est possible de spécifier les clés que l'on souhaite récupérer.

    !python
    >>> Book.objects.filter(name__icontains='django') \
                    .values('title' , 'release')

    [{'title': 'Two scoops of django', 'release': date(2013, 08, 31)}, ]


Un ``ValuesQuerySet`` peut être très intéressant quand le nombre d'attributs dont on a besoin est faible, car on évite de charger les instances sous forme de modèles python.

Attention, dans le cas d'un attribut de type ``ForeignKey``, la clé et la valeur retournées seront le nom de le colonne et la valeur trouvée en base de données (ex: ``'author_id': 12``)

--------------------------------------------------------------------------------

## ``QuerySet`` avancés - `values_list()`

Cette méthode est semblable à la précédente mais elle retourne une liste de tuples plutôt qu'une liste de dictionnaires.

    !python
    >>> Book.objects.filter(name__icontains='django') \
                    .values_list('title' , 'release')

    [('Two scoops of django', date(2013, 08, 31)),
     ('Django avancé', date(2013, 05, 15))]

Si un seul attribut est précisé, il est possible d'ajouter le paramètre ``flat=True``
pour obtenir une liste non imbriquée.

    !python
    >>> Book.objects.filter(name__icontains='django') \
                    .values_list('title', flat=True)

    ['Two scoops of django', 'Django avancé']

--------------------------------------------------------------------------------

##  ``QuerySet`` avancés -méthodes à retenir

* ``count()``: retourne le nombre d'éléments dans le queryset
* ``exists()``: retourne True si le queryset n'est pas vide
* ``first()`` et ``last()`` : retourne le premier ou dernier élément du queryset
* ``latest(field_name=date)`` et ``earliest()`` : retourne le premier ou dernier élément du queryset en fonction d'un champ date

### La méthode ``create()``

La méthode basique pour créer une instance est d'instancier le modèle puis de 
faire appel à la méthode ``save()`` de cette instance, mais une autre solution très pratique existe.

Elle permet de réaliser l'opération ci-dessus en une seule instruction :

    !python
    book = Book.objects.create(title="New django book", price="42€")


--------------------------------------------------------------------------------

## ``QuerySet`` avancés - raccourcis

### La méthode ``get_or_create()``

La méthode ``get_or_create()`` permet de tenter de récupérer un objet 
(via ``get()``), et de le créer si il n'existe pas. Elle retourne un tuple 
comprenant un booléen qui précise si l'instance vient d'être créée, et l'instance elle-même :

    !python
    book, created = Book.objects.get_or_create(
        title="New django book", date(2013, 05, 15),
        defaults={'price': '42€'})

Les valeurs passées directement en paramètres sont utilisées lors de l'appel 
de la méthode ``get()``, les valeurs passées dans ``defaults`` sont utilisées 
lors de la création éventuelle de l'instance pour initialiser la valeur des 
propriétés correspondantes.

--------------------------------------------------------------------------------

# Le concept ``Manager``

### Rappel

Un ``Manager`` est l'interface à travers laquelle les opérations de requêtage en base de données sont mises à disposition d'un modèle Django. Chaque modèle possède un ``Manager`` par défaut accessible via la propriété ``objects``.

    !python

    from django.db import models

    class Book(models.Model):
        #...
        objects = models.Manager()

Ce ``Manager`` par défaut fournit nativement quelques méthodes très souvent utilisées, comme :

    !python
    >>> Book.objects.get(pk=12)
    >>> Book.objects.all()
    >>> Book.objects.filter(title__icontains='django')
    >>> Book.objects.exclude(date__lt=date(2013, 01, 01))

--------------------------------------------------------------------------------

## Manager personnalisé

### Objectif

Il peut cependant être utile d'écrire son propre ``Manager`` pour principalement
deux raisons :

* ajouter des méthodes supplémentaires ;
* modifier le ``QuerySet`` initial retourné par le ``Manager``.

### Méthode

Un ``Manager`` personnalisé est une classe héritant de ``Manager`` que l'on instancie dans un attribut du modèle.


    !python
    from django.db import models

    class CustomBookManager(models.Manager):
        # ...

    class Book(models.Model):
        #...
        custom_books = CustomBookManager()

--------------------------------------------------------------------------------

## Manager - Ajouter des méthodes supplémentaires

Écrire un ``Manager`` personnalisé est la bonne solution pour ajouter des méthodes de niveau *table* (qui renvoit des informations sur un ensemble d'instances) contrairement aux méthodes du modèle dites de niveau *ligne* (qui renvoit des informations sur une instance).

    !python
    from django.db import models

    class AuthorManager(models.Manager):
        def with_nb_books(self):
            return self.get_queryset() \
                       .annotate(nb_books=Count('books')) \
                       .order_by('nb_books')

    class Author(models.Model):
        # ...
        objects = AuthorManager()
<!-- -->

    !shell
    >>> authors_with_nb_books = Author.objects.with_nb_books()
    >>> authors_with_nb_books[0]
    <Author : John Doe>
    >>> authors_with_nb_books[0].nb_books
    42

--------------------------------------------------------------------------------

## Manager - Modifier le ``QuerySet`` initial

La méthode ``Manager.get_queryset()`` renvoit un ``QuerySet`` par défaut qui correspond à ``Model.objects.all()``. 

Il peut être intéressant de créer un ``Manager`` pour surcharger cette méthode et retourner un ``QuerySet`` personnalisé.

    !python
    from django.db import models

    class EnglishBookManager(models.Manager):
        def get_queryset(self):
             return super().get_queryset().filter(lang='EN')

    class FrenchBookManager(models.Manager):
        def get_queryset(self):
             return super().get_queryset().filter(lang='FR')

    class Book(models.Model):
        #...
        objects = models.Manager()
        english_books = EnglishBookManager()
        french_books = FrenchBookManager()

--------------------------------------------------------------------------------

## Manager - Modifier le ``QuerySet`` initial

## Attention !

Quand on surcharge le ``QuerySet`` initial, il est souvent préférable de ne pas remplacer l'attribut ``objects`` par le ``Manager`` personnalisé. 

En effet, ``objects`` est le ``Manager`` utilisé par défaut (dans l'administration par exemple). Il est donc très risqué d'altérer son comportement.

En revanche, remplacer ``objects`` par un ``Manager`` personnalisé qui ne fait qu'ajouter des méthodes personnalisées ne pose pas de problème, puisque le comportement naturel n'est pas altéré.

--------------------------------------------------------------------------------

# Tutoriel : Écrire un ``Manager`` personnalisé permettant de lister les tâches urgentes et non réalisées

.fx: alternate

--------------------------------------------------------------------------------

# L'héritage de modèles

--------------------------------------------------------------------------------

## L'héritage par classe abstraite

Ce type d'héritage est souvent utilisé pour mettre en commun un certain nombre d'informations et/ou de comportements entre plusieurs modèles. Cette classe abstraite ne sera pas utilisé de manière autonome.

### Caractéristiques

* La classe abstraite ne donnera pas lieu a une création de table en base de données ;
* Chaque champ de classe abstraite est présente dans chaque modèle qui en hérite ;
* Il n'est pas possible de faire des requêtes sur la classe abstraite.


--------------------------------------------------------------------------------

## L'héritage par classe abstraite - Exemple

    !python
    # models.py

    class CommonInfo(models.Model):
        creation_date = models.DateField()
        modification_date = models.DateField()

        class Meta:
            abstract = True

    class Book(CommonInfo):
        title = models.CharField(max_length=100)
        # ...

    class Author(CommonInfo):
        name = models.CharField(max_length=100)
        # ...


--------------------------------------------------------------------------------

## L'héritage traditionnel (multi-tables)

Pour spécialiser un modèle déjà existant (éventuellement d'une application externe) ou/et si on souhaite que les modèles aient des tables séparées, il faut utiliser l'héritabe multi-tables.

### Caractéristiques

* Chaque modèle a sa propre table en base de données ;
* Il est possible de requêter chaque modèle indépendamment ;
* La relation est représentée par un champ ``OneToOneField``


--------------------------------------------------------------------------------

## L'héritage traditionnel (multi-tables) - Exemple

    !python
    # models.py

    class Book(models.Model):
        title = models.CharField(max_length=100)
        author = models.ForeignKey(Author)

    class Comic(Book):
        illustrator = models.CharField(max_length=100)
        # ...

    class Biography(Book):
        personage = models.CharField(max_length=100)
        # ...

--------------------------------------------------------------------------------

## L'héritage "proxy"

Grâce aux modèles *proxy*, il est possible de modifier le comportement d'un objet (``Manager`` personnalisé, ajout d'une méthode, ...) sans toucher aux données (champs) et donc sans créer une nouvelle table pour ce modèle dérivé.

### Caractéristiques

* Le modèle *proxy* n'engendre pas de nouvelle table en base de données ;
* Le modèle *proxy* et son modèle parent travaille sur la même table ;
* Il est possible de le requêter de manière indépendante.

--------------------------------------------------------------------------------

## L'héritage "proxy" - Exemple

    !python
    # models.py

    class Book(models.Model):
        title = models.CharField(max_length=100)
        author = models.ForeignKey(Author)

    class OrderedByAuthorBook(Book):

        class Meta:
            proxy = True
            ordering = ['author']

--------------------------------------------------------------------------------

# Tutoriel : Stocker les dates de création/modification pour les listes et les tâches. 

.fx: alternate

--------------------------------------------------------------------------------

# ORM et performance

--------------------------------------------------------------------------------

## Perf ORM: Le problème N+1 avec les ForeignKey

On accède à une relation dans une boucle ce qui entraine :

* **1** requête pour récupérer la collection de taille N sur laquelle on boucle
* **N** requêtes pour récupérer l'attribut lié

Exemple :

     !html+django
     {% for task in object_list %}
     <li>
       <a href="{% url 'task_detail' task.pk %}">{{ task }}</a>
       Liste: {{task.todo_list.label }}
     </li>
     {% endfor %}

---

## Perf ORM: Diagnostique avec Django Debug Toolbar

![](img/ddt_nplus1.png)

---

## Perf ORM: Solution avec select_related

    !python
    class TaskList(ListView):
        model = Task

        def get_queryset(self):
            queryset = super(TaskList, self).get_queryset()
            return queryset.select_related("todo_list")

L'ORM fait une seule requête avec une jointure :

![](img/ddt_nplus1_fixed.png) 

---

## Perf ORM: Le problème N+1 avec ManyToMany

    !html+django
    {% for list in object_list %}
      <li>
        <a href="{% url 'todolist_detail' list.pk %}">{{ list }}</a>
        Users: {% for user in list.users.all %}
                  {{ user.username }}
               {% endfor %}
      </li>
    {% endfor %}

![](img/ddt_nplus1_manytomany.png)

---

## Perf ORM: Solution avec prefetch_related


    !python
    class TodoListList(ListView):
        model = TodoList

        def get_queryset(self):
            queryset = super(TodoListList, self).get_queryset()
            return queryset.prefetch_related("users")

L'ORM ne fait qu'une seule requête supplémentaire avec une clause ``IN`` :

![](img/ddt_nplus1_manytomany_fixed.png)

-----------------

## Le cache

Lorsque les requêtes sont lentes même si elles sont optimisées, il reste le cache.

Plusieurs possibilités de backend :

* memcached
* base de données
* fichiers
* mémoire (RAM)
* autre (API extensible, redis par ex)

Plusieurs niveaux :

  * par site (middleware)
  * par vue (décorateur)
  * par un tag dans les templates
  * par l'API `from django.core.cache import cache` puis utilisation des fonctions `cache_get()`, `cache_set()`, `cache_get_or_set()`

Documentation : <https://docs.djangoproject.com/fr/1.10/topics/cache/>

------------------------------------------

# Les signaux

Permet d'appeler du code quand certains événements se produisent dans l'application :

* initialisation de l'application (pre_init / post_init)
* écritures dans la base de données (pre_save / post_save / pre_delete ...)
* traitement des requêtes (request_started / request_finished)

Exemple :


    !python
    from django.db.models.signals import post_save
    from django.dispatch import receiver
    from todo.models import Task

    @receiver(post_save, sender=Task)
    def db_update_callback(sender, instance, created, **kwargs):
        print('Task "{0}" saved!'.format(instance.name))


---

# Tutoriel : Envoyer un email quand une tâche est effectuée

.fx: alternate

--------------------------------------------------------------------------------

# Aller plus loin avec les vues

--------------------------------------------------------------------------------

## Les vues basées sur des classes

Les vues basées sur des classes possèdent des avantages sur les vues classiques :

* possibilité d'organiser le code dans différentes méthodes (notamment selon la méthode HTTP entrante) ;
* possibilité d'utiliser l'héritage et les *mixins* pour factoriser et réutiliser le code.

Ces vues sont aussi généralement écrites dans le fichier ``views.py`` de l'application.

### Un exemple tiré de la documention Django

    !python
    # some_app/views.py
    from django.views.generic import TemplateView

    class AboutView(TemplateView):
        template_name = "about.html"

--------------------------------------------------------------------------------

## Les vues basées sur des classes - Worfklow de base

* La méthode ``as_view()`` est appelée par l'``URLDispatcher`` ;
* Cette méthode instancie la classe et appelle la méthode ``dispatch()`` de l'instance créée ;
* Celle-ci appelle la méthode ``get()``, ``post()``, ... en fonction de la méthode HTTP entrante (GET, POST, ...) ;
* Le traitement qui suit dépend du cas d'utilisation, puis une ``HttpResponse`` est relayée par ``dispatch()``.

La documentation : <https://docs.djangoproject.com/fr/1.10/topics/class-based-views/>

Vue d'ensemble : <http://ccbv.co.uk/>

--------------------------------------------------------------------------------

## Function-based vs. Class-based views

### Class-based views

Il faut probablement utiliser une vue basée sur une classe ... 

* si une des classes de vues génériques fournies par Django s'approche vraiment du besoin
* si la vue peut être créée par héritage d'une autre en surchargeant seulement des attributs
* si la vue à créer peut être réutilisée par héritage et avec peu de modifications par la suite

### Function-based views

Il faut probablement utiliser une vue basée sur une fonction ... 

* si une implémentation basée sur une classe semble complexe
* si la vue n'a pas vocation à être réutilisée

--------------------------------------------------------------------------------

## Passage aux vues basées sur des classes

### Vue simple basée sur une fonction

    !python
    from django.http import HttpResponse

    def my_view(request):
        if request.method == 'GET':
            # traitements
            return HttpResponse('result')

### Vue simple basée sur une classe

    !python
    from django.http import HttpResponse
    from django.views.generic.base import View

    class MyView(View):
        def get(self, request):
            # traitements
            return HttpResponse('result')

--------------------------------------------------------------------------------

## Quelques classes de base

### Les vues basiques

Dans ``django.views.generic.base`` :

* ``View`` est la classe *mère* et fourni le workflow vu précédemment.
* ``TemplateView`` est une classe permettant très simplement de faire le rendu d'une template.

### Les vues permettant de traiter un formulaire

Dans ``django.views.generic.edit`` :

* ``FormView`` facilite la gestion d'un formulaire en permettant une bonne organisation du code et en limitant l'indentation.

--------------------------------------------------------------------------------

## Quelques classes de base

### Les vues permettant d'afficher des instances

Dans ``django.views.generic`` :

* ``ListView`` permet de lister très simplement des instances d'un modèle.
* ``DetailView`` permet d'afficher le détail d'une instance d'un modèle.

### Les vues permettant de modifier des instances

Dans ``django.views.generic.edit`` :

* ``CreateView`` et ``UpdateView`` sont très utiles pour la création/modification d'instance, de l'affichage du formulaire jusqu'à l'enregistrement de l'instance.
* ``DeleteView`` facilite l'implémentation de vues pour la suppression d'intance.

--------------------------------------------------------------------------------

# Protéger une vue

## Les décorateurs

Les décorateurs sont des fonctions Python dont le rôle est de modifier le comportement par défaut d'autres fonctions ou classes. 

Il est par exemple possible de protéger une vue avec un ou plusieurs décorateurs.

## Quelques décorateurs utiles

* ``require_http_methods`` permet de limiter l'accès à une vue sur certaines méthodes HTTP précises ;
* ``login_required`` permet de limiter l'accès à une vue aux utilisateurs connectés ;
* ``permission_required`` permet de limiter l'accès à une vue aux utilisateurs possédant la permission précisée.

--------------------------------------------------------------------------------

## Protéger une vue basée sur une fonction

### Exemple avec ``login_required``

    !python
    from django.contrib.auth.decorators import login_required

    @login_required()
    def my_view(request):
        # Seul un utilisateur connecté peut accéder à cette vue
        # ...

### Exemple avec ``require_http_methods``

    !python
    from django.contrib.auth.decorators import login_required

    @require_http_methods(["GET", "POST"])
    def my_view(request):
        # On ne peut pas accéder à cette vue qu'en GET ou POST
        # ...

--------------------------------------------------------------------------------

## Protéger une vue basée sur une classe

### Dans la vue

    !python
    # views.py
    from django.contrib.auth.decorators import login_required
    from django.utils.decorators import method_decorator

    class MyProtectedView(TemplateView):
        template_name = 'my_protected_view.html'

        @method_decorator(login_required)
        def dispatch(self, *args, **kwargs):
            return super(MyProtectedView, self).dispatch(*args, **kwargs)

### Dans l'``URLConf``

    !python
    # urls.py
    from django.contrib.auth.decorators import login_required
    from my_app.views import MyProtectedView

    urlpatterns = patterns('',
        (r'^secret/', login_required(MyProtectedView.as_view())),
    )

--------------------------------------------------------------------------------

# Tutoriel : Limiter l'accès aux vues aux utilisateurs connectés

.fx: alternate

--------------------------------------------------------------------------------

# Gérer les erreurs

--------------------------------------------------------------------------------

## Retourner une erreur 404

Il est important de gérer les erreurs selon les concepts du protocole HTTP. Une ressource non trouvée sur un site doit donc retourner une erreur de type 404. On utilise pour cela une exception de type ``Http404``.

    !python
    from django.http import Http404

    def book_detail(request, book_id):
        try:
            book = Book.objects.get(pk=book_id)
        except Book.DoesNotExist:
            raise Http404

        return render(request, 'library/book_detail.html', {
            'book': book
        })

Bon c'est assez fastidieux donc il existe un raccourci :

    !python
    from django.shortcuts import get_object_or_404, render

    def book_detail(request, book_id):
        book = get_object_or_404(Book, pk=book_id)
        return render(request, 'library/book_detail.html', {'book': book})

--------------------------------------------------------------------------------

## Retourner une erreur 403

Comme pour l'erreur 404, Il est important de gérer les erreurs en accord avec le protocole HTTP. Un problème de permission doit donc engendrer une erreur de type 403. On utilise pour cela une exception de type ``PermissionDenied``.

    !python
    from django.core.exceptions import PermissionDenied

    def book_detail(request, book_id):
        
        if not library.is_registered(user):
            raise PermissionDenied
        
        book = Book.objects.get(pk=book_id)
        
        return render(request, 'books/detail.html', {'book': book})

--------------------------------------------------------------------------------

## Affichage par défaut d'une erreur

Plusieurs types d'erreurs peuvent être générées manuellement ou automatiquement par Django, principalement : 400, 403, 404 et 500.

Par défaut, chaque erreur correspond à une vue dont Django fait le rendu quand l'exception est levée :

* Erreur 400 : ``django.views.defaults.bad_request``
* Erreur 403 : ``django.views.defaults.permission_denied``
* Erreur 404 : ``django.views.defaults.page_not_found``
* Erreur 500 : ``django.views.defaults.server_error``

### Mode ``debug``

Le réglage ``DEBUG`` (dans ``settings.py``) permet d'activer ou non
l'affichage de la page de débogage pendant le développement. Cette page vient en
remplacement des vues d'erreurs listées ci-dessus. Il est donc important de
la désactiver en production.

--------------------------------------------------------------------------------

## Personnaliser l'affichage d'une erreur

### Avec un template personnalisé

Pour personnaliser simplement l'affichage, il suffit de nommer la template
403.html, 404.html, ... et Django fera le rendu de cette template automatiquement.

### Avec une vue personnalisée

Si on souhaite que le traitement de l'erreur soit plus complexe, il est possible
de créer une vue dont il faudra préciser le nom dans l'``URLConf`` :

    !python
    # views.py
    def my_403_view(request):
        send_mail_to_admin()
        # ...
        return render(request, '403.html')

    # urls.py
    urlpatterns = [
        # ...
    ]
    handler403 = 'my_app.views.my_403_view'

--------------------------------------------------------------------------------

# Tutoriel : Personnaliser la page d'erreur 404

.fx: alternate

---

# Middleware

Altérer le traitement des requêtes de manière globale

    !python
    class SimpleMiddleware(object):
        def __init__(self, get_response):
            # Initialisation
            self.get_response = get_response

        def __call__(self, request):

            # Code exécuté pour chaque requête avant la vue

            response = self.get_response(request)

            # Code exécuté pour chaque requête après la vue

            return response

--------------------------------------------------------------------------------

## Middleware activés par défaut

    !python
    MIDDLEWARE = [
        'django.middleware.security.SecurityMiddleware',
        'django.contrib.sessions.middleware.SessionMiddleware',
        'django.middleware.common.CommonMiddleware',
        'django.middleware.csrf.CsrfViewMiddleware',
        'django.contrib.auth.middleware.AuthenticationMiddleware',
        'django.contrib.messages.middleware.MessageMiddleware',
        'django.middleware.clickjacking.XFrameOptionsMiddleware',
    ]

Voir la documentation des [middlewares standards](https://docs.djangoproject.com/en/dev/ref/middleware/)

### Exemple de middleware

Le middleware `MessageMiddleware` permet d'afficher des messages au utilisateurs.
Ceux-ci sont enregistrés dans la session et sont affichés la prochaine fois qu'une page HTML est affichée.

L'ordre est donc important, `SessionMiddleware` doit être placé avant `MessageMiddleware`.

----------------------------------------

# TP: écrire un middleware qui affiche un message important lorsque des tâches sont dépassées

.fx: alternate

# Presenter notes

On prendra soin de ne faire qu'une seule requête avec values_list()

----------------------------------------

# Aller plus loin avec les templates

--------------------------------------------------------------------------------

## Chaînes de caractères sécurisées et échappement

###  Chaîne de caractère sécurisée

Une chaîne est dite sécurisée quand elle a été marquée comme n'ayant pas besoin
d'échappement lors d'un rendu HTML, c'est à dire que les caractères qui ne
doivent pas être interprétés par le moteur HTML ont déjà été transformés en leurs 
entités appropriées.
Une chaîne sécurisée est représentée par un objet ``SafeString``.

### Échappement

Plusieurs filtres et tag permettent de gérer l'échappement des chaînes :

* ``escape`` : transforme les caractères HTML d'une chaîne en entités (ex: "<" en "``&lt;``")
* ``safe`` : marque la chaîne comme n'ayant pas besoin d'échappement
* ``{% autoescape on|off %}`` : précise si les chaînes doivent être ou non échappées systématiquement à l'intérieur de ce bloc

--------------------------------------------------------------------------------

# Écrire un filtre personnalisé

--------------------------------------------------------------------------------

## Arborescence des fichiers

Les filtres personnalisés doivent être écrits dans une module ``templatetags``
de l'application.


    !console
    ├── my_app
    │   ├── __init__.py
    │   ├── admin.py
    │   ├── forms.py
    │   ├── models.py
    │   ├── templatetags
    │   │   ├── __init__.py
    │   │   └── my_todo_app_filters.py


Le nom du fichier en lui-même n'a pas d'importance, mais il sera utilisé pour
charger les filtres au niveau des templates.

--------------------------------------------------------------------------------

## Structure d'un filtre

Un filtre personnalisé est une simple fonction python qui prend un ou deux arguments :

* la valeur de la variable dont on veut modifier l'affichage (pas nécessairement
une chaîne de caractères)
* un argument optionnel (qui peut avoir une valeur par défaut ou non)

Un filtre sans argument supplémentaire sera appelé de la manière suivante :

    !python
    {{ variable|my_simple_filter }}

et un filtre avec argument sera utilisé ainsi :

    !python
    {{ variable|my_filter:"foo" }}

--------------------------------------------------------------------------------

## Quelques exemples

### Exemple d'un filtre avec un seul argument

    !python
    # filters.py
    def lower(value):
        """Converts a string into all lowercase"""
        return value.lower()

    {# template #}
    {{ variable|lower }}

### Exemple d'un filtre avec deux arguments

    !python
    # filters.py
    def cut(value, arg):
        """Removes all values of arg from the given string"""
        return value.replace(arg, '')

    {# template #}
    {{ variable|cut:'0' }}

--------------------------------------------------------------------------------

## Enregistrement du filtre personnalisé

La classe ``Library`` permet d'ajouter les filtres personnalisés à la bibliothèque
de filtres Django pour pouvoir ensuite les charger et les utiliser dans les
templates. Deux solutions :

### Via un appel de fonction classique

    !python
    from django import template
    register = template.Library()

    def my_filter(value):
        # code du filtre

    register.filter('my_filter', my_filter)

### Via un décorateur

    !python
    from django import template
    register = template.Library()

    @register.filter()
    def my_filter(value):
        # code du filtre

--------------------------------------------------------------------------------

## Filtres et échappement

L'échappement (ou non) de la chaîne retournée en sortie du filtre doit être
contrôlé.

Si le filtre n'introduit pas de caractère HTML (comme "&" ou "<"), il peut être
marqué comme sécurisé au moment de l'enregistrement grâce à l'équipement ``is_safe``.

    !python
    @register.filter(is_safe=True)
    def my_filter(value):
        # code du filtre

Django traîtera alors l'échappement de la chaîne en entrée en sachant que le
filtre personnalisé n'aura pas d'impact.


Il est aussi possible de marqué la chaîne comme sécurisée manuellement en sortie du 
filtre (nécessaire si le filtre introduit du HTML).

    !python
    from django.utils.safestring import mark_safe

    @register.filter(is_safe=True)
    def my_filter(value):
        # code du filtre
        return mark_safe(output)

--------------------------------------------------------------------------------

# Tutoriel : Écrire un filtre personnalisé qui transforme la date d'une tâche en  temps restant pour la réaliser

.fx: alternate

--------------------------------------------------------------------------------

# Aller plus loin avec les formulaires

--------------------------------------------------------------------------------

## Initialiser un formulaire

### Founir des données initiales

Il existe une méthode très simple pour initialiser les champs d'un formulaire :
la méthode ``__init__()`` peut prendre en argument un dictionnaire "``initial``"
dont les clés doivent correspondre aux noms des champs du formulaire.

    !python
    # forms.py
    class AccountForm(forms.Form):
        lastname = forms.CharField(max_length=100)
        firstname = forms.CharField(max_length=100)

    #views.py
    def create_account(request):
        initial = {
            'lastname': request.user.last_name,
            'firstname': request.user.first_name
        }
        form = AccountForm(initial=initial)

--------------------------------------------------------------------------------

## Initialiser un formulaire

### Personnaliser la méthode ``__init__()``

Pour aller plus loin, il est possible de surcharger la méthode ``__init__()``
pour réaliser des traitements particuliers (initialiser des valeurs complexes,
limiter les choix d'un champ select, cacher dynamiquement des champs, ...).

    !python
    # forms.py
    class PeriodForm(forms.Form):
        begin = forms.DateField()
        end = forms.DateField()

        def __init__(self, *args, **kwargs):
            super(PeriodForm, self).__init__(*args, **kwargs)

            begin = self.initial.get('begin', None)
            if begin:
                self.initial['end'] = begin + delta(months=1)


--------------------------------------------------------------------------------

## Valider un champ formulaire

Un formulaire Django dispose d'un mécanisme de validation assez poussé qui consiste
à valider chaque champ un par un, puis à exécuter une méthode réalisant une validation
plus globale.

Un échec de validation doit engendrer une exception de type ``ValidationError``.

### Exemple

Pour valider un champ de formulaire, il suffit de créer une méthode de formulaire
nommée par le nom du champ préfixé de ``clean_``.

    !python
    # forms.py
    class SearchBookForm(forms.Form):
        search_text = forms.CharField(max_length=100)
    
        def clean_search_text(self):
            search_text = self.cleaned_data['search_text']
            if 'django' not in search_text:
                msg = 'You should search Django books :)!'
                raise forms.ValidationError(msg)
            return search_text

--------------------------------------------------------------------------------

## Valider un formulaire de manière globale

Implémenter la méthode ``clean`` permet de faire une validation globale du formulaire,
utile notamment pour faire des vérifications sur plusieurs champs dépendants les uns
des autres.

    !python
    # forms.py
    class PeriodForm(forms.Form):
        begin = forms.DateField()
        end = forms.DateField()

        def clean(self):
            cleaned_data = super(PeriodForm, self).clean()
            begin = cleaned_data.get('begin')
            end = cleaned_data.get('end')

            if begin and end and begin >= end:
                msg = 'End date must be later than begin date!'
                self.add_error('end', msg)
                # ou raise forms.ValidationError(msg)

            return cleaned_data

--------------------------------------------------------------------------------

# Tutoriel : Mettre en place l'initialisation et la validation du formulaire de contact

.fx: alternate

--------------------------------------------------------------------------------

# Gestion des fichiers

--------------------------------------------------------------------------------

## Introduction à la gestion des fichiers statiques

Les sites web ont très souvent besoin de servir des fichiers dits *statiques*, 
principalement des images, CSS et JS.

Django fournit un module ``django.contrib.staticfiles`` qui facilite cette gestion.

### Quelques réglages

Comme toujours, pour que l'application soit utilisable, il faut qu'elle soit présente
dans les ``INSTALLED_APPS`` du projet.

``STATIC_URL`` permet ensuite de spécifier l'URL à partir de laquelle ces fichiers
statiques seront disponibles.

    !python
    # settings.py
    INSTALLED_APPS = (
      ...
      'django.contrib.staticfiles',
      ...
    )

    STATIC_URL = '/static/'


--------------------------------------------------------------------------------

## Gérer les fichiers statiques

### Stockage

Les fichiers statiques doivent être stockés dans un répertoire `static` dans
chaque application. Les scripts par défaut configurés dans ``STATICFILES_FINDERS``
(cf ``settings.py``) pourront alors retrouver les fichiers statiques de chaque application.

Il est aussi possible de stocker des fichiers statiques dans d'autres dossiers, 
il faut alors ajouter ceux-ci à la liste ``STATICFILES_DIRS``.

La commande ``collectstatic`` permet d'aggréger ces fichiers dans un répertoire 
unique défini par ``STATIC_ROOT`` :

    !console
    $ ./manage.py collectstatic

Cette méthode permet aux applications ou au projet de surcharger des fichiers
statiques d'autres applications (l'ordre de découverte est important).

--------------------------------------------------------------------------------

## Utiliser les fichiers statiques

### Dans les templates

Le tag ``{% static %}`` permet de créer une URL dynamiquement vers un fichier
statique, elle sera automatiquement préfixée par `STATIC_URL`.

Exemple pour une image :

    !html+django
    {# my_app/templates/my_app/my_template.html #}
    {% load static %}
    ...
    <img src="{% static "my_app/img/myexample.jpg" %}" alt="My image"/>

Exemple pour un CSS :

    !html+django
    {# base.html #}
    {% load static %}
    <html>
      <head>
        <link href="{% static "my_app/css/styles.css" %}" rel="stylesheet"/>

--------------------------------------------------------------------------------

## Servir les fichiers statiques

### En développement

En cours de développement (``DEBUG = True``), le serveur *standalone* de Django
se charge de servir les fichiers statiques lui-même via une vue dédiée : 
``django.contrib.staticfiles.views.serve``.

Cette méthode est peu efficace et peu sécurisée, et ne doit pas être utilisée
en production.

### En production

Le paramètre ``STATIC_ROOT`` permet de spécifier le chemin vers
le répertoire des fichiers statiques sur le système de fichiers.

Grâce à ce réglage, la commande ``collectstatic`` copie tous les fichiers statiques
vers le chemin précisé.

Il faut ensuite paramétrer le serveur web pour qu'il serve lui-même ces fichiers.

--------------------------------------------------------------------------------

# Tutoriel : Mettre en place une feuille de styles simple

.fx: alternate

--------------------------------------------------------------------------------

# Gérer les fichiers media

Les fichiers dits *media* sont les fichiers uploadés par les utilisateurs.

Comme pour les statiques, deux réglages principaux sont à connaître :

* ``MEDIA_URL`` : URL à laquelle il faut mettre à disposition les fichiers media
* ``MEDIA_ROOT`` : chemin vers lequel les fichiers media doivent être stockés

De manière interne, la gestion des fichiers est faite via le module ``django.core.files``
qui apporte notamment une classe ``File`` et des sous-classes comme ``ImageFile`` 
disposant de propriétés (``name``, ``size``, ...) et de méthodes très utiles
(``open()``, ``read()``, ``save()``).

--------------------------------------------------------------------------------

## Utiliser les fichiers media dans les modèles

Deux champs ``FileField`` et ``ImageField`` sont fournis pour pouvoir associer
facilement un fichier ou une image à une instance de modèle.

    !python
    # models.py
    class Book(models.Model):
        # ...
        summary = models.FileField(upload_to='summaries')

    class Author(models.Model):
        # ...
        photo = models.ImageField(upload_to='avatars')

Les instance de ```Book`` pourront donc chacun avoir un fichier attaché :

    !python
    >>> book = Book.objects.get(pk=12)
    >>> book.summary
    <FieldFile: summaries/summary_12.pdf>
    >>> book.summary.url
    u'http://my_site.com/media/summaries/summary_12.pdf'


--------------------------------------------------------------------------------

## Utiliser les fichiers media dans les formulaires

Il existe des champs de formulaires correspondant aux champs de modèles vus précédemment.
Il  est donc très facile d'obtenir un champ d'upload dans un formulaire.

    !python
    # forms.py
    class MyForm(forms.Form):
        # ...
        my_file = forms.FileField()


L'utilisation de ce type de formulaire implique quelques spécificités.
Dans la vue, l'objet ``request.FILES`` doit être fourni à l'initialisation du
formulaire :

    !python
    # views.py
    def my_view(request):
        # ...
        form = MyForm(request.POST, request.FILES)
        # ...

Dans la template, il faut préciser l'attribut ``enctype`` du ``<form>`` :

    !python
    {# my_app/templates/my_app/my_form_template.html #}
    <form action="" enctype="multipart/form-data" method="POST">
     ...
    </form>

-------------------------------------------------------------------------------

# Internationalisation et localisation

--------------------------------------------------------------------------------

## i18n et l10n : quelques réglages

Plusieurs réglages dans ``settings.py`` permettent d'activer ou non certaines
fonctionnalités liées à l'internationalisation et la localisation :

* ``USE_I18N`` : active ou non le module de traduction
* ``USE_L10N`` : active ou non l'affichage des dates et des nombres selon la langue
* ``USE_TZ`` : active ou non la gestion des fuseaux horaires
* ``LANGUAGE_CODE`` : langue par défaut
* ``LANGUAGES`` : liste des langues connues par l'application
* ``LOCALE_PATHS`` : chemins vers les fichiers de traduction


--------------------------------------------------------------------------------

## Traduire l'interface (fichiers python)

### Utilisation de la bibliothèque ``gettext``

Pour traduire les différents textes de l'interface, on utilise les fonctions ``ugetttext``,
ou plus souvent ``ugettext_lazy``. Pour la simplicité de l'écriture, on importe
généralement cette fonction avec l'alias '_'.

    !python
    from django.utils.translation import ugettext_lazy as _

### Exemple d'utilisation dans un modèle

    !python
    # models.py
    from django.utils.translation import ugettext_lazy as _

    class Book(models.Model):
        name = models.CharField(max_length=100,
                                verbose_name=_('Title'))

        class Meta:
            db_table = 'task'
            verbose_name = _('Book')
            verbose_name_plural = _('Books')

--------------------------------------------------------------------------------

## Traduire l'interface (fichiers python)

Exemple d'utilisation dans un formulaire

    !python
    # forms.py
    from django.utils.translation import ugettext_lazy as _

    class BookSearchForm(models.Model):
        search_text = models.CharField(label=_('Search text'))

Exemple d'utilisation dans une vue

    !python
    # views.py
    from django.utils.translation import ugettext_lazy as _
    
    def confirmation(request, result):
        if result == 'OK':
            confirmation = _('Verification succeeded')
        else:
            confirmation = _('Verification failed')
        # Render form
        return render(request, 'confirmation.html', {
            'confirmation': confirmation,
        })

--------------------------------------------------------------------------------

## Traduire l'interface (templates)

Deux tags permettant de traduire l'interface directement dans les templates
sont disponibles :

### Le tag {% trans %}

Il permet de traduire une chaine de caractères simple ou le contenu d'une variable.

    !html+django
    {% load i18n %}
    <title>{% trans "List of books" %}</title>
    <title>{% trans page_title %}</title>

### Le tag {% blocktrans %}

Il permet de mixer chaînes de caractères et variables pour traduire des chaînes complexes.

    !html+django
    {% load i18n %}
    {% blocktrans with book_t=book|title author_t=author|title %}
    <p>This is {{ book_t }} by {{ author_t }}</p>
    {% endblocktrans %}

--------------------------------------------------------------------------------

## Gérer les fichiers de traduction

### Créer / mettre à jour le fichier de traduction

La commande ``makemessages`` permet de créer le fichier traduction pour une langue
donnée (fichier texte avec l'extension ".po" contenant les identifiants de messages
et les traductions correspondantes). Cette commande doit être lancée depuis la racine
de l'application ou du projet pour lequel on crée le fichier car elle génère une
arborescence de dossiers ``locale/LANG/LC_MESSAGES``.

    !console
     $ django-admin.py makemessages -l fr


### Compiler le fichier de traduction

La commande ``compilemessages`` permet de compiler le fichier de traduction
pour qu'il soit utilisable dans le code Python.

    !console
     $ django-admin.py compilemessages -l fr

--------------------------------------------------------------------------------

# Scripting

--------------------------------------------------------------------------------

## ``django-admin.py`` et ``manage.py``

Les commandes ``django-admin.py`` sont très utilisées pour l'administration d'un 
pojet Django (création d'une application, synchronisation de la base, compilation
des fichiers de traduction, ...).

Le point d'entrée ``./manage.py`` se greffe autour de ``django-admin.py`` et 
s'exécute dans le contexte du projet (chargement des ``settings``, ajout du projet
dans ``sys.path``).

Lancer le script sans argument permet de lister les commandes disponibles :

    !console
    $ ./manage.py
    
    [django]
        check
        cleanup
        compilemessages
        createcachetable
        ...

Note : Il faut que l'environnement virtualisé soit démarré pour que Django
soit chargé et les différents modules soient chargés.

--------------------------------------------------------------------------------

## Écrire une commande d'administration

Écrire une commande *standalone* peut être très utile dans le cadre de tâches
d'administration qui peuvent être lancées périodiquement et automatiquement (cron).

### Arborescence des fichiers

Les commandes doivent être des fichiers Python placés dans un module 
``management/command`` de l'application.


    !console
    ├── my_app
    │   ├── __init__.py
    │   ├── admin.py
    │   ├── models.py
    │   ├── management
    │   │   ├── __init__.py
    │   │   ├── commands
    │   │   │   ├── __init__.py
    │   │   │   ├── my_test_command.py

--------------------------------------------------------------------------------

## Structure d'une commande d'administration

Pour créer une commande personnalisée, il faut écrire une classe ``Command`` qui hérite
de ``django.core.management.base.BaseCommand``.

    !python
    # my_app/management/command/my_test_command.py
    from django.core.management.base import BaseCommand, CommandError


    class Command(BaseCommand):
        args = '...'
        help = 'Do specific administration stuff'


        def handle(self, *args, **options):

            self.stdout.write('Command started')
            # Traitements
            self.stdout.write('Command ended')

--------------------------------------------------------------------------------

## Éxécuter une commande d'administration

### Éxécution manuelle

Il est bien sûr possible de lancer une commande personnalisée à la main, tout
simplement en utilisant directement dans le terminal le point d'entrée ``./manage.py`` :

    !console
    $ ./manage.py my_test_command

### Éxécution automatique

Il peut être aussi très utile de l'automatiser via une tâche cron :

    !python
    # Cron tasks
    0 * * * * /project_path/manage.py my_test_command

### Exécution depuis du code

    !python
    from django.core import management
    management.call_command("my_test_command")

--------------------------------------------------------------------------------

# Tutoriel : Écrire une commande qui supprime automatiquement les tâches réalisées avec un deadline qui remonte à plus d'une semaine

.fx: alternate

--------------------------------------------------------------------------------

# Personnaliser l'interface d'administration

--------------------------------------------------------------------------------

## Fonctionnement de l'interface d'admin

Pour activer l'interface d'administration il faut commencer par :

* inclure le module et ses dépendances dans les ``INSTALLED_APPS`` ;
* instancier un objet ``AdminSite`` et connecter une URL vers cette page ;
* déclarer les modèles que l'on souhaite voir apparaître dans l'interface d'administration dans les fichiers ``admin.py`` des applications.

## La classe ``ModelAdmin``

La classe est la représentation d'un modèle dans l'interface d'administration.

--------------------------------------------------------------------------------

## Fonctionnement de l'interface d'admin

### Déclaration d'un ``ModelAdmin``

Si on ne souhaite pas personnaliser la représentation du modèle dans l'interface
d'administration, il existe une version simplifiée de déclaration

    !python
    # admin.py
    from django.contrib import admin
    from myproject.myapp.models import Book

    admin.site.register(Book)

Il est cependant possible de surcharger le ``ModelAdmin`` d'un modèle afin de
personnaliser son comportement :

    !python
    # admin.py
    from django.contrib import admin
    from myproject.myapp.models import Book

    class BookAdmin(admin.ModelAdmin):
        # Personnalisations

    admin.site.register(Book, BookAdmin)


--------------------------------------------------------------------------------

## Personnaliser les listes

### Personnaliser l'affichage

Les propriétés ``list_display`` et ``list_display_links`` permettent respectivement
de spécifier les colonnes que l'on souhaite voir apparaitre dans la liste et de préciser
lesquelles d'entre elles doivent être cliquables.

### Personnaliser le filtrage

L'attribut ``list_filter`` permet de mettre en place une recherche type recherche
à facettes dans une barre latérale à droite.

Si le modèle à une propriété de type ``Date`` ou ``Datetime``, la propriété ``date_hierarchy`` peut être précisée pour créer un index par date.

## Personnaliser la recherche

L'attribut ``search_fields`` permet de lister les champs sur lesquels la recherche 
doit être exécutée.

--------------------------------------------------------------------------------

## Personnaliser les listes - exemple combiné

    !python
    # admin.py
    from django.contrib import admin
    from myproject.myapp.models import Book

    class BookAdmin(admin.ModelAdmin):
        list_display = ['title', 'release']
        list_display_links = ['title']
        list_filter = ['author']
        date_hierarchy = 'release'
        search_fields = ['title', 'author__name']

    admin.site.register(Book, BookAdmin)


--------------------------------------------------------------------------------

## Personnaliser les formulaires

### Personnaliser les champs

Grâce aux propriétés ``fields`` ou ``exclude``, il est possible de spécifier
quels champs on souhaite voir apparaître dans les formulaires de l'interface
d'administration

La propriété ``fieldsets`` permet d'aller plus loin et d'organiser la mise en 
page du formulaire.

### Surcharger le formulaire

Il est possible d'aller encore plus loin en surchargeant la template d'un formulaire
ou même d'écrire son propre formulaire et de le déclarer dans le ``ModelAdmin``

--------------------------------------------------------------------------------

## Personnaliser les formulaires - Exemple

    !python
    # admin.py
    from django.contrib import admin
    from django import forms
    from myproject.myapp.models import Book

    class BookAdminForm(forms.ModelForm):
        # ...
    
    class BookAdmin(admin.ModelAdmin):
        list_display = ['title', 'release']
        list_display_links = ['title']
        list_filter = ['author']
        date_hierarchy = 'release'
        search_fields = ['title', 'author__name']

        form = BookAdminForm

    admin.site.register(Book, BookAdmin)


--------------------------------------------------------------------------------

# Tutoriel : Personnaliser légèrement l'interface d'administration

.fx: alternate

# Presenter notes

Ici on attendra une personnalisation qui s'approche du front-end

---

# Deploiement

---

## Serveurs WSGI

WSGI : interface entre un serveur web et une application web en Python

* Gunicorn
* mod_wsgi (fonctionne avec Apache HTTP Server)
* uWSGI
* Chaussette

Application WSGI minimale :

    !python
    def application(environ, start_response):
        data = b'Hello, World!\n'
        start_response('200 OK', [
            ('Content-type', 'text/plain'),
            ('Content-Length', str(len(data)))
        ])
        return iter([data])

---

## Serveur web 

* nginx : léger, rapide, recommandé
* Apache HTTP Server : très complet et nombreux modules

<!-- -->

    !nginx
    upstream wsgi_server {
      server unix:/webapps/django/run/gunicorn.sock fail_timeout=0;
    }

    server {
        listen   80;
        server_name example.com;
        location /static/ { alias /webapps/hello_django/static/; }
        location /media/ { alias /webapps/hello_django/media/; }
        location / {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $http_host;
            proxy_redirect off;
            if (!-f $request_filename) {
                proxy_pass http://wsgi_server;
                break;
            }
        }
    }


--------------------------------------------------------------------------------

# Merci !
