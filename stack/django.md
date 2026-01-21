# Django (2025)

> **Last updated**: January 2026
> **Versions covered**: 5.0, 5.1, 5.2 LTS (supported until April 2028)
> **Purpose**: High-level Python web framework for rapid development

---

## Philosophy (2025-2026)

Django remains the **"batteries-included" framework** for Python web development, emphasizing DRY (Don't Repeat Yourself) and rapid development.

**Key philosophical shifts:**
- **Fat models, thin views** — Business logic in models/services
- **Apps = bounded contexts** — Each app does one thing well (see App Segmentation below)
- **Async ORM maturing** — Django 5.2 adds async auth, all QuerySet methods have `a-` variants
- **Type hints adoption** — Better IDE support, use `django-stubs`
- **Django Ninja rising** — For API-first projects (alternative to DRF)
- **Celery for background tasks** — Signals are synchronous
- **Composite primary keys** — New in Django 5.2 for legacy DB integration

---

## App Segmentation: When to Split vs. Single App

This is one of the most important architectural decisions in Django. **There's no one-size-fits-all answer.**

### Use Multiple Apps When:

| Scenario | Reasoning |
|----------|-----------|
| **Large team** (5+ devs) | Reduces merge conflicts, clear ownership |
| **Reusable components** | Auth, payments, notifications can be extracted |
| **Future microservices** | Apps become natural service boundaries |
| **Different release cycles** | Decouple features that evolve independently |
| **70+ models** | Single app becomes unmaintainable |

### Use Single Core App When:

| Scenario | Reasoning |
|----------|-----------|
| **Small team** (1-3 devs) | Less overhead, faster iteration |
| **MVP/Prototype** | Speed over structure |
| **Tightly coupled domain** | Everything references everything |
| **No reuse planned** | YAGNI — You Aren't Gonna Need It |

### Decision Framework

```
┌─────────────────────────────────────────────────────────────┐
│ Can this feature be used in another project?                │
│   YES → Separate app                                        │
│   NO  → Could it become a microservice later?               │
│           YES → Separate app                                │
│           NO  → Does it have its own data model?            │
│                   YES → Consider separate app               │
│                   NO  → Keep in existing app                │
└─────────────────────────────────────────────────────────────┘
```

### Nested Modules for Large Apps

When an app grows too large but splitting isn't warranted, use **internal modules**:

```
apps/shop/                      # Single "shop" app
├── __init__.py
├── models/                     # Split models into modules
│   ├── __init__.py             # from .product import Product; from .order import Order
│   ├── product.py
│   ├── order.py
│   └── customer.py
├── services/                   # Business logic modules
│   ├── __init__.py
│   ├── cart.py
│   ├── checkout.py
│   └── inventory.py
├── views/
│   ├── __init__.py
│   ├── product_views.py
│   └── order_views.py
├── admin.py
├── urls.py
└── tests/
    ├── test_cart.py
    └── test_checkout.py
```

```python
# apps/shop/models/__init__.py
from .product import Product, Category
from .order import Order, OrderItem
from .customer import Customer

__all__ = ['Product', 'Category', 'Order', 'OrderItem', 'Customer']
```

### Real-World Examples

| Project Type | Recommended Structure |
|--------------|----------------------|
| Blog | Single `blog` app |
| SaaS MVP | `core`, `accounts`, `billing` (3 apps) |
| E-commerce | `products`, `orders`, `customers`, `payments`, `inventory` |
| Large platform | 10-20+ apps with clear boundaries |

---

## TL;DR

- Group apps in an `apps/` directory for large projects
- One app = one purpose (auth, blog, payments)
- Business logic in services, not views
- Use `queryset.count()` not `len(queryset)`
- Use F() expressions for database-level operations
- Signals are synchronous — use Celery for background work
- Environment isolation with virtualenv
- Write tests — Django's test framework is powerful

---

## Best Practices

### Project Structure

```
my_project/
├── config/                 # Project configuration
│   ├── __init__.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py         # Shared settings
│   │   ├── development.py  # Dev settings
│   │   └── production.py   # Prod settings
│   ├── urls.py
│   ├── wsgi.py
│   └── asgi.py
├── apps/                   # All Django apps
│   ├── __init__.py
│   ├── users/
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── forms.py
│   │   ├── services.py     # Business logic
│   │   ├── selectors.py    # Query logic
│   │   ├── admin.py
│   │   └── tests/
│   ├── blog/
│   └── payments/
├── templates/              # Project-level templates
│   ├── base.html
│   └── components/
├── static/                 # Project-level static files
├── media/                  # User uploads
├── tests/                  # Integration tests
├── requirements/
│   ├── base.txt
│   ├── development.txt
│   └── production.txt
├── manage.py
└── .env
```

### Settings Organization

```python
# config/settings/base.py
from pathlib import Path
import os
from dotenv import load_dotenv

load_dotenv()

BASE_DIR = Path(__file__).resolve().parent.parent.parent

SECRET_KEY = os.getenv('SECRET_KEY')

# Application definition
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    # Third-party
    'rest_framework',
    'corsheaders',
    # Local apps
    'apps.users',
    'apps.blog',
    'apps.payments',
]

# Static files
STATIC_URL = '/static/'
STATICFILES_DIRS = [BASE_DIR / 'static']
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Media files
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

# Templates
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        ...
    },
]
```

```python
# config/settings/development.py
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME', 'myproject_dev'),
        'USER': os.getenv('DB_USER', 'postgres'),
        'PASSWORD': os.getenv('DB_PASSWORD', ''),
        'HOST': os.getenv('DB_HOST', 'localhost'),
        'PORT': os.getenv('DB_PORT', '5432'),
    }
}
```

### Model Design

```python
# apps/blog/models.py
from django.db import models
from django.contrib.auth import get_user_model
from django.utils import timezone

User = get_user_model()

class TimeStampedModel(models.Model):
    """Abstract base model with created/updated timestamps."""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class Post(TimeStampedModel):
    """Blog post model."""
    class Status(models.TextChoices):
        DRAFT = 'draft', 'Draft'
        PUBLISHED = 'published', 'Published'
        ARCHIVED = 'archived', 'Archived'

    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    author = models.ForeignKey(
        User,
        on_delete=models.CASCADE,
        related_name='posts'
    )
    content = models.TextField()
    status = models.CharField(
        max_length=20,
        choices=Status.choices,
        default=Status.DRAFT,
        db_index=True
    )
    published_at = models.DateTimeField(null=True, blank=True)
    views_count = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['-published_at']
        indexes = [
            models.Index(fields=['slug']),
            models.Index(fields=['status', '-published_at']),
        ]

    def __str__(self):
        return self.title

    def publish(self):
        """Publish the post."""
        self.status = self.Status.PUBLISHED
        self.published_at = timezone.now()
        self.save(update_fields=['status', 'published_at', 'updated_at'])
```

### Services Layer (Business Logic)

```python
# apps/blog/services.py
from django.db import transaction
from django.core.exceptions import ValidationError
from .models import Post
from .selectors import get_post_by_slug

class PostService:
    @staticmethod
    @transaction.atomic
    def create_post(*, author, title: str, content: str, slug: str) -> Post:
        """Create a new blog post."""
        if Post.objects.filter(slug=slug).exists():
            raise ValidationError(f"Post with slug '{slug}' already exists")

        post = Post.objects.create(
            author=author,
            title=title,
            content=content,
            slug=slug,
        )
        return post

    @staticmethod
    def publish_post(*, post_id: int) -> Post:
        """Publish a draft post."""
        post = Post.objects.select_for_update().get(id=post_id)

        if post.status == Post.Status.PUBLISHED:
            raise ValidationError("Post is already published")

        post.publish()
        return post

    @staticmethod
    def increment_views(*, post_id: int) -> None:
        """Increment post view count atomically."""
        from django.db.models import F
        Post.objects.filter(id=post_id).update(
            views_count=F('views_count') + 1
        )
```

### Selectors Layer (Query Logic)

```python
# apps/blog/selectors.py
from django.db.models import QuerySet, Prefetch
from .models import Post

def get_published_posts() -> QuerySet[Post]:
    """Get all published posts."""
    return Post.objects.filter(
        status=Post.Status.PUBLISHED
    ).select_related('author')

def get_post_by_slug(slug: str) -> Post | None:
    """Get a single post by slug."""
    return Post.objects.filter(slug=slug).select_related('author').first()

def get_posts_by_author(author_id: int) -> QuerySet[Post]:
    """Get all posts by a specific author."""
    return Post.objects.filter(
        author_id=author_id
    ).order_by('-created_at')

def get_popular_posts(limit: int = 10) -> QuerySet[Post]:
    """Get most viewed published posts."""
    return Post.objects.filter(
        status=Post.Status.PUBLISHED
    ).order_by('-views_count')[:limit]
```

### Views (Thin Views)

```python
# apps/blog/views.py
from django.views.generic import ListView, DetailView
from django.shortcuts import get_object_or_404
from .selectors import get_published_posts, get_post_by_slug
from .services import PostService

class PostListView(ListView):
    template_name = 'blog/post_list.html'
    context_object_name = 'posts'
    paginate_by = 10

    def get_queryset(self):
        return get_published_posts()


class PostDetailView(DetailView):
    template_name = 'blog/post_detail.html'
    context_object_name = 'post'

    def get_object(self):
        post = get_post_by_slug(self.kwargs['slug'])
        if not post:
            raise Http404("Post not found")
        # Increment views
        PostService.increment_views(post_id=post.id)
        return post
```

### QuerySet Optimization

```python
# ✅ Use select_related for ForeignKey/OneToOne
posts = Post.objects.select_related('author').all()

# ✅ Use prefetch_related for ManyToMany/reverse FK
posts = Post.objects.prefetch_related('comments', 'tags').all()

# ✅ Use F() for database-level operations
from django.db.models import F
Post.objects.filter(id=1).update(views_count=F('views_count') + 1)

# ✅ Use count() not len()
count = Post.objects.filter(status='published').count()

# ✅ Use exists() for existence checks
if Post.objects.filter(slug='my-post').exists():
    pass

# ✅ Use only() or defer() to limit fields
posts = Post.objects.only('id', 'title', 'slug').all()

# ✅ Use bulk operations
Post.objects.bulk_create([Post(...), Post(...)])
Post.objects.bulk_update(posts, ['status'])
```

### Forms and Validation

```python
# apps/blog/forms.py
from django import forms
from .models import Post

class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'slug', 'content', 'status']

    def clean_slug(self):
        slug = self.cleaned_data['slug']
        if Post.objects.filter(slug=slug).exclude(pk=self.instance.pk).exists():
            raise forms.ValidationError("This slug is already in use.")
        return slug
```

### Async Views & ORM (Django 5.2+)

```python
# apps/blog/views.py
from django.http import JsonResponse

# Django 5.2: Native async ORM — no more sync_to_async needed!
async def async_post_list(request):
    """Async view with native async ORM."""
    posts = []
    # Use async for on QuerySets directly
    async for post in Post.objects.filter(status='published')[:10]:
        posts.append({'id': post.id, 'title': post.title})
    return JsonResponse({'posts': posts})

# Async model methods
async def async_create_post(request):
    """Using async model methods."""
    post = await Post.objects.acreate(
        title='My Post',
        slug='my-post',
        author=request.user,
        content='...'
    )
    return JsonResponse({'id': post.id})

# Async aggregations
async def async_stats(request):
    """Async count and aggregations."""
    count = await Post.objects.filter(status='published').acount()
    exists = await Post.objects.filter(slug='my-post').aexists()
    return JsonResponse({'count': count, 'exists': exists})
```

### Composite Primary Keys (Django 5.2+)

```python
# For legacy databases or many-to-many with extra fields
from django.db import models

class OrderItem(models.Model):
    """Table with composite primary key."""
    pk = models.CompositePrimaryKey('order_id', 'product_id')
    order = models.ForeignKey('Order', on_delete=models.CASCADE)
    product = models.ForeignKey('Product', on_delete=models.CASCADE)
    quantity = models.PositiveIntegerField()
    price = models.DecimalField(max_digits=10, decimal_places=2)

    class Meta:
        # No need for unique_together anymore
        pass

# Usage
item = OrderItem.objects.get(pk=(order_id, product_id))
```

### Django 5.0+ Field Groups (Form Rendering)

```python
# Template rendering made easier
# In template:
{{ form.title.as_field_group }}  # Renders label, input, errors, help_text

# Customize the template project-wide in settings:
FORM_RENDERER = 'django.forms.renderers.TemplatesSetting'

# Or per-field:
class MyForm(forms.Form):
    title = forms.CharField(
        bound_field_class=CustomBoundField  # Custom rendering
    )
```

---

## Anti-Patterns

### ❌ Fat Views

**Why it's bad**: Business logic scattered across views is hard to test and reuse.

```python
# ❌ DON'T — Logic in view
def create_post(request):
    if Post.objects.filter(slug=request.POST['slug']).exists():
        # validation logic here
        pass
    post = Post.objects.create(...)
    # send email logic here
    # update statistics here
    return redirect('post_detail', post.slug)

# ✅ DO — Use services
def create_post(request):
    form = PostForm(request.POST)
    if form.is_valid():
        post = PostService.create_post(
            author=request.user,
            **form.cleaned_data
        )
        return redirect('post_detail', post.slug)
```

### ❌ Using len() on QuerySets

**Why it's bad**: Fetches all objects into memory just to count.

```python
# ❌ DON'T
if len(Post.objects.filter(status='published')) > 0:
    pass

# ✅ DO
if Post.objects.filter(status='published').exists():
    pass

# ✅ DO — For counting
count = Post.objects.filter(status='published').count()
```

### ❌ Queries in Templates

**Why it's bad**: N+1 queries, hard to optimize.

```html
<!-- ❌ DON'T -->
{% for post in posts %}
    <p>Author: {{ post.author.username }}</p>  <!-- Query per post! -->
{% endfor %}

<!-- ✅ DO — Use select_related in view -->
```

### ❌ Overloading save() with Side Effects

**Why it's bad**: `bulk_create` bypasses `save()`.

```python
# ❌ DON'T
class Post(models.Model):
    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        send_notification_email()  # Won't run on bulk_create!

# ✅ DO — Use services or signals
```

### ❌ Misusing Signals

**Why it's bad**: Signals are synchronous and create hidden dependencies.

```python
# ❌ DON'T — Heavy operations in signals
@receiver(post_save, sender=Post)
def send_emails(sender, instance, **kwargs):
    send_email_to_subscribers(instance)  # Blocks the request!

# ✅ DO — Use Celery for background tasks
@receiver(post_save, sender=Post)
def queue_email_task(sender, instance, **kwargs):
    send_email_task.delay(instance.id)
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| 5.0 | Dec 2023 | Field groups (`as_field_group`), database-computed default values, `GeneratedField` |
| 5.1 | Aug 2024 | Async auth backends, `@login_required` for async views, querystring template tag |
| **5.2 LTS** | **Apr 2025** | **Composite primary keys**, shell auto-imports, async user/permission methods, `utf8mb4` MySQL default |
| 5.2.8 | Late 2025 | Python 3.14 support added |
| 6.0 | Expected 2026 | Next major release |

### Support Timeline (Important)

| Version | Support Until | Notes |
|---------|---------------|-------|
| Django 4.2 LTS | **April 2026** | Upgrade to 5.2 LTS before this |
| Django 5.1 | **December 2025** | Security/data loss fixes only |
| **Django 5.2 LTS** | **April 2028** | Current recommended version |

### Django 5.2 LTS Highlights (Supported until April 2028)

**Composite Primary Keys (Long-Awaited Feature):**
```python
from django.db import models

class OrderItem(models.Model):
    pk = models.CompositePrimaryKey('order_id', 'product_id')
    order = models.ForeignKey('Order', on_delete=models.CASCADE)
    product = models.ForeignKey('Product', on_delete=models.CASCADE)
    quantity = models.PositiveIntegerField()

# Usage
item = OrderItem.objects.get(pk=(order_id, product_id))
```

**Limitations of Composite PKs:**
- Cannot migrate to/from composite PK (hard in plain SQL anyway)
- Relationship fields (FK, O2O, M2M, GenericFK) cannot point to composite PK models
- Django admin does not support composite PKs
- Workaround: Use `ForeignObject` internal API for relationships

**Shell Auto-imports (DX Boost):**
```bash
python manage.py shell
# Models from ALL installed apps are auto-imported!
>>> Post.objects.count()  # No import needed
>>> User.objects.filter(is_active=True)  # Just works
```
- Inspired by `django-extensions` shell_plus
- Now built into Django core

**Async Auth Methods:**
```python
# New async methods on User model
user = await User.objects.aget(pk=1)
has_perm = await user.ahas_perm('blog.add_post')
groups = [g async for g in user.groups.all()]
```

**Customizable Form Rendering (BoundField):**
```python
# Specify custom BoundField at three levels:
# 1. Project Level (settings)
# 2. Form Level
# 3. Field Level
class MyForm(forms.Form):
    title = forms.CharField(
        bound_field_class=CustomBoundField
    )
```

**Database Changes:**
- PostgreSQL 13 support ends November 2025 → **PostgreSQL 14+ required**
- MySQL connections now default to **utf8mb4** (not deprecated utf8mb3)
- `response.text` returns string representation of body
- `HttpRequest.accepted_types` sorted by client preference
- `QuerySet.values()/values_list()` maintains specified field order

**Python Support:** 3.10, 3.11, 3.12, 3.13, 3.14 (as of 5.2.8)

### Community Events 2026

- **DjangoCon Europe 2026**: April 15, 2026 — Athens, Greece
- **DjangoCon US 2026**: September 14, 2026 — Chicago, Illinois, USA

---

## Quick Reference

| Task | Solution |
|------|----------|
| Create project | `django-admin startproject config .` |
| Create app | `python manage.py startapp appname` |
| Migrations | `python manage.py makemigrations && migrate` |
| Shell | `python manage.py shell_plus` |
| Run server | `python manage.py runserver` |
| Create superuser | `python manage.py createsuperuser` |
| Run tests | `python manage.py test` |
| Collect static | `python manage.py collectstatic` |

---

## Resources

- [Official Django Documentation](https://docs.djangoproject.com/)
- [Django Anti-Patterns](https://www.django-antipatterns.com/)
- [Django Best Practices 2025](https://medium.com/write-a-catalyst/django-best-practices-2025-settings-apps-and-structure-9c96970c1a5b)
- [LearnDjango - Projects vs Apps](https://learndjango.com/tutorials/django-best-practices-projects-vs-apps)
- [Django Project Structure Guide](https://studygyaan.com/django/best-practice-to-structure-django-project-directories-and-files)
- [Toptal - Top 10 Django Mistakes](https://www.toptal.com/developers/django/django-top-10-mistakes)
