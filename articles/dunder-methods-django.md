# Comment Revisiter les Dunder Methods de Python a Amélioré ma Façon de Concevoir des APIs Django

---

## 🔄 Revenir aux Bases pour Mieux Coder

Il y a quelques semaines, j’ai pris le temps de revisiter un pan souvent survolé du langage Python : les dunder methods, ces méthodes "magiques" encadrées par des doubles underscores comme `__str__`, `__getitem__`, ou encore `__iter__`. 

Mais pourquoi s’y replonger ?

Parce qu’elles permettent de rendre nos objets **plus expressifs, plus Pythonic** et donc plus faciles à utiliser, débugger et maintenir.

En les explorant à nouveau, j’ai découvert qu’elles pouvaient également avoir un impact direct sur la **conception de mes APIs Django**, notamment dans la clarté du code, la recherche, le tri et la pagination.

---

## 🔍 C’est Quoi une Dunder Method ?

Les dunder methods ("**d**ouble **under**score") sont des méthodes spéciales de Python qui permettent de **personnaliser le comportement natif de vos objets**.

### ✨ Exemples courants

| Méthode       | Rôle                                                |
|----------------|--------------------------------------------------------|
| `__str__`      | Affichage lisible (admin Django, logs, shell)          |
| `__repr__`     | Affichage pour debug (repr() dans le shell)            |
| `__getitem__`  | Accès par index ou clé : `obj["champ"]`                |
| `__iter__`     | Rendre l’objet itérable dans une boucle `for`          |
| `__eq__`       | Comparaison d’objets avec `==`                         |
| `__lt__`       | Tri d’objets avec `sorted()`                          |

Ces méthodes ne sont pas là pour faire joli : elles donnent à vos objets **le même comportement que les types natifs Python** (listes, dictionnaires, etc.).

---

## 🚀 Application à un Projet Django : le Cas du Modèle `Product`

Prenons un modèle Django simple :

```python
class Product(models.Model):
    name = models.CharField(max_length=255)
    category = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    description = models.TextField()

    def __str__(self):
        return f"{self.name} - {self.category}"
```

Le `__str__` ici permet un affichage humain dans l’admin, les logs ou le shell. Sans lui : `Product object (1)`.

Mais allons plus loin.

---

## 🧰 Utiliser `__getitem__` pour un accès dynamique aux champs

```python
class Product(models.Model):
    ...

    def __getitem__(self, key):
        return getattr(self, key)
```

▶️ Maintenant :
```python
p = Product(name="Banana", category="Fruit")
print(p["name"])  # Banana
```

Utile ? Oui, surtout pour des serializers, des templates ou du code générique.

---

## 🌍 Rendre un Modèle Triable avec `__lt__`

```python
class Product(models.Model):
    ...

    def __lt__(self, other):
        return self.price < other.price
```

Maintenant :
```python
products = list(Product.objects.all())
products.sort()  # Triés par prix croissant
```

---

## 🚫 Une Confusion Fréquente : Pagination et Dunders ?

L’article original fait un lien entre la pagination DRF et les dunders. Or, **la pagination est une abstraction DRF**, et non un mécanisme Python natif.

C’est donc à tort qu’on assimile cela à `__getitem__`.

---

## 💡 Bonus : Recherche Full-Text Dynamique avec DRF

L’auteur propose une recherche générique sur tous les champs via introspection du modèle :

```python
for field in Product._meta.get_fields():
    if hasattr(field, 'attname'):
        q_objects |= Q(**{f"{field.attname}__icontains": search_query})
```

### ⚠️ Risques :
- Inclusion de champs FK, OneToOne, etc.
- Moins performant

### ✅ Proposition améliorée

```python
SEARCHABLE_FIELDS = ['name', 'category', 'description']

def get_queryset(self):
    queryset = super().get_queryset()
    search_query = self.request.query_params.get('search')
    if search_query:
        q_objects = Q()
        for field in self.SEARCHABLE_FIELDS:
            q_objects |= Q(**{f"{field}__icontains": search_query})
        queryset = queryset.filter(q_objects)
    return queryset
```

---

## 🚀 Conclusion : Pourquoi Revenir aux Dunder Methods

Revisiter les dunder methods en tant que développeur Python, c’est :

- 🤝 Améliorer la lisibilité des objets (logs, admin, debugging)
- 🌬️ Simplifier l’accès dynamique aux données (via `__getitem__`)
- ↑ Renforcer la compatibilité avec les outils Python standards (`sorted`, `dict`, `set`, etc.)
- 🛠️ Enrichir la logique métier sans alourdir les vues ou serializers

---

## 🔄 Pour Résumer

| Objectif                  | Dunder utilisé           | Bénéfice concret                      |
|--------------------------|---------------------------|----------------------------------------|
| Affichage humain         | `__str__`                 | Lisibilité dans l’admin/logs           |
| Accès dynamique         | `__getitem__`             | Code générique plus simple              |
| Tri d’objets             | `__lt__` / `__gt__`        | `sorted()` sur une liste de modèles     |
| Comparaison              | `__eq__`, `__hash__`      | Ajout dans un `set`, test d’égalité     |

---

## 🚀 Et Maintenant ?

Avant de complexifier votre code avec des librairies ou du code ad hoc, **regardez si un dunder peut faire le job**.

Parfois, une simple méthode `__str__`, `__lt__` ou `__getitem__` peut rendre votre API Django **plus intuitive, plus débogable et plus propre**.