# 🌾 Django Project Mastery

> **👋 Hey Fresher — Read This First!**

> - Django is a "batteries-included" Python web framework — it ships with an ORM (talks to your database without writing raw SQL), an admin panel, a forms system, and built-in authentication, so you spend your time on business logic instead of plumbing.
> - The core idea is **MTV** — Models (your data), Templates (your HTML), Views (the logic that connects the two). Django calls the URL router first, the router calls a view, the view talks to models and renders a template.
> - This document uses **short, complete code blocks** — each one covers exactly one Django concept — followed by a plain-English explanation of every important line.
> - **Company in this project:** KhetiSetu — an agri-tech startup in Nashik that connects farmers directly to grocery buyers, cutting out middlemen who were taking 30-40% margins. You just joined as a Junior Backend Developer. Your senior engineer is Sunita, and your engineering lead is Devendra. You will build KhetiSetu's core marketplace backend in Django — from the first model to a working REST API for the mobile app.

#### What You Will Learn and Build in This Project

You will build **KhetiSetu's produce marketplace backend** — the same kind of system used by real agri-tech and e-commerce platforms — starting from a blank Django project and ending with an authenticated, tested, API-backed application that farmers and buyers can actually use.

Django Project & App Structure, Models & Migrations, Django Admin, Class-Based Views, URL Routing, Templates & Template Inheritance, ModelForms, Authentication & Authorization, QuerySet Scoping, Django REST Framework, Serializers & ViewSets

> **📦 Phase 1 — Project Setup, Models & Migrations**
>
> Scaffold the Django project, create the `marketplace` app, and design the `Farmer`, `Produce`, and `Order` models that everything else is built on.

> **🛠️ Phase 2 — Django Admin**
>
> Register the models in Django's built-in admin so KhetiSetu's operations team can manage listings without needing a custom UI on day one.

> **🌐 Phase 3 — Views & URL Routing**
>
> Build list, detail, create, and update views for produce listings, and wire them into the URL router.

> **🎨 Phase 4 — Templates & Forms**
>
> Build a shared base template, list/detail pages, and a `ModelForm` so farmers can list produce through a web form instead of the admin.

> **🔐 Phase 5 — Authentication & Authorization**
>
> Add login-protected views so farmers only see and edit their own listings, and buyers can only see their own orders.

> **📡 Phase 6 — REST API with Django REST Framework**
>
> Expose the same models as a JSON API so KhetiSetu's Android app can list produce, place orders, and check order status.

**Scene 1 — KhetiSetu Office, Nashik | The Middleman Problem**

> **Devendra** _Engineering Lead — KhetiSetu_
>
> Ishaan, here's the problem we're solving. A farmer in Niphad grows tomatoes worth ₹18 per kg. By the time it reaches a Nashik grocery store through two middlemen, it's ₹34 per kg — and the farmer only got ₹11 of that. We're building a platform where farmers list produce directly and buyers — grocery stores, hotels, caterers — order directly. No middlemen. Django is our backend. You'll be building the core marketplace this week.

> **Ishaan (You)** _Junior Backend Developer — Day 1 at KhetiSetu_
>
> I've used Django for a college project — a to-do app. This sounds a lot bigger. Where do I even start?

> **Sunita** _Senior Backend Engineer — KhetiSetu_
>
> Same principles, bigger stakes. Start with the data — what are the "nouns" in this business? A Farmer lists Produce. A Buyer places an Order for that Produce. Once the models are right, the rest of Django — admin, views, forms, API — all builds on top of them. We never start with views or templates. We always start with models.

> **Devendra** _Engineering Lead_
>
> One more thing — we're not writing raw SQL anywhere in this codebase. Django's ORM handles that, and it also protects us from SQL injection automatically as long as you stick to the ORM. Let's get your first migration running.

### 1. Phase 1 — Project Setup, Models & Migrations

**Business Problem:** KhetiSetu needs a database that represents farmers, their produce listings (with price, quantity, and harvest date), and the orders buyers place against those listings. Get this wrong and every feature built on top of it — admin, views, API — inherits the mistake.

#### 1.1 Scaffold the Project and App

```
# Create a virtual environment and install Django
python3 -m venv venv
source venv/bin/activate
pip install django djangorestframework

# Start the project and the marketplace app
django-admin startproject khetisetu .
python manage.py startapp marketplace
```

> **📖 Project vs App**
>
> - **Project** (`khetisetu`) — the whole Django site: settings, root URL config, WSGI/ASGI entry points. A project can contain many apps.
> - **App** (`marketplace`) — one self-contained piece of functionality: models, views, templates, migrations for one feature area. KhetiSetu will later add a `payments` app and a `logistics` app the same way.
> - After `startapp`, add `"marketplace"` to `INSTALLED_APPS` in `khetisetu/settings.py` — Django ignores apps it doesn't know about.

#### 1.2 Define the Models

```python
# marketplace/models.py
from django.db import models
from django.contrib.auth.models import User


class Farmer(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)
    village = models.CharField(max_length=100)
    phone_number = models.CharField(max_length=15)
    verified = models.BooleanField(default=False)

    def __str__(self):
        return f"{self.user.get_full_name()} ({self.village})"


class Produce(models.Model):
    UNIT_CHOICES = [
        ("kg", "Kilogram"),
        ("dozen", "Dozen"),
        ("crate", "Crate"),
    ]

    farmer = models.ForeignKey(Farmer, on_delete=models.CASCADE, related_name="listings")
    name = models.CharField(max_length=120)
    price_per_unit = models.DecimalField(max_digits=8, decimal_places=2)
    unit = models.CharField(max_length=10, choices=UNIT_CHOICES, default="kg")
    quantity_available = models.PositiveIntegerField()
    harvested_on = models.DateField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"{self.name} - {self.farmer.village}"


class Order(models.Model):
    STATUS_CHOICES = [
        ("pending", "Pending"),
        ("confirmed", "Confirmed"),
        ("delivered", "Delivered"),
        ("cancelled", "Cancelled"),
    ]

    buyer = models.ForeignKey(User, on_delete=models.CASCADE, related_name="orders")
    produce = models.ForeignKey(Produce, on_delete=models.CASCADE, related_name="orders")
    quantity = models.PositiveIntegerField()
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default="pending")
    placed_at = models.DateTimeField(auto_now_add=True)
```

> **📖 What Each Field Type Does**
>
> - **OneToOneField(User, ...)** — extends Django's built-in `User` model with farmer-specific fields instead of replacing it. One `User` row maps to exactly one `Farmer` row.
> - **ForeignKey(Farmer, related_name="listings")** — a many-to-one relationship: one farmer has many `Produce` listings. `related_name="listings"` lets you write `farmer.listings.all()` instead of the clunkier `farmer.produce_set.all()`.
> - **on_delete=models.CASCADE** — if a `Farmer` is deleted, delete their `Produce` listings too. Django forces you to declare this behavior explicitly for every ForeignKey — there's no silent default.
> - **DecimalField, not FloatField** — for money, always use `DecimalField`. Floats introduce rounding errors (₹18.10 can become ₹18.099999...) that are unacceptable in a marketplace that handles real payments.
> - **choices=** — restricts a `CharField` to a fixed set of values and renders as a dropdown in forms and the admin automatically.

#### 1.3 Create and Run Migrations

```
python manage.py makemigrations marketplace
python manage.py migrate
```

**Migration comparison**

- **makemigrations** — reads your `models.py`, compares it to the last known state, and writes a migration file describing the change (add table, add column, etc). Nothing touches the database yet.
- **migrate** — actually applies pending migration files to the database, running the SQL Django generated. Run this after every `makemigrations`, and after pulling teammates' migrations from Git.

> **Key takeaways**
> - Always design models before views — the data shape drives everything downstream.
> - Use `DecimalField` for money, never `FloatField`.
> - `related_name` makes reverse lookups (`farmer.listings.all()`) readable — set it deliberately, not by default.
> - `makemigrations` writes the plan; `migrate` executes it. Commit migration files to Git.

### 2. Phase 2 — Django Admin

**Business Problem:** KhetiSetu's operations team needs to verify new farmers, spot-check produce listings for suspicious pricing, and manage orders — today, without waiting for a custom dashboard to be built. Django's admin gives them this for free.

#### 2.1 Register the Models

```python
# marketplace/admin.py
from django.contrib import admin
from .models import Farmer, Produce, Order


@admin.register(Farmer)
class FarmerAdmin(admin.ModelAdmin):
    list_display = ("user", "village", "phone_number", "verified")
    list_filter = ("verified", "village")
    search_fields = ("user__first_name", "user__last_name", "phone_number")
    actions = ["mark_verified"]

    @admin.action(description="Mark selected farmers as verified")
    def mark_verified(self, request, queryset):
        queryset.update(verified=True)


@admin.register(Produce)
class ProduceAdmin(admin.ModelAdmin):
    list_display = ("name", "farmer", "price_per_unit", "unit", "quantity_available", "harvested_on")
    list_filter = ("unit", "harvested_on")
    search_fields = ("name", "farmer__user__first_name")


admin.site.register(Order)
```

> **📖 Reading the ModelAdmin**
>
> - **list_display** — the columns shown on the change-list page. Without it, the admin only shows the object's `__str__()`.
> - **list_filter** — adds a sidebar filter panel. `("verified", "village")` gives ops staff one-click filtering by verification status and village.
> - **search_fields** — powers the search box at the top. `"user__first_name"` follows the ForeignKey/OneToOne relationship using Django's double-underscore lookup syntax.
> - **actions + @admin.action** — a custom bulk action. Ops staff can select 40 unverified farmers after a field visit and mark them all verified in one click, instead of editing each record individually.
> - `admin.site.register(Order)` is the plain, unstyled way to register a model — fine for models you won't customize yet.

**Quiz: KhetiSetu's ops lead wants to filter the Produce admin list by harvest date range without writing a custom view. What's the simplest way?**
- Write a custom Django view with a date-range form
- Add `"harvested_on"` to `list_filter` in `ProduceAdmin`
- Export all data to Excel and filter manually
- Add a new model field called `date_range`

> **Answer/explanation:** Adding `"harvested_on"` to `list_filter` is correct. Django's admin automatically detects that it's a `DateField` and renders a built-in date drill-down filter (Today, Past 7 days, This month, This year) with zero extra code. Writing a custom view is unnecessary engineering effort for something the admin already provides. Exporting to Excel throws away the live, searchable nature of the admin. Adding a new field doesn't filter anything — it just stores more data.

### 3. Phase 3 — Views & URL Routing

**Business Problem:** Farmers and buyers need actual web pages — browse all produce, view one listing's details, and (for farmers) create new listings. The admin panel is for KhetiSetu staff, not the public.

**Scene 2 — KhetiSetu Sprint Planning | Function-Based or Class-Based?**

> **Sunita** _Senior Backend Engineer_
>
> For the produce catalog, we need four views: list all produce, view one listing, create a listing, and update a listing. That's the exact CRUD pattern Django's generic class-based views were built for. Use those instead of writing four functions from scratch.

> **Ishaan (You)**
>
> I've only ever written function-based views. Why would generic class-based views be better here?

> **Sunita** _Senior Backend Engineer_
>
> They're not universally "better" — they're better for exactly this shape of problem: standard list/detail/create/update/delete against one model. You get pagination, queryset filtering, and template resolution for free, and the code that's left is just the KhetiSetu-specific logic, like scoping listings to the logged-in farmer.

#### 3.1 Class-Based Views

```python
# marketplace/views.py
from django.contrib.auth.mixins import LoginRequiredMixin
from django.urls import reverse_lazy
from django.views.generic import ListView, DetailView, CreateView, UpdateView
from .models import Produce


class ProduceListView(ListView):
    model = Produce
    template_name = "marketplace/produce_list.html"
    context_object_name = "listings"
    paginate_by = 20
    ordering = ["-created_at"]


class ProduceDetailView(DetailView):
    model = Produce
    template_name = "marketplace/produce_detail.html"
    context_object_name = "listing"


class ProduceCreateView(LoginRequiredMixin, CreateView):
    model = Produce
    fields = ["name", "price_per_unit", "unit", "quantity_available", "harvested_on"]
    template_name = "marketplace/produce_form.html"
    success_url = reverse_lazy("produce-list")

    def form_valid(self, form):
        form.instance.farmer = self.request.user.farmer
        return super().form_valid(form)


class ProduceUpdateView(LoginRequiredMixin, UpdateView):
    model = Produce
    fields = ["price_per_unit", "quantity_available"]
    template_name = "marketplace/produce_form.html"
    success_url = reverse_lazy("produce-list")

    def get_queryset(self):
        return Produce.objects.filter(farmer=self.request.user.farmer)
```

> **📖 What's Happening in Each View**
>
> - **ListView + paginate_by = 20** — automatically slices the queryset into pages of 20 and adds `page_obj` to the template context. No manual `LIMIT`/`OFFSET` math.
> - **LoginRequiredMixin** — put first in the class's parent list. It redirects anonymous users to the login page before the view logic ever runs.
> - **form_valid()** — CreateView already validated the form; this hook lets you set `farmer` to the logged-in user's `Farmer` record before saving, instead of trusting a hidden form field a malicious buyer could tamper with.
> - **get_queryset() on the UpdateView** — this is the security-critical line. It restricts which `Produce` rows this view can even fetch to ones owned by `request.user.farmer`. Without it, any logged-in farmer could edit `/produce/17/edit/` and change someone else's listing just by guessing the ID.

**Function-based vs class-based views**

- **Function-based views** — a plain Python function, best when the logic is genuinely one-off (a custom dashboard aggregating three models, a webhook handler) and doesn't map to standard CRUD.
- **Class-based generic views** — best for the repetitive list/detail/create/update/delete pattern KhetiSetu needs for `Produce`. Less code, consistent behavior, and mixins like `LoginRequiredMixin` compose cleanly.

#### 3.2 URL Routing

```python
# marketplace/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path("", views.ProduceListView.as_view(), name="produce-list"),
    path("produce/<int:pk>/", views.ProduceDetailView.as_view(), name="produce-detail"),
    path("produce/new/", views.ProduceCreateView.as_view(), name="produce-create"),
    path("produce/<int:pk>/edit/", views.ProduceUpdateView.as_view(), name="produce-update"),
]
```

```python
# khetisetu/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("", include("marketplace.urls")),
]
```

> **📖 URL Naming**
>
> - **`<int:pk>`** — a path converter. Django only matches this URL if the segment is an integer, and passes it into the view as `pk`. Class-based generic views use `pk` automatically to fetch the right object.
> - **name="produce-detail"** — never hardcode `/produce/17/` in a template. Use `{% url 'produce-detail' listing.pk %}` so URLs stay correct even if the path structure changes later.
> - **include("marketplace.urls")** — keeps app-level URLs inside the app, so the root `urls.py` stays a short table of contents.

### 4. Phase 4 — Templates & Forms

**Business Problem:** Farmers without technical background need a simple web form to list produce — not the Django admin, which exposes too many internal fields and looks unfamiliar to a non-technical user.

#### 4.1 Base Template with Inheritance

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>{% block title %}KhetiSetu{% endblock %}</title>
</head>
<body>
  <nav>
    <a href="{% url 'produce-list' %}">All Produce</a>
    {% if user.is_authenticated %}
      <a href="{% url 'produce-create' %}">List Produce</a>
      <span>Hi, {{ user.first_name }}</span>
    {% else %}
      <a href="{% url 'login' %}">Login</a>
    {% endif %}
  </nav>
  <main>
    {% block content %}{% endblock %}
  </main>
</body>
</html>
```

```html
<!-- marketplace/templates/marketplace/produce_list.html -->
{% extends "base.html" %}

{% block title %}Fresh Produce — KhetiSetu{% endblock %}

{% block content %}
  <h1>Fresh Produce Listings</h1>
  <ul>
    {% for listing in listings %}
      <li>
        <a href="{% url 'produce-detail' listing.pk %}">{{ listing.name }}</a>
        — ₹{{ listing.price_per_unit }} / {{ listing.get_unit_display }}
        ({{ listing.quantity_available }} available)
      </li>
    {% empty %}
      <li>No produce listed yet.</li>
    {% endfor %}
  </ul>

  {% if is_paginated %}
    {% if page_obj.has_previous %}
      <a href="?page={{ page_obj.previous_page_number }}">Previous</a>
    {% endif %}
    Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}
    {% if page_obj.has_next %}
      <a href="?page={{ page_obj.next_page_number }}">Next</a>
    {% endif %}
  {% endif %}
{% endblock %}
```

> **📖 Template Inheritance**
>
> - **`{% extends "base.html" %}`** must be the very first line — it tells Django this template fills in blocks defined in `base.html` rather than being a standalone page.
> - **`{% block content %}...{% endblock %}`** — a named region the child template overrides. Anything outside a block in the child template is ignored.
> - **`{{ listing.get_unit_display }}`** — Django auto-generates this method for any field with `choices=`. It turns the stored value `"kg"` into the human-readable label `"Kilogram"`.
> - **`{% empty %}`** inside `{% for %}` — renders only when the queryset is empty, replacing a separate `{% if listings %}` check.

#### 4.2 ModelForm for Listing Produce

```python
# marketplace/forms.py
from django import forms
from .models import Produce


class ProduceForm(forms.ModelForm):
    class Meta:
        model = Produce
        fields = ["name", "price_per_unit", "unit", "quantity_available", "harvested_on"]
        widgets = {
            "harvested_on": forms.DateInput(attrs={"type": "date"}),
        }

    def clean_price_per_unit(self):
        price = self.cleaned_data["price_per_unit"]
        if price <= 0:
            raise forms.ValidationError("Price must be greater than zero.")
        return price
```

> **📖 Why ModelForm, Not a Plain Form**
>
> - **`class Meta: model = Produce`** — the form generates its fields directly from the model, so a `DecimalField` on the model becomes the right kind of form field automatically. You never redefine the same field twice.
> - **`widgets`** — overrides how a specific field renders in HTML. Here, `harvested_on` renders as a native browser date picker instead of a plain text box.
> - **`clean_price_per_unit()`** — a field-level validation hook. Django calls any method named `clean_<fieldname>` automatically during `form.is_valid()`. This stops a farmer from accidentally listing produce at ₹0.
> - The `ProduceCreateView` from Phase 3 uses `fields = [...]` directly and Django builds an equivalent form for you — `ProduceForm` here is useful when you need custom validation like this, and you'd swap it in with `form_class = ProduceForm` instead of `fields = [...]`.

### 5. Phase 5 — Authentication & Authorization

**Business Problem:** A buyer must never see another buyer's orders, and a farmer must never edit another farmer's listing. Django's `User` model and `LoginRequiredMixin` handle "are you logged in?" — but "is this *your* data?" is a check you write yourself, every time.

**Scene 3 — KhetiSetu Bug Bash | The Data Leak That Wasn't Caught in Review**

> **Devendra** _Engineering Lead_
>
> During QA, a tester logged in as one buyer, then changed the order ID in the URL and saw a different buyer's phone number and delivery address. That's not a Django bug — it's ours. We forgot to scope the queryset.

> **Ishaan (You)**
>
> So `LoginRequiredMixin` wasn't enough?

> **Sunita** _Senior Backend Engineer_
>
> Right — `LoginRequiredMixin` only proves *someone* is logged in. It says nothing about *which* rows they're allowed to see. Every view that fetches a specific object by ID needs its own `get_queryset()` filtered to `request.user`. This is the single most common security bug in Django apps built by beginners.

```python
# marketplace/views.py (continued)
from django.views.generic import ListView, DetailView
from .models import Order


class MyOrdersListView(LoginRequiredMixin, ListView):
    model = Order
    template_name = "marketplace/order_list.html"
    context_object_name = "orders"

    def get_queryset(self):
        return Order.objects.filter(buyer=self.request.user).select_related("produce")
```

> **📖 Scoping Every Query to the Logged-In User**
>
> - **`get_queryset()` returning `filter(buyer=self.request.user)`** — no buyer can ever see another buyer's orders, no matter what ID appears in the URL, because the base queryset never includes rows they don't own.
> - **`select_related("produce")`** — a performance detail: without it, rendering the order list triggers one extra database query per order to fetch its related `Produce`. `select_related` does a SQL `JOIN` and fetches everything in one query — critical once KhetiSetu has thousands of orders.

> **Key takeaways**
> - `LoginRequiredMixin` answers "is this user logged in?" — it never answers "does this user own this row?"
> - Always scope `get_queryset()` to `request.user` in any view that fetches a specific object.
> - Use `select_related` for ForeignKey/OneToOne lookups you know you'll need, to avoid N+1 queries.

### 6. Phase 6 — REST API with Django REST Framework

**Business Problem:** KhetiSetu's Android app — used by buyers in the field with patchy connectivity — needs a JSON API, not server-rendered HTML. Django REST Framework (DRF) builds this on top of the exact same models.

#### 6.1 Serializers and ViewSets

```python
# marketplace/serializers.py
from rest_framework import serializers
from .models import Produce, Order


class ProduceSerializer(serializers.ModelSerializer):
    farmer_name = serializers.CharField(source="farmer.user.get_full_name", read_only=True)

    class Meta:
        model = Produce
        fields = ["id", "name", "price_per_unit", "unit", "quantity_available", "harvested_on", "farmer_name"]


class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ["id", "produce", "quantity", "status", "placed_at"]
        read_only_fields = ["status", "placed_at"]
```

```python
# marketplace/views.py (continued)
from rest_framework import viewsets, permissions
from .serializers import ProduceSerializer, OrderSerializer


class ProduceViewSet(viewsets.ModelViewSet):
    queryset = Produce.objects.all().order_by("-created_at")
    serializer_class = ProduceSerializer
    permission_classes = [permissions.IsAuthenticatedOrReadOnly]


class OrderViewSet(viewsets.ModelViewSet):
    serializer_class = OrderSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Order.objects.filter(buyer=self.request.user)

    def perform_create(self, serializer):
        serializer.save(buyer=self.request.user)
```

```python
# khetisetu/urls.py (continued)
from rest_framework.routers import DefaultRouter
from marketplace.views import ProduceViewSet, OrderViewSet

router = DefaultRouter()
router.register("api/produce", ProduceViewSet)
router.register("api/orders", OrderViewSet, basename="order")

urlpatterns += [
    path("", include(router.urls)),
]
```

> **📖 How the API Pieces Fit Together**
>
> - **ModelSerializer** — converts `Produce` model instances to JSON and validates incoming JSON back into model data, the same role a `ModelForm` plays for HTML forms.
> - **`source="farmer.user.get_full_name"`** — pulls a value from a related object and renames it in the JSON output as `farmer_name`, so the Android app doesn't need to make a second request just to show who's selling.
> - **`ModelViewSet`** — bundles list, retrieve, create, update, and delete into one class. Paired with a `DefaultRouter`, it generates all the standard REST URLs (`GET /api/produce/`, `POST /api/produce/`, `GET /api/produce/5/`, etc.) automatically.
> - **`permission_classes = [permissions.IsAuthenticatedOrReadOnly]`** — anyone can browse produce listings (`GET`), but only logged-in farmers can create or edit them (`POST`/`PUT`).
> - **`get_queryset()` + `perform_create()` on OrderViewSet** — the same ownership pattern from Phase 5, now applied to the API: buyers only ever see their own orders, and a new order is automatically tagged to whoever is authenticated — never to a `buyer` ID the client could fake in the request body.

**Quiz: A buyer sends `POST /api/orders/` with `{"buyer": 42, "produce": 7, "quantity": 10}`, trying to place an order under a different buyer's account. What actually happens with the OrderViewSet above?**
- The order is created for buyer ID 42 as requested
- DRF rejects the request with a 403 Forbidden error
- The order is created, but `buyer` is silently forced to the authenticated user, ignoring the `"buyer": 42` in the request body
- The API crashes with a 500 error

> **Answer/explanation:** The correct answer is that the order is created but `buyer` is silently forced to the authenticated user. `OrderSerializer` doesn't even include `buyer` in its `fields`, so DRF ignores that key in the incoming JSON entirely, and `perform_create()` explicitly sets `buyer=self.request.user` when saving. There's no crash and no 403 — the request just succeeds with the correct, safe buyer. This is the standard DRF pattern for preventing clients from spoofing ownership fields: keep the field out of the serializer and set it server-side in `perform_create`.

##### 🏋️ Hands-On Exercises — Extend the Project

1. **Add a `Category` model:** Create a `Category` model (Vegetables, Fruits, Grains, Dairy) with a `ForeignKey` from `Produce`. Update `ProduceListView` to support filtering by category via a query parameter, e.g. `/?category=vegetables`.
2. **Prevent overselling:** Add a `clean()` method (model-level validation) to `Order` that raises a `ValidationError` if `quantity` requested exceeds `produce.quantity_available`. Call `full_clean()` before saving in `perform_create`.
3. **Add order status transitions:** Build an admin action that moves selected orders from `pending` to `confirmed`, and reduces `Produce.quantity_available` by the ordered amount when confirmed — using `F()` expressions to avoid race conditions.
4. **Farmer dashboard view:** Build a `FarmerDashboardView` (class-based, `LoginRequiredMixin`) showing the logged-in farmer's total listings, total quantity sold, and pending orders — using `aggregate()` and `Sum`/`Count`.
5. **Token authentication for the API:** Add `rest_framework.authtoken` to `INSTALLED_APPS`, generate tokens for existing users, and switch `OrderViewSet`/`ProduceViewSet` to use `TokenAuthentication` so the Android app can authenticate with a header instead of session cookies.

### Django Project Complete 🎉

You have built KhetiSetu's core marketplace backend: models for farmers, produce, and orders; a Django admin for the operations team; class-based views and templates for the public site; a `ModelForm` with custom validation for listing produce; authentication that scopes every query to the logged-in user; and a full REST API with Django REST Framework that KhetiSetu's Android app calls directly. This is the same architecture pattern used by production Django marketplaces today.

> **Devendra**
>
> "Ishaan, we onboarded 340 farmers in Nashik district this month, and the mobile app orders are already 60% of total volume — all hitting the API you built in Phase 6. The `get_queryset()` scoping you added after the bug bash hasn't leaked a single row since."

> **Sunita**
>
> "What I want you to remember from this project isn't the syntax — it's the discipline. Models first, then admin, then views, then API. And never trust an ID in a URL or a request body without checking it belongs to the logged-in user. That one habit will save you from the most common security bug in web development."

> **Next: Testing, Deployment & Performance for Django**

> - `django.test.TestCase` — write unit tests for models, views, and the API using Django's test client and DRF's `APITestCase`
> - `select_related` / `prefetch_related` at scale — eliminate N+1 queries as KhetiSetu's listing count grows into the tens of thousands
> - Environment-based settings — split `settings.py` into base/dev/prod, move secrets to environment variables
> - Deploying with Gunicorn + Nginx, and running `collectstatic` for production static files
> - Celery + Redis — move slow work (SMS notifications to farmers on new orders) off the request/response cycle
