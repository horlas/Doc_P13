# Class Based Views exemple

Nous prenons pour exemple la fonctionnalité Update du Profil Utilisateur
Dans cette vue nous renvoyons 5 formulaires en Get. Le post de ces cinq formulaires sont dissociés les uns des autres et font appel à leur vue particulière . En cas de succès, la redirection se fait sur la vue "parente" . Ce qui donne au niveau de l'utilisateur une impression de mise à jour immédiate (ajax) .

Vue affichant les cinq formulaires :

```
class UpdateProfilView(LoginRequiredMixin, TemplateView):
    ''' Update or set the datas witch concern the musician'''

    template_name = 'musicians/update_profile.html'

    def get(self, request, *args, **kwargs):

        avatar_form = AvatarForm(self.request.GET or None,
                                 instance=request.user.userprofile)
        profile_form = ProfileForm(self.request.GET or None,
                                   instance=request.user.userprofile)
        instru_form = InstruCreateForm(self.request.GET or None)
        del_instru_form = InstruDeleteForm(request.user)
        local_form = LocalForm(self.request.GET or None,
                               instance=request.user.userprofile)
        context = self.get_context_data(**kwargs)
        context['avatar_form'] = avatar_form
        context['profile_form'] = profile_form
        context['local_form'] = local_form
        context['instru_form'] = instru_form
        context['del_instru_form'] = del_instru_form
        return self.render_to_response(context)
```        
  
Extraits de deux des vues gérant le post du formulaire
 
```
 class UpdateAvatarView(FormView, SuccessMessageMixin):

    form_class = AvatarForm
    template_name = 'musicians/update_profile.html'

    @method_decorator(login_required)
    @transaction.atomic
    def post(self, request, *args, **kwargs):
        avatar_form = self.form_class(request.POST,
                                      request.FILES ,
                                      instance=request.user.userprofile)

        if avatar_form.is_valid():
            avatar_form.save()
            messages.success(self.request, (" Votre image a été  mise à jour!"))
            return redirect(reverse_lazy('musicians:update_profile', kwargs={'pk': self.request.user.id})) #, self.get_context_data(success=True))

        else:
            avatar_form = self.form_class(instance=request.user.userprofile)
            return self.render_to_response(self.get_context_data(avatar_form =avatar_form))
```

```
class InstruCreateView(LoginRequiredMixin, CreateView, SuccessMessageMixin):
    '''View to add instrument to a musician'''

    model = Instrument
    fields = ['instrument', 'level']
    template_name = 'musicians/update_profile.html'

    def get_success_url(self):
        return reverse_lazy('musicians:update_profile', kwargs={'pk': self.request.user.id})

    def form_valid(self, instru_form):
        instru_form.instance.musician = self.request.user
        messages.success(self.request, ("Votre Instrument a été ajouté ! "))
        return super().form_valid(instru_form)
```

# Vue Ajax

Nous avons utilisés dans le projet certaines vues qui retournent des données au format Json et qui sont traitées grâce à un script coder en Javascript (ajax).

Voici l'exemple de la vue qui retourne les données lorsque l'utilisateur fait une recherche sur le site . Ces données sont extraites de la base en fonction de la saisie utilisateur sur la base du choix : Annonces/Musiciens/Groupes et du code postal.

```
def search(request):
    ''' ajax return of datas witch depends of an user input'''

    if request.is_ajax():
        item = request.POST.get('item')
        cp = request.POST.get('cp')
        results = []
        if item == 'Annonces' and cp == '':
            # return all announcements
            ads = MusicianAnnouncement.objects.exclude(is_active=False) \
                                      .order_by('created_at').values('id',
                                               'title',
                                               'town',
                                               'county_name',
                                               'created_at')
            for a in ads:
                a['created_at'] = a['created_at'].strftime("%d %B %Y")
                a['tag'] = 'annonces'
                results.append(a)

        if item == 'Annonces' and cp != '':
            url = 'https://geo.api.gouv.fr/departements?code={}&fields=nom'.format(cp)
            response = requests.get(url)
            dept = response.json()[0]['nom']
            # query data base
            ads = MusicianAnnouncement.objects.filter(county_name=dept)\
                                              .exclude(is_active=False)\
                                              .order_by('created_at').values('id',
                                                                             'title',
                                                                             'town',
                                                                             'county_name',
                                                                             'created_at')

            for a in ads:
                a['created_at'] = a['created_at'].strftime("%d %B %Y")
                a['tag'] = 'annonces'
                results.append(a)

        if item == 'Musiciens':
            # query data : users who live in the same county, and witch have some data
            mus = User.objects.filter(userprofile__code__startswith=cp)\
                                     .exclude(userprofile__username='')\
                                     .order_by('-id')

            for m in mus:
                # get instrument
                instru_queryset = m.instrument_set.all()
                dict_instru = {i.id: i.instrument for i in instru_queryset}
                # for musicians who have no avatar display default imq
                try:
                    avatar = m.userprofile.avatar.url
                except ValueError:
                    avatar = 'static/core/img/0.jpg'
                dic = {"name": m.userprofile.username, 'avatar': avatar,
                       "pk" : m.userprofile.pk, 'town': m.userprofile.town,
                       "county_name": m.userprofile.county_name,
                        "instrument" : dict_instru, "tag": "musicians"}
                results.append(dic)

        if item == 'Groupes':          
            # query data : band
            bands = Band.objects.filter(code__startswith=cp) \
                                .exclude(name='') \
                                .order_by('-id')
            for b in bands:
                # for band witch have no avatar display default img
                try:
                    avatar = b.avatar.url
                except ValueError :
                    avatar = 'static/core/img/0_band.jpg'

                dic = {'name' : b.name, 'avatar' : avatar,
                       'town' : b.town, 'county_name' : b.county_name,
                       'type' : b.type, 'musical_genre' : b.musical_genre,
                       'bio' : b.bio , 'slug': b.slug, 'tag': 'groupes'}
                results.append(dic)

    else:

        results = 'fail'

    return JsonResponse(results, safe=False)
```    




   

