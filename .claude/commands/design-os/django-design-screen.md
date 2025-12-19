# Django Design Screen

You are helping the user create a screen design for a section of their Django product. The screen design will be Django views, templates, and forms that can be exported and integrated into any Django project.

## Step 1: Check Prerequisites

First, identify the target section and verify that `spec.md`, `models.py`, and `fixtures.json` exist.

Read `/product/product-roadmap.md` to get the list of available sections.

If there's only one section, auto-select it. If there are multiple sections, use the AskUserQuestion tool to ask which section the user wants to create a screen design for.

Then verify all required files exist:

- `product/sections/[section-id]/spec.md`
- `product/sections/[section-id]/models.py`
- `product/sections/[section-id]/fixtures.json`

If spec.md doesn't exist:

"I don't see a specification for **[Section Title]** yet. Please run `/shape-section` first to define the section's requirements."

If models.py or fixtures.json don't exist:

"I don't see sample data for **[Section Title]** yet. Please run `/django-sample-data` first to create models and fixtures for the screen designs."

Stop here if any file is missing.

## Step 2: Check for Design System

Check for optional enhancements:

**Design Tokens:**
- Check if `/product/design-system/colors.json` exists
- Check if `/product/design-system/typography.json` exists

If design tokens exist, read them and use them for template styling. If they don't exist, show a warning:

"Note: Design tokens haven't been defined yet. I'll use default styling, but for consistent branding, consider running `/design-tokens` first."

## Step 3: Analyze Requirements

Read and analyze all files:

1. **spec.md** - Understand the user flows and UI requirements
2. **models.py** - Understand the data structure
3. **fixtures.json** - Understand the sample content

Identify what views are needed based on the spec. Common patterns:

- List view (showing multiple items)
- Detail view (showing a single item)
- Create view (form for adding new items)
- Update view (form for editing existing items)
- Delete view (confirmation for removing items)

## Step 4: Clarify the Screen Design Scope

If the spec implies multiple views, use the AskUserQuestion tool to confirm which view to build first:

"The specification suggests a few different views for **[Section Title]**:

1. **[View 1]** - [Brief description]
2. **[View 2]** - [Brief description]

Which view should I create first?"

If there's only one obvious view, proceed directly.

## Step 5: Invoke the Frontend Design Skill

Before creating the screen design, read the `frontend-design` skill to ensure high-quality design output.

Read the file at `.claude/skills/frontend-design/SKILL.md` and follow its guidance for creating distinctive, production-grade interfaces.

## Step 6: Create the View

Create the view file at `src/sections/[section-id]/views/[view_name].py`.

### View Structure (Function-Based)

```python
"""
Views for [Section Title]
"""

from django.shortcuts import render, get_object_or_404, redirect
from django.contrib import messages

from .models import Invoice
from .forms import InvoiceForm


def invoice_list(request):
    """Display all invoices for the current user."""
    invoices = Invoice.objects.all()

    # Optional: filtering
    status = request.GET.get('status')
    if status:
        invoices = invoices.filter(status=status)

    return render(request, 'invoices/invoice_list.html', {
        'invoices': invoices,
        'status_choices': Invoice._meta.get_field('status').choices,
        'current_status': status,
    })


def invoice_detail(request, pk):
    """Display a single invoice."""
    invoice = get_object_or_404(Invoice, pk=pk)
    return render(request, 'invoices/invoice_detail.html', {
        'invoice': invoice,
    })


def invoice_create(request):
    """Create a new invoice."""
    if request.method == 'POST':
        form = InvoiceForm(request.POST)
        if form.is_valid():
            invoice = form.save()
            messages.success(request, 'Invoice created successfully.')
            return redirect('invoice_detail', pk=invoice.pk)
    else:
        form = InvoiceForm()

    return render(request, 'invoices/invoice_form.html', {
        'form': form,
        'title': 'Create Invoice',
    })


def invoice_update(request, pk):
    """Update an existing invoice."""
    invoice = get_object_or_404(Invoice, pk=pk)

    if request.method == 'POST':
        form = InvoiceForm(request.POST, instance=invoice)
        if form.is_valid():
            form.save()
            messages.success(request, 'Invoice updated successfully.')
            return redirect('invoice_detail', pk=invoice.pk)
    else:
        form = InvoiceForm(instance=invoice)

    return render(request, 'invoices/invoice_form.html', {
        'form': form,
        'invoice': invoice,
        'title': 'Edit Invoice',
    })


def invoice_delete(request, pk):
    """Delete an invoice."""
    invoice = get_object_or_404(Invoice, pk=pk)

    if request.method == 'POST':
        invoice.delete()
        messages.success(request, 'Invoice deleted.')
        return redirect('invoice_list')

    return render(request, 'invoices/invoice_confirm_delete.html', {
        'invoice': invoice,
    })
```

### View Structure (Class-Based)

If the user prefers CBVs:

```python
"""
Views for [Section Title]
"""

from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView
from django.urls import reverse_lazy
from django.contrib.messages.views import SuccessMessageMixin

from .models import Invoice
from .forms import InvoiceForm


class InvoiceListView(ListView):
    model = Invoice
    template_name = 'invoices/invoice_list.html'
    context_object_name = 'invoices'
    paginate_by = 25


class InvoiceDetailView(DetailView):
    model = Invoice
    template_name = 'invoices/invoice_detail.html'
    context_object_name = 'invoice'


class InvoiceCreateView(SuccessMessageMixin, CreateView):
    model = Invoice
    form_class = InvoiceForm
    template_name = 'invoices/invoice_form.html'
    success_message = 'Invoice created successfully.'

    def get_success_url(self):
        return reverse_lazy('invoice_detail', kwargs={'pk': self.object.pk})


class InvoiceUpdateView(SuccessMessageMixin, UpdateView):
    model = Invoice
    form_class = InvoiceForm
    template_name = 'invoices/invoice_form.html'
    success_message = 'Invoice updated successfully.'

    def get_success_url(self):
        return reverse_lazy('invoice_detail', kwargs={'pk': self.object.pk})


class InvoiceDeleteView(DeleteView):
    model = Invoice
    template_name = 'invoices/invoice_confirm_delete.html'
    success_url = reverse_lazy('invoice_list')
```

## Step 7: Create the Form

Create forms at `src/sections/[section-id]/forms.py`:

```python
"""
Forms for [Section Title]
"""

from django import forms

from .models import Invoice, LineItem


class InvoiceForm(forms.ModelForm):
    class Meta:
        model = Invoice
        fields = ['client_name', 'client_email', 'due_date', 'status']
        widgets = {
            'due_date': forms.DateInput(attrs={'type': 'date'}),
        }


class LineItemForm(forms.ModelForm):
    class Meta:
        model = LineItem
        fields = ['description', 'quantity', 'rate']


# For inline editing of line items with an invoice
LineItemFormSet = forms.inlineformset_factory(
    Invoice,
    LineItem,
    form=LineItemForm,
    extra=1,
    can_delete=True,
)
```

## Step 8: Create the Templates

Create templates at `src/sections/[section-id]/templates/[section]/`.

### Base/Layout Integration

Templates should extend a base template. Create a note about the expected base:

```html
{#
  Expected base template: base.html

  Required blocks:
  - title: Page title
  - content: Main page content

  Available context from base:
  - user: Current user (if authenticated)
  - messages: Django messages framework
#}
```

### List Template

`src/sections/[section-id]/templates/invoices/invoice_list.html`:

```html
{% extends "base.html" %}

{% block title %}Invoices{% endblock %}

{% block content %}
<div class="max-w-6xl mx-auto px-4 py-8">
  <!-- Header -->
  <div class="flex items-center justify-between mb-8">
    <h1 class="text-2xl font-bold text-stone-900 dark:text-stone-100">
      Invoices
    </h1>
    <a
      href="{% url 'invoice_create' %}"
      class="inline-flex items-center px-4 py-2 bg-lime-600 hover:bg-lime-700 text-white font-medium rounded-lg transition-colors"
    >
      <svg class="w-5 h-5 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 4v16m8-8H4"/>
      </svg>
      New Invoice
    </a>
  </div>

  <!-- Filters -->
  <div class="mb-6">
    <form method="get" class="flex gap-4">
      <select
        name="status"
        class="px-3 py-2 border border-stone-300 dark:border-stone-600 rounded-lg bg-white dark:bg-stone-800"
        onchange="this.form.submit()"
      >
        <option value="">All Statuses</option>
        {% for value, label in status_choices %}
          <option value="{{ value }}" {% if current_status == value %}selected{% endif %}>
            {{ label }}
          </option>
        {% endfor %}
      </select>
    </form>
  </div>

  <!-- Invoice List -->
  <div class="bg-white dark:bg-stone-800 rounded-xl shadow-sm border border-stone-200 dark:border-stone-700 overflow-hidden">
    {% if invoices %}
      <ul class="divide-y divide-stone-200 dark:divide-stone-700">
        {% for invoice in invoices %}
          <li>
            <a
              href="{% url 'invoice_detail' invoice.pk %}"
              class="block px-6 py-4 hover:bg-stone-50 dark:hover:bg-stone-700/50 transition-colors"
            >
              <div class="flex items-center justify-between">
                <div>
                  <p class="font-medium text-stone-900 dark:text-stone-100">
                    {{ invoice.client_name }}
                  </p>
                  <p class="text-sm text-stone-500 dark:text-stone-400">
                    {{ invoice.invoice_number }}
                  </p>
                </div>
                <div class="text-right">
                  <p class="font-medium text-stone-900 dark:text-stone-100">
                    ${{ invoice.total }}
                  </p>
                  <span class="inline-flex items-center px-2 py-1 text-xs font-medium rounded-full
                    {% if invoice.status == 'paid' %}
                      bg-green-100 text-green-800 dark:bg-green-900/30 dark:text-green-400
                    {% elif invoice.status == 'overdue' %}
                      bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-400
                    {% elif invoice.status == 'sent' %}
                      bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-400
                    {% else %}
                      bg-stone-100 text-stone-800 dark:bg-stone-700 dark:text-stone-300
                    {% endif %}
                  ">
                    {{ invoice.get_status_display }}
                  </span>
                </div>
              </div>
            </a>
          </li>
        {% endfor %}
      </ul>
    {% else %}
      <div class="px-6 py-12 text-center">
        <p class="text-stone-500 dark:text-stone-400">No invoices yet.</p>
        <a
          href="{% url 'invoice_create' %}"
          class="inline-flex items-center mt-4 text-lime-600 hover:text-lime-700 font-medium"
        >
          Create your first invoice
        </a>
      </div>
    {% endif %}
  </div>
</div>
{% endblock %}
```

### Detail Template

`src/sections/[section-id]/templates/invoices/invoice_detail.html`:

```html
{% extends "base.html" %}

{% block title %}{{ invoice.invoice_number }}{% endblock %}

{% block content %}
<div class="max-w-4xl mx-auto px-4 py-8">
  <!-- Header -->
  <div class="flex items-center justify-between mb-8">
    <div>
      <a
        href="{% url 'invoice_list' %}"
        class="text-sm text-stone-500 hover:text-stone-700 dark:text-stone-400 dark:hover:text-stone-200"
      >
        ← Back to Invoices
      </a>
      <h1 class="text-2xl font-bold text-stone-900 dark:text-stone-100 mt-2">
        {{ invoice.invoice_number }}
      </h1>
    </div>
    <div class="flex gap-2">
      <a
        href="{% url 'invoice_update' invoice.pk %}"
        class="px-4 py-2 border border-stone-300 dark:border-stone-600 rounded-lg hover:bg-stone-50 dark:hover:bg-stone-700 transition-colors"
      >
        Edit
      </a>
      <a
        href="{% url 'invoice_delete' invoice.pk %}"
        class="px-4 py-2 text-red-600 border border-red-300 rounded-lg hover:bg-red-50 dark:hover:bg-red-900/20 transition-colors"
      >
        Delete
      </a>
    </div>
  </div>

  <!-- Invoice Card -->
  <div class="bg-white dark:bg-stone-800 rounded-xl shadow-sm border border-stone-200 dark:border-stone-700 p-6">
    <dl class="grid grid-cols-2 gap-6">
      <div>
        <dt class="text-sm text-stone-500 dark:text-stone-400">Client</dt>
        <dd class="mt-1 font-medium text-stone-900 dark:text-stone-100">{{ invoice.client_name }}</dd>
      </div>
      <div>
        <dt class="text-sm text-stone-500 dark:text-stone-400">Email</dt>
        <dd class="mt-1 text-stone-900 dark:text-stone-100">{{ invoice.client_email }}</dd>
      </div>
      <div>
        <dt class="text-sm text-stone-500 dark:text-stone-400">Amount</dt>
        <dd class="mt-1 text-2xl font-bold text-stone-900 dark:text-stone-100">${{ invoice.total }}</dd>
      </div>
      <div>
        <dt class="text-sm text-stone-500 dark:text-stone-400">Status</dt>
        <dd class="mt-1">
          <span class="inline-flex items-center px-3 py-1 text-sm font-medium rounded-full
            {% if invoice.status == 'paid' %}
              bg-green-100 text-green-800 dark:bg-green-900/30 dark:text-green-400
            {% elif invoice.status == 'overdue' %}
              bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-400
            {% else %}
              bg-stone-100 text-stone-800 dark:bg-stone-700 dark:text-stone-300
            {% endif %}
          ">
            {{ invoice.get_status_display }}
          </span>
        </dd>
      </div>
      <div>
        <dt class="text-sm text-stone-500 dark:text-stone-400">Due Date</dt>
        <dd class="mt-1 text-stone-900 dark:text-stone-100">{{ invoice.due_date|date:"F j, Y" }}</dd>
      </div>
    </dl>

    <!-- Line Items -->
    {% if invoice.line_items.exists %}
      <div class="mt-8 border-t border-stone-200 dark:border-stone-700 pt-6">
        <h2 class="font-medium text-stone-900 dark:text-stone-100 mb-4">Line Items</h2>
        <table class="w-full">
          <thead>
            <tr class="text-left text-sm text-stone-500 dark:text-stone-400">
              <th class="pb-2">Description</th>
              <th class="pb-2 text-right">Qty</th>
              <th class="pb-2 text-right">Rate</th>
              <th class="pb-2 text-right">Amount</th>
            </tr>
          </thead>
          <tbody class="divide-y divide-stone-100 dark:divide-stone-700">
            {% for item in invoice.line_items.all %}
              <tr>
                <td class="py-2 text-stone-900 dark:text-stone-100">{{ item.description }}</td>
                <td class="py-2 text-right text-stone-600 dark:text-stone-300">{{ item.quantity }}</td>
                <td class="py-2 text-right text-stone-600 dark:text-stone-300">${{ item.rate }}</td>
                <td class="py-2 text-right font-medium text-stone-900 dark:text-stone-100">${{ item.amount }}</td>
              </tr>
            {% endfor %}
          </tbody>
        </table>
      </div>
    {% endif %}
  </div>
</div>
{% endblock %}
```

### Form Template

`src/sections/[section-id]/templates/invoices/invoice_form.html`:

```html
{% extends "base.html" %}

{% block title %}{{ title }}{% endblock %}

{% block content %}
<div class="max-w-2xl mx-auto px-4 py-8">
  <h1 class="text-2xl font-bold text-stone-900 dark:text-stone-100 mb-8">
    {{ title }}
  </h1>

  <form method="post" class="space-y-6">
    {% csrf_token %}

    {% for field in form %}
      <div>
        <label
          for="{{ field.id_for_label }}"
          class="block text-sm font-medium text-stone-700 dark:text-stone-300 mb-1"
        >
          {{ field.label }}
        </label>
        {{ field }}
        {% if field.errors %}
          <p class="mt-1 text-sm text-red-600">{{ field.errors.0 }}</p>
        {% endif %}
        {% if field.help_text %}
          <p class="mt-1 text-sm text-stone-500">{{ field.help_text }}</p>
        {% endif %}
      </div>
    {% endfor %}

    <div class="flex gap-4 pt-4">
      <button
        type="submit"
        class="px-6 py-2 bg-lime-600 hover:bg-lime-700 text-white font-medium rounded-lg transition-colors"
      >
        {% if invoice %}Update{% else %}Create{% endif %} Invoice
      </button>
      <a
        href="{% if invoice %}{% url 'invoice_detail' invoice.pk %}{% else %}{% url 'invoice_list' %}{% endif %}"
        class="px-6 py-2 border border-stone-300 dark:border-stone-600 rounded-lg hover:bg-stone-50 dark:hover:bg-stone-700 transition-colors"
      >
        Cancel
      </a>
    </div>
  </form>
</div>
{% endblock %}
```

## Step 9: Create URL Configuration

Create `src/sections/[section-id]/urls.py`:

```python
"""
URL configuration for [Section Title]
"""

from django.urls import path

from . import views

urlpatterns = [
    path('', views.invoice_list, name='invoice_list'),
    path('create/', views.invoice_create, name='invoice_create'),
    path('<int:pk>/', views.invoice_detail, name='invoice_detail'),
    path('<int:pk>/edit/', views.invoice_update, name='invoice_update'),
    path('<int:pk>/delete/', views.invoice_delete, name='invoice_delete'),
]
```

## Step 10: Confirm and Next Steps

Let the user know:

"I've created the screen design for **[Section Title]**:

**Files created:**

- `src/sections/[section-id]/views/[view_name].py` - Django views
- `src/sections/[section-id]/forms.py` - Django forms
- `src/sections/[section-id]/urls.py` - URL configuration
- `src/sections/[section-id]/templates/[section]/` - HTML templates

**Views implemented:**

- List view with filtering
- Detail view
- Create/Update forms
- Delete confirmation

**Design tokens applied:** [If tokens exist, mention colors used]

**Next steps:**

- Run `/screenshot-design` to capture a screenshot for documentation
- If the spec calls for additional views, run `/django-design-screen` again
- When all sections are complete, run `/export-product` to generate the export package"

## Design Requirements

- **Mobile responsive:** Use Tailwind responsive prefixes (`sm:`, `md:`, `lg:`)
- **Light & dark mode:** Use `dark:` variants for all colors
- **Use design tokens:** If defined, apply the product's color palette
- **Follow the frontend-design skill:** Create distinctive, memorable interfaces

### Applying Design Tokens

**If `/product/design-system/colors.json` exists:**
- Use the primary color for buttons and key accents (e.g., `bg-lime-600`)
- Use the secondary color for tags, highlights
- Use the neutral color for backgrounds, text, borders

**If design tokens don't exist:**
- Fall back to `stone` for neutrals and `lime` for accents

## Important Notes

- Templates use Tailwind CSS for styling
- All templates extend a `base.html` (document expected blocks)
- Forms should include `{% csrf_token %}`
- Use Django's messages framework for feedback
- Include both light and dark mode styles
- Views should be portable - no project-specific imports
