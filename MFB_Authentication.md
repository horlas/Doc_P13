# Authentification sur My First Band

## Ré-écriture de l'authentification Django pour permettre de la connexion avec l'email et un password (au lieu d'un username)
Pour cela il faut réécrire le gestionnaire du model User 

```
class UserManager(BaseUserManager):
    """Define a model manager for User model with no username field."""

    use_in_migrations = True

    def _create_user(self, email, password, **extra_fields):
        """Create and save a User with the given email and password."""
        if not email:
            raise ValueError('The given email must be set')
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_user(self, email , password=None , **extra_fields):
        """Create and save a regular User with the given email and password."""
        extra_fields.setdefault('is_staff', False)
        extra_fields.setdefault('is_superuser', False)
        return self._create_user(email, password, **extra_fields)

    def create_superuser(self, email, password, **extra_fields):
        """Create and save a SuperUser with the given email and password."""
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)

        if extra_fields.get('is_staff') is not True:
            raise ValueError('Superuser must have is_staff=True.')
        if extra_fields.get('is_superuser') is not True:
            raise ValueError('Superuser must have is_superuser=True.')

        return self._create_user(email, password, **extra_fields)
         
```
    
et lier le modèle User

```
class User(AbstractUser):
    '''User model with email instead username'''
    username = None
    email = models.EmailField(_("Adresse email"), unique=True)
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = []

    objects = UserManager()  ##  This is the new line in the User model. ##

```

Et dans le settings.py ne pas oublier d'indiquer que le modèle pour l'authentification qui est utilisé pour l'application est celui customisé:
```
AUTH_USER_MODEL = 'authentication.User'
```

[Lien](https://docs.djangoproject.com/fr/2.1/topics/auth/customizing/#a-full-example) vers la documentation Django 

## Création automatique d'un Useprofile à chaque création d'un modèle User et lier les deux

Application : Musicians , Model : UserProfile
Nous avons utiliser les signaux permettant de générer une nouvelle instance de modèle lors de la création d'une instance liée

```
class UserProfile(models.Model):
    '''
    Custom User Profile
    '''
(...)

    user = models.OneToOneField(User, primary_key=True,  on_delete=models.CASCADE)
    username = models.CharField("Nom", max_length=60)
    
(...)    
 
    
    # Here we instantiate a Fieltracker to track any fields specially avatar field
    tracker = FieldTracker()

    # creating the profile when creating the user
    @receiver(post_save, sender=User)
    def create_user_profile(sender, instance, created, **kwargs):
        if created:
            UserProfile.objects.create(user=instance)

    @receiver(post_save, sender=User)
    def save_user_profile(sender, instance, **kwargs):
        instance.userprofile.save()

```

## Authentification par Google
Création d'une clé d'API Google + et d'une clé secrète (variables d'environnement).
Ré-écriture de Pipeline pour avoir une configuation personalisée (par exemple on avait pas besoin des informations complémentaires du compte Gmail ( First et Last name)), pour lier aussi l'utilisateur à sa connexion par gmail.

```
# Social_authentication

AUTHENTICATION_BACKENDS = (
    'social_core.backends.google.GoogleOAuth2',
    'django.contrib.auth.backends.ModelBackend',
)

# When using PostgreSQL, it’s recommended to use the built-in JSONB field to store the extracted extra_data
SOCIAL_AUTH_POSTGRES_JSONFIELD = True

# In case you need a custom namespace, this setting is also needed:
SOCIAL_AUTH_URL_NAMESPACE = 'social'

# Api Google authentication
SOCIAL_AUTH_GOOGLE_OAUTH2_KEY = os.environ.get('GOOGLE_KEY')
SOCIAL_AUTH_GOOGLE_OAUTH2_SECRET = os.environ.get('GOOGLE_SECRET')

SOCIAL_AUTH_ADMIN_USER_SEARCH_FIELDS = ['email', 'password']

LOGIN_REDIRECT_URL = '/'

# custom functions to create user instance on social auth
SOCIAL_AUTH_PIPELINE = (
    # Get the information we can about the user and return it in a simple
    # format to create the user instance later. On some cases the details are
    # already part of the auth response from the provider, but sometimes this
    # could hit a provider API.
    'social_core.pipeline.social_auth.social_details',
    # Get the social uid from whichever service we're authing thru. The uid is
    # the unique identifier of the given user in the provider.
    'social_core.pipeline.social_auth.social_uid',
    # Verifies that the current auth process is valid within the current
    # project, this is where emails and domains whitelists are applied (if
    # defined).
    'social_core.pipeline.social_auth.auth_allowed',
    # Checks if the current social-account is already associated in the site.
    'social_core.pipeline.social_auth.social_user',
    # Verifies that the current auth process is valid within the current
    # project, this is where emails and domains whitelists are applied (if
    # defined).
    'social_core.pipeline.social_auth.auth_allowed',
    # Checks if the current social-account is already associated in the site.
    'social_core.pipeline.social_auth.social_user',
    # Associates the current social details with another user account with
    # a similar email address. Disabled by default.
    'social_core.pipeline.social_auth.associate_by_email',
    # Create a user account if we haven't found one yet.
    'social_core.pipeline.user.create_user',
    # Create the record that associates the social account with the user.
    'social_core.pipeline.social_auth.associate_user',
```    

Ce que ça donne sur l'administration Django lorsque un utilisateur s'est connecté:
[admin](http://127.0.0.1:8000/admin/social_django/usersocialauth/)



## Ré-initialisation du mot de pass par mail

On s'est appuyé sur le serveur smtp de Gmail 

settings.py

```
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_USE_TLS = True
EMAIL_PORT = 587
EMAIL_HOST_USER = os.environ.get('SECRET_ACCOUNT')
EMAIL_HOST_PASSWORD = os.environ.get('SECRET_MDP')
DEFAULT_FROM_EMAIL = 'MyFirstBand Team <noreply@example.com>'
```

Écriture des urls en important les vues de django.contrib.auth 

```
app_name = 'authentication'

urlpatterns = [
    path('signup/', views.SignupCustomView.as_view(), name='signup'),
    path('accounts/login/', auth_views.LoginView.as_view(template_name='authentication/login.html',
                                                          authentication_form=CustomLoginForm), name='login'),
    path('accounts/password_reset/', auth_views.PasswordResetView.as_view(
                                                    template_name='authentication/password_reset_form.html',
                                                    email_template_name='authentication/reset_password_email.html',
                                                    subject_template_name='authentication/reset_password_subject.txt',
                                                    success_url= reverse_lazy('authentication:reset_password_done'),
                                                    ) ,name='password_reset'),
    path('accounts/password_reset_done/', auth_views.PasswordResetDoneView.as_view(
                                            template_name='authentication/password_reset_done.html'), name='reset_password_done'),
    path('accounts/password_reset_confirm/<uidb64>/<token>/ ', auth_views.PasswordResetConfirmView.as_view(
                                                            template_name='authentication/password_reset_confirm.html',
                                                            success_url= reverse_lazy('authentication:reset_password_complete'),
                                                            post_reset_login_backend='django.contrib.auth.backends.ModelBackend',
                                                            post_reset_login=True ), name='reset_password_confirm'),
    path('accounts/password_reset_complete/', auth_views.PasswordResetCompleteView.as_view(template_name='authentication/password_reset_complete.html'),
                                                                                            name='reset_password_complete'),
    path('accounts/', include('django.contrib.auth.urls')),
    # path('accounts/', include('social_django.urls', namespace='social'))

]
```

et ajout des templates necessaires

Toutes ces vues additionnelles sont testées , même l'envoi de mail 

```
    def test_reset_password_post(self):
        ''' we test the continuity of the initialization of the password'''
        email_data = {
            "email": self.user.email
        }
        response = self.client.post(self.url, email_data, follow=True)
        self.assertRedirects(
            response,
            expected_url= reverse('authentication:reset_password_done'),
            status_code=302,
            target_status_code=200
        )
        self.assertContains(response,
                            "Nous vous avons envoyé un e-mail avec les instructions pour")
        # check if the mail is well sent
        self.assertEqual(len(mail.outbox), 1)
        # grab uid and token
        msg = mail.outbox[0]
        queries = re.findall('/([\w\-]+)', msg.body)
        token = queries[5]
        uid = queries[4]
        # get the page witch user can post his new password
        url_reset_confirm = reverse('authentication:reset_password_confirm', kwargs={'uidb64': uid, 'token': token})
        response_confirm = self.client.get(url_reset_confirm, follow=True)
        # check the status code and the content
        self.assertEqual(response_confirm.status_code, 200)
        list_template = [t.name for t in response.templates]
        assert 'authentication/password_reset_done.html' in list_template
        # post new password
        data = {
            "new_password1": "1234aqwz",
            "new_password2": "1234aqwz"
        }
        url_reset_password_post = reverse('authentication:reset_password_confirm',
                                          kwargs={'uidb64': uid, 'token': "set-password"})
        response_post_password =  self.client.post(url_reset_password_post, data, follow=True)
        self.assertRedirects(response_post_password,
                             expected_url=reverse_lazy('authentication:reset_password_complete'),
                             status_code=302,
                             target_status_code=200
                             )
```                             

