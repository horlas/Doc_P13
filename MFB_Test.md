# Tests

* Couverture de 91 %
* Utilisation du Client de Test 
* Utilisation de Factory Request 

## exemple de test sur la vue AnswerAnnouncement

Cette vue gère le post de la réponse à une annonce . Cette réponse ne doit pas dépasser 200 caractères, elle doit lier la réponse à l'annonce, elle doit s'assurer que l'utilisateur ne puisse pas répondre à sa propre annonce et elle doit créer le modèle MusicianAnswerAnnouncement en instanciant tous les champs proprement 

```
response = MusicianAnswerAnnouncement(
                    content = content,
                    created_at=timezone.now(),
                    author=self.request.user,
                    musician_announcement=a,
                    recipient=a.author
                     )
```

Nous la testons en utilisation Factory Request (test d'une vue comme une fonction)

On commence par créer une classe personnalisée

```
class MyTestCase(TestCase):
    '''Here is a parent class with custom global setup'''
    def setUp(self):
        # web client
        self.client = Client()
        # our test user
        self.email = 'tata@gmail.com'
        self.password = 'aqwz7418'
        self.test_user = User.objects.create_user(self.email, self.password)

        # his profil to test get info
        self.userprofile = UserProfile.objects.get(user=self.test_user)
        self.userprofile.username = 'Super Tatie'
        self.userprofile.bio = 'Elle passe ses nuits sans dormir'\
                                'À gacher son bel avenir'\
                                'La groupie du pianiste'
        self.userprofile.birth_year = 1948
        self.userprofile.town = 'Montpellier'
        self.userprofile.county_name = 'Hérault'
        self.userprofile.gender = 'F'
        self.userprofile.save()

        # her instruments
        self.instrument = Instrument.objects.create(instrument='Contrebassiste',
                                                    level='Intermediaire',
                                                    musician=self.test_user)

        # we need to create some announcement
        self.a1 = MusicianAnnouncement.objects.create(title='title1', content='Coucou Toi!', author=self.test_user)
        self.a2 = MusicianAnnouncement.objects.create(title='title2', content='content', author=self.test_user)
        self.a3 = MusicianAnnouncement.objects.create(title='title3', content='content', author=self.test_user)

        # log user
        self.login = self.client.login(username=self.email, password=self.password)

        # Factory
        self.factory = RequestFactory()

        # add a second members to test some features
        self.email2 = 'paul@gmail.com'
        self.password2 = 'aqwz7418'
        self.test_user2 = User.objects.create_user(self.email2, self.password2)
```        

Test de la vue

```
class AnswerAnnouncementTest(MyTestCase):

    def setUp(self):
        super(AnswerAnnouncementTest, self).setUp()
        self.url = reverse('announcement:post_answer')

    def test_view_post_invalid_1(self):
        ''' User connected can not response to his own announcement'''
        # after count response
        data = {'answer_text' : 'Salut, ça roule ?',
                'a_id' : self.a1.id }
        request = self.factory.post(self.url, data)
        request.user = self.test_user

        # adding session
        middleware = SessionMiddleware()
        middleware.process_request(request)
        request.session.save()

        # adding messages
        messages = FallbackStorage(request)
        setattr(request, '_messages', messages)

        # test the view
        response = views.AnswerAnnouncement.as_view()(request)
        # test the success message
        for m in messages:
            message = str(m)
        self.assertEqual(message, "Vous ne pouvez pas répondre à votre propre annonce")
        # # redirect to list_announcement
        self.assertEqual(response.status_code, 302)
        redirect_url = '/announcement/detail_post/{}'.format(self.a1.id)
        self.assertEqual(response['location'], redirect_url)

        # no response is posted
        message = MusicianAnswerAnnouncement.objects.count()
        self.assertEqual(message, 0)

    def test_view_post_invalid_2(self):
        ''' User connected can not response to his own announcement'''
        data = {'answer_text': 'xccccccccccccccccccccccccccccccc'
                               'cccccccccccccccccccccccccccccccc'
                               'cccccccccccccccccccccccccccccccccc'
                               'ccccccccccccccccccccccccccccccccccccc'
                               'cccccccccccccccccccccccccccccccccccccccc'
                               'cccccccccccccccccccccccccccccccccccccccccccccccc',
                'a_id' : self.a1.id }
        request = self.factory.post(self.url, data)
        request.user = self.test_user2

        # adding session
        middleware = SessionMiddleware()
        middleware.process_request(request)
        request.session.save()

        # adding messages
        messages = FallbackStorage(request)
        setattr(request, '_messages', messages)

        # test the view
        response = views.AnswerAnnouncement.as_view()(request)
        # test the success message
        for m in messages:
            message = str(m)
        self.assertEqual(message, "Réponse trop longue")
        # redirect to list_announcement
        self.assertEqual(response.status_code, 302)
        redirect_url = '/announcement/detail_post/{}'.format(self.a1.id)
        self.assertEqual(response['location'], redirect_url)

        # no response is posted
        message = MusicianAnswerAnnouncement.objects.count()
        self.assertEqual(message, 0)

    def test_view_post_valid(self):
        ''' User connected can not response to his own announcement'''
        data = {'answer_text': 'Salut! Ton annonce est chouette',
                'a_id' : self.a1.id }
        request = self.factory.post(self.url, data)
        request.user = self.test_user2
        # adding session
        middleware = SessionMiddleware()
        middleware.process_request(request)
        request.session.save()

        # adding messages
        messages = FallbackStorage(request)
        setattr(request, '_messages', messages)

        # test the view
        response = views.AnswerAnnouncement.as_view()(request)
        # print(response.templates)
        # test the success message
        for m in messages:
            message = str(m)
        self.assertEqual(message, 'Votre réponse est envoyée vous pouvez la retrouver dans Mes messages!')
        # # redirect to list_announcement
        self.assertEqual(response.status_code, 302)
        redirect_url = '/announcement/detail_post/{}'.format(self.a1.id)
        self.assertEqual(response['location'], redirect_url)

        # no response is posted
        message = MusicianAnswerAnnouncement.objects.count()
        self.assertEqual(message, 1)

        # test if the announcement foreign_key is correct
        message= MusicianAnswerAnnouncement.objects.get(content='Salut! Ton annonce est chouette')
        self.assertEqual(message.musician_announcement, self.a1)
        # test if the recipient is correct
        self.assertEqual(message.recipient, self.test_user)
        
```        


## exemple de test d'une vue retournant un fichier Json

Le cas particulier d'une vue en appel Ajax

```
class TestSearchView(MyTestCase):

    def setUp(self):
        super(TestSearchView, self).setUp()
        self.url = reverse('core:search')
        # specially for ajax view
        self.kwargs = {'HTTP_X_REQUESTED_WITH': 'XMLHttpRequest'}
        # create some context:
        self.a1 = MusicianAnnouncement.objects.create(title="Coucou", content="Coucou Toi!", author=self.test_user, town="Sete", county_name="Gard")
        self.a2 = MusicianAnnouncement.objects.create(title="title2", content="content", author=self.test_user, town="Sete", county_name="Gard")
        self.a3 = MusicianAnnouncement.objects.create(title="title3", content="content", author=self.test_user, town="Sete", county_name="Gard")

        first_created_at = self.a1.created_at.strftime("%d %B %Y")
        second_created_at = self.a2.created_at.strftime("%d %B %Y")
        third_created_at = self.a3.created_at.strftime("%d %B %Y")

        self.layout_response = [
            {"tag": "annonces", "title": self.a1.title, "created_at": first_created_at, "county_name": self.a1.county_name, "id": self.a1.id,
             "town": self.a1.town },
            {"tag": "annonces", "title": self.a2.title, "created_at": second_created_at,  "county_name": self.a2.county_name, "id": self.a2.id,
             "town": self.a2.town},
            {"tag": "annonces", "title": self.a3.title, "created_at": third_created_at, "county_name": self.a3.county_name,
            "id": self.a3.id, "town": self.a3.town}]


    def test_annonces_without_cp(self):
        ''' case test query 'Annonces' without cp, the view return all announcements '''
        data = {'item': 'Annonces', 'cp': ''}
        request = self.factory.post(self.url, data, **self.kwargs)
        response = search(request)
        self.assertEqual(response.status_code, 200)
        response_content = str(response.content, encoding='utf8')
        self.assertJSONEqual(response_content, self.layout_response)

    def test_annonces(self):
        ''' case test query 'Annonces' with cp, the view return all announcements of county code '''
        data = {'item': 'Annonces', 'cp': '30'}
        request = self.factory.post(self.url, data, **self.kwargs)
        response = search(request)
        self.assertEqual(response.status_code, 200)
        response_content = str(response.content, encoding='utf8')
        self.assertJSONEqual(response_content, self.layout_response)

    def test_musicians(self):
        ''' test case with 'Musiciens' and cp code'''
        data = {'item': 'Musiciens', 'cp': '34'}
        request = self.factory.post(self.url, data, **self.kwargs)
        response = search(request)
        self.assertEqual(response.status_code, 200)
        response_content = str(response.content, encoding='utf8')
        # print(response_content)
        layout_response = [{"town": "Montpellier",
                            "instrument": {str(self.instrument.id): "Contrebassiste"},
                            "pk": self.test_user.pk, "tag": "musicians",
                            "avatar":  self.userprofile.avatar.url,
                            "name": "Super Tatie", "county_name": "Herault"}]
        self.assertJSONEqual(response_content, layout_response)

        # without avatar image

    def test_band(self):
        ''' test case with 'Groupes' without cp code without avatar image'''
        data = {'item': 'Groupes', 'cp': ''}
        request = self.factory.post(self.url, data, **self.kwargs)
        response = search(request)
        self.assertEqual(response.status_code, 200)
        response_content = str(response.content, encoding='utf8')
        layout_response = [{"bio": self.band_test.bio,
                            "type": "Groupe de Compos",
                            "name": self.band_test.name,
                            "tag": "groupes",
                            "avatar": "static/core/img/0_band.jpg",
                            "slug": self.band_test.slug,
                            "musical_genre": self.band_test.musical_genre,
                            "town": "Londres",
                            "county_name": "Angleterre"}]
        self.assertJSONEqual(response_content, layout_response)

    def test_view_fail(self):
        ''' in the impossible case where the view will return a 'fail
        we mock this without get the XMLHttpRequest '''
        data = {'item': 'Groupes', 'cp': ''}
        request = self.factory.post(self.url, data)
        response = search(request)
        self.assertEqual(response.status_code, 200)
        response_content = str(response.content, encoding='utf8')
        self.assertEqual(response_content, '"fail"')
        
```        
                   