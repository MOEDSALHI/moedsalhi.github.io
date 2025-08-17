# Secure Django Backend Guide ![Python](https://img.shields.io/badge/python-3.12-blue) ![Django](https://img.shields.io/badge/django-5.0-green) ![DRF](https://img.shields.io/badge/DRF-3.15-red)

## 📑 Table des matières

- [Secure Django Backend Guide   ](#secure-django-backend-guide---)
  - [📑 Table des matières](#-table-des-matières)
  - [🚀 Introduction](#-introduction)
  - [⚙️ Installation](#️-installation)
  - [🔧 Configuration initiale](#-configuration-initiale)
  - [🗄️ Modèle de données](#️-modèle-de-données)
  - [🧩 Sérialisation](#-sérialisation)
  - [🔐 Autorisations](#-autorisations)
  - [📡 Vues et endpoints](#-vues-et-endpoints)
  - [🌐 Routage et JWT](#-routage-et-jwt)
  - [⚖️ Configuration avancée](#️-configuration-avancée)
  - [🧪 Tests automatisés](#-tests-automatisés)
  - [🚦 Bonnes pratiques](#-bonnes-pratiques)
  - [⚠️ Pièges courants](#️-pièges-courants)
  - [📊 Schéma JWT](#-schéma-jwt)
  - [📎 Références](#-références)

------------------------------------------------------------------------

## 🚀 Introduction

Ce projet est un **guide complet** pour construire une API Django +
Django REST Framework (DRF) **sécurisée** et **scalable**.\
Il inclut : - Authentification JWT avec rotation et blacklist. -
Permissions au niveau objet (anti-BOLA). - Pagination, recherche,
filtrage. - Sécurité de déploiement (CORS, HTTPS, cookies sécurisés,
HSTS). - Documentation OpenAPI avec DRF Spectacular. - Tests automatisés
avec `pytest`.

------------------------------------------------------------------------

## ⚙️ Installation

``` bash
pip install "django>=5.0" djangorestframework   djangorestframework-simplejwt[crypto]   drf-spectacular django-cors-headers
django-admin startproject blog_api
cd blog_api
python manage.py startapp posts
```

------------------------------------------------------------------------

## 🔧 Configuration initiale

Dans `settings.py` :

``` python
INSTALLED_APPS = [
    ...
    "rest_framework",
    "drf_spectacular",
    "corsheaders",
    "posts",
]
MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",
    ...
]
CORS_ALLOWED_ORIGINS = ["https://frontend.example.com"]
```

------------------------------------------------------------------------

## 🗄️ Modèle de données

``` python
class Post(models.Model):
    title = models.CharField(max_length=150, db_index=True)
    slug = models.SlugField(max_length=160, unique=True)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name="posts")
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        constraints = [
            models.UniqueConstraint(fields=["author","title"], name="uq_author_title")
        ]
        ordering = ["-created_at"]
```

------------------------------------------------------------------------

## 🧩 Sérialisation

``` python
class PostSerializer(serializers.ModelSerializer):
    author = serializers.StringRelatedField(read_only=True)
    class Meta:
        model = Post
        fields = ["id","title","slug","content","author","created_at","updated_at"]
        read_only_fields = ["author","created_at","updated_at"]
```

------------------------------------------------------------------------

## 🔐 Autorisations

``` python
class IsOwnerOrReadOnly(BasePermission):
    def has_object_permission(self, request, view, obj):
        return True if request.method in SAFE_METHODS else obj.author_id == request.user.id
```

------------------------------------------------------------------------

## 📡 Vues et endpoints

``` python
class PostViewSet(viewsets.ModelViewSet):
    queryset = Post.objects.select_related("author")
    serializer_class = PostSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
    filter_backends = [filters.SearchFilter, filters.OrderingFilter]
    search_fields = ["title","content","author__username"]
    ordering_fields = ["created_at","title"]
    def perform_create(self, serializer):
        serializer.save(author=self.request.user)
```

------------------------------------------------------------------------

## 🌐 Routage et JWT

``` python
router = DefaultRouter()
router.register("posts", PostViewSet, basename="post")
urlpatterns = [
    path("api/", include(router.urls)),
    path("api/token/", TokenObtainPairView.as_view(), name="token_obtain_pair"),
    path("api/token/refresh/", TokenRefreshView.as_view(), name="token_refresh"),
]
```

------------------------------------------------------------------------

## ⚖️ Configuration avancée

``` python
REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": ("rest_framework_simplejwt.authentication.JWTAuthentication",),
    "DEFAULT_PERMISSION_CLASSES": ("rest_framework.permissions.IsAuthenticatedOrReadOnly",),
    "DEFAULT_PAGINATION_CLASS": "rest_framework.pagination.PageNumberPagination",
    "PAGE_SIZE": 20,
}
SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
    "ROTATE_REFRESH_TOKENS": True,
    "BLACKLIST_AFTER_ROTATION": True,
}
```

------------------------------------------------------------------------

## 🧪 Tests automatisés

``` python
@pytest.mark.django_db
def test_create_post_requires_auth():
    client = APIClient()
    resp = client.post("/api/posts/", {"title": "Test", "slug": "test", "content": "..."}, format="json")
    assert resp.status_code == 401
```

------------------------------------------------------------------------

## 🚦 Bonnes pratiques

-   Jamais de `fields='__all__'` côté API publique.
-   Activer `python manage.py check --deploy` avant mise en prod.
-   Rotation + blacklist des JWT.
-   Limiter CORS aux domaines connus.
-   Indexer les champs filtrés.
-   Toujours utiliser `select_related` / `prefetch_related`.

------------------------------------------------------------------------

## ⚠️ Pièges courants

-   `DEBUG=True` en production.
-   JWT stockés en localStorage → préférer header HTTP.
-   Throttling ≠ protection DoS.
-   CORS permissif (`*`) en prod.

------------------------------------------------------------------------

## 📊 Schéma JWT

``` mermaid
sequenceDiagram
Client->>API: POST /api/token
API-->>Client: access + refresh
Client->>API: GET /api/posts (Bearer access)
API-->>Client: 200 OK
Client->>API: POST /api/token/refresh
API-->>Client: new access
```

------------------------------------------------------------------------

## 📎 Références

-   [Django](https://docs.djangoproject.com/en/stable/)
-   [DRF](https://www.django-rest-framework.org/)
-   [SimpleJWT](https://django-rest-framework-simplejwt.readthedocs.io/)
-   [Django Deployment
    Checklist](https://docs.djangoproject.com/en/stable/howto/deployment/checklist/)
-   [OWASP API Security Top 10](https://owasp.org/API-Security/)
