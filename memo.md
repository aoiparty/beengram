python manage.py migrate
migrate...データベース
外部からデータベースを登録

1
User
username
email
profile
follow
icon
like

Post
user
img
note
post_date

2
url/home > PostListViewを参照

``` py
class PostListView(LoginRequiredMixin, ListView):
    model = Post
    ordering = ("-post_date",)
    paginate_by = 20

    def get_queryset(self):
        queryset = super().get_queryset().select_related("user")
        if "follow" in self.request.GET:
            queryset = queryset.filter(
                Q(user=self.request.user)
                | Q(user__in=self.request.user.follow.all())
            )
            自分が投稿したもの、もしくは自分がフォローしているユーザの投稿
        return queryset
```

3
デフォルト＞object or 小文字（post）

``` py
class PostDetailView(LoginRequiredMixin, DetailView):
    model = Post

    def get_queryset(self):
        return (
            super()
            .get_queryset()
            .select_related("user")
        )
```

４
フォローボタン　＞　いろいろなところにある

``` py
class FollowMixin:
    def post(self, request, *args, **kwargs):
        #def post: POST処理をしたとき
        target = get_object_or_404(User, pk=self.request.POST.get("target", 0))
        if "follow" in self.request.POST:
            self.request.user.follow.add(target)
        elif "unfollow" in self.request.POST:
            self.request.user.follow.remove(target)
        return HttpResponseRedirect(self.request.META.get("HTTP_REFERER"))

# 「FollowMixin」継承している　　　　　　　↓
class ProfileView(LoginRequiredMixin, FollowMixin, DetailView):
    ...
```

5

フォロー＞http://127.0.0.1:8000/profile/follow-list/10
フォロワー＞http://127.0.0.1:8000/profile/follow-list/10?followed

urlが同じ

``` py
class FollowListView(LoginRequiredMixin, FollowMixin, ListView):
...
      def get_queryset(self):
        ...
        if "followed" in self.request.GET:
        #followedがGET処理（url）のなかにある時（？folloedが含まれているか）
            queryset = user.followed
        else:
            queryset = user.follow
        follow_list = self.request.user.follow.all().values_list(
            "id", flat=True
        )
        self.queryset = queryset.annotate(
            is_follow=Case(
                When(id__in=follow_list, then=True),
                default=False,
            )
        )
        return super().get_queryset()

```

6
search?~~
で分岐

７

``` py
def _search_users(self, keyword):
        follow_list = self.request.user.follow.all().values_list(
            "id", flat=True
        )
       # フォローリストにidがある時True
        # ない時False
        queryset = User.objects.all().annotate(
            is_follow=Case(
                When(id__in=follow_list, then=True),
                default=False,
            )
        )
```
if文どこ

８
``` py

class SearchView(LoginRequiredMixin, FollowMixin, ListView):
    template_name = "main/search.html"
    paginate_by = 20

    def get_queryset(self):
        form = SearchForm(self.request.GET)
        if form.is_valid():
            keyword = form.cleaned_data["keyword"]
            if "post" in self.request.GET:
                queryset = self._search_posts(keyword)
            else:
                queryset = self._search_users(keyword)
        else:
            if "post" in self.request.GET:
                queryset = Post.objects.none()
            else:
                queryset = User.objects.none()
        return queryset

    def _search_posts(self, keyword):
        queryset = Post.objects.all().select_related("user")

        for word in keyword.split():
            queryset = queryset.filter(note__icontains=word)
            #kensaku
        return queryset.order_by("-post_date")

    def _search_users(self, keyword):
        follow_list = self.request.user.follow.all().values_list(
            "id", flat=True
        )
        queryset = User.objects.all().annotate(
            is_follow=Case(
                When(id__in=follow_list, then=True),
                default=False,
            )
        )

        for word in keyword.split():
            queryset = queryset.filter(username__icontains=word)
            #検索
        return queryset.order_by("-is_follow", "username")

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        if "keyword" in self.request.GET:
            context["form"] = SearchForm(self.request.GET)
        else:
            context["form"] = SearchForm()
        return context
```

9
``` py
path(
        "settings/",
        TemplateView.as_view(template_name="main/settings.html"),
        name="settings",
    ),
    #これだけで完結

```

10
``` html
{% block footer %}{% include "main/footer.html" with current="profile" %}{% endblock %}

<!-- {% include "ファイル名" %} で差し込める-->