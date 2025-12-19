# Django Design API

You are helping the user create an API design for a section of their Django product. The API will use Django REST Framework (DRF) with serializers and viewsets that can be exported and integrated into any Django project.

## Step 1: Check Prerequisites

First, identify the target section and verify that `spec.md` and `models.py` exist.

Read `/product/product-roadmap.md` to get the list of available sections.

If there's only one section, auto-select it. If there are multiple sections, use the AskUserQuestion tool to ask which section the user wants to create an API for.

Then verify required files exist:

- `product/sections/[section-id]/spec.md`
- `product/sections/[section-id]/models.py`

If spec.md doesn't exist:

"I don't see a specification for **[Section Title]** yet. Please run `/shape-section` first to define the section's requirements."

If models.py doesn't exist:

"I don't see models for **[Section Title]** yet. Please run `/django-sample-data` first to create models for the API."

Stop here if any file is missing.

## Step 2: Analyze Requirements

Read and analyze:

1. **spec.md** - Understand the user flows and actions
2. **models.py** - Understand the data structure

Map user flows to API endpoints:

| User Flow | HTTP Method | Endpoint |
|-----------|-------------|----------|
| View list | GET | `/api/invoices/` |
| View detail | GET | `/api/invoices/{id}/` |
| Create | POST | `/api/invoices/` |
| Update | PUT/PATCH | `/api/invoices/{id}/` |
| Delete | DELETE | `/api/invoices/{id}/` |
| Custom action | POST | `/api/invoices/{id}/send/` |

## Step 3: Clarify API Scope

Use the AskUserQuestion tool to confirm:

"For **[Section Title]**, I can create:

1. **Full CRUD API** - List, retrieve, create, update, delete
2. **Read-only API** - Just list and retrieve
3. **Custom** - Specific endpoints only

Which approach? Also, any custom actions beyond CRUD? (e.g., 'send invoice', 'mark paid')"

## Step 4: Create Serializers

Create `src/sections/[section-id]/serializers.py`:

```python
"""
DRF Serializers for [Section Title]
"""

from rest_framework import serializers

from .models import Invoice, LineItem


class LineItemSerializer(serializers.ModelSerializer):
    """Serializer for line items (nested in invoice)."""

    amount = serializers.DecimalField(
        max_digits=10,
        decimal_places=2,
        read_only=True,
    )

    class Meta:
        model = LineItem
        fields = ['id', 'description', 'quantity', 'rate', 'amount']


class InvoiceListSerializer(serializers.ModelSerializer):
    """Lightweight serializer for invoice lists."""

    status_display = serializers.CharField(
        source='get_status_display',
        read_only=True,
    )

    class Meta:
        model = Invoice
        fields = [
            'id',
            'invoice_number',
            'client_name',
            'total',
            'status',
            'status_display',
            'due_date',
        ]


class InvoiceDetailSerializer(serializers.ModelSerializer):
    """Full serializer for invoice detail/create/update."""

    line_items = LineItemSerializer(many=True, read_only=True)
    status_display = serializers.CharField(
        source='get_status_display',
        read_only=True,
    )

    class Meta:
        model = Invoice
        fields = [
            'id',
            'invoice_number',
            'client_name',
            'client_email',
            'total',
            'status',
            'status_display',
            'due_date',
            'created_at',
            'updated_at',
            'line_items',
        ]
        read_only_fields = ['id', 'invoice_number', 'created_at', 'updated_at']


class InvoiceCreateSerializer(serializers.ModelSerializer):
    """Serializer for creating invoices with nested line items."""

    line_items = LineItemSerializer(many=True)

    class Meta:
        model = Invoice
        fields = [
            'client_name',
            'client_email',
            'due_date',
            'line_items',
        ]

    def create(self, validated_data):
        line_items_data = validated_data.pop('line_items')
        invoice = Invoice.objects.create(**validated_data)

        for item_data in line_items_data:
            LineItem.objects.create(invoice=invoice, **item_data)

        # Calculate total
        invoice.total = sum(item.amount for item in invoice.line_items.all())
        invoice.save()

        return invoice
```

### Serializer Patterns

1. **List vs Detail serializers** - Keep list responses lightweight
2. **Nested serializers** - For related objects
3. **Read-only computed fields** - Use `SerializerMethodField` or `source`
4. **Write serializers** - Handle nested creates/updates explicitly

## Step 5: Create ViewSets

Create `src/sections/[section-id]/views.py`:

```python
"""
DRF ViewSets for [Section Title]
"""

from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from django_filters.rest_framework import DjangoFilterBackend

from .models import Invoice
from .serializers import (
    InvoiceListSerializer,
    InvoiceDetailSerializer,
    InvoiceCreateSerializer,
)


class InvoiceViewSet(viewsets.ModelViewSet):
    """
    API endpoint for invoices.

    list: Get all invoices
    retrieve: Get a single invoice
    create: Create a new invoice
    update: Update an invoice
    partial_update: Partially update an invoice
    destroy: Delete an invoice
    """

    queryset = Invoice.objects.all()
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend]
    filterset_fields = ['status']

    def get_serializer_class(self):
        if self.action == 'list':
            return InvoiceListSerializer
        if self.action == 'create':
            return InvoiceCreateSerializer
        return InvoiceDetailSerializer

    def get_queryset(self):
        """Filter to user's own invoices."""
        return Invoice.objects.filter(created_by=self.request.user)

    def perform_create(self, serializer):
        """Set the creator on save."""
        serializer.save(created_by=self.request.user)

    # Custom actions

    @action(detail=True, methods=['post'])
    def send(self, request, pk=None):
        """Send the invoice to the client."""
        invoice = self.get_object()

        if invoice.status != 'draft':
            return Response(
                {'error': 'Only draft invoices can be sent.'},
                status=status.HTTP_400_BAD_REQUEST,
            )

        invoice.status = 'sent'
        invoice.save()

        # TODO: Send email to client

        return Response({'status': 'Invoice sent'})

    @action(detail=True, methods=['post'])
    def mark_paid(self, request, pk=None):
        """Mark the invoice as paid."""
        invoice = self.get_object()

        if invoice.status not in ('sent', 'overdue'):
            return Response(
                {'error': 'Only sent or overdue invoices can be marked paid.'},
                status=status.HTTP_400_BAD_REQUEST,
            )

        invoice.status = 'paid'
        invoice.save()

        return Response({'status': 'Invoice marked as paid'})
```

### ViewSet Patterns

1. **Use ModelViewSet** for full CRUD
2. **Override `get_serializer_class`** for different actions
3. **Override `get_queryset`** for filtering (e.g., user's own data)
4. **Use `@action` decorator** for custom endpoints
5. **Add permissions** appropriate to your auth model

## Step 6: Create URL Configuration

Create `src/sections/[section-id]/urls.py`:

```python
"""
API URL configuration for [Section Title]
"""

from django.urls import path, include
from rest_framework.routers import DefaultRouter

from .views import InvoiceViewSet

router = DefaultRouter()
router.register(r'invoices', InvoiceViewSet, basename='invoice')

urlpatterns = [
    path('', include(router.urls)),
]
```

## Step 7: Document the API

Create `src/sections/[section-id]/api.md`:

```markdown
# [Section Title] API

## Endpoints

### List Invoices

```
GET /api/invoices/
```

Query parameters:
- `status` - Filter by status (draft, sent, paid, overdue)

Response:
```json
[
  {
    "id": 1,
    "invoice_number": "INV-2024-001",
    "client_name": "Acme Corp",
    "total": "1500.00",
    "status": "sent",
    "status_display": "Sent",
    "due_date": "2024-02-15"
  }
]
```

### Get Invoice

```
GET /api/invoices/{id}/
```

Response:
```json
{
  "id": 1,
  "invoice_number": "INV-2024-001",
  "client_name": "Acme Corp",
  "client_email": "billing@acme.com",
  "total": "1500.00",
  "status": "sent",
  "status_display": "Sent",
  "due_date": "2024-02-15",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z",
  "line_items": [
    {
      "id": 1,
      "description": "Web Design",
      "quantity": 1,
      "rate": "1500.00",
      "amount": "1500.00"
    }
  ]
}
```

### Create Invoice

```
POST /api/invoices/
```

Request:
```json
{
  "client_name": "Acme Corp",
  "client_email": "billing@acme.com",
  "due_date": "2024-02-15",
  "line_items": [
    {
      "description": "Web Design",
      "quantity": 1,
      "rate": "1500.00"
    }
  ]
}
```

### Update Invoice

```
PUT /api/invoices/{id}/
PATCH /api/invoices/{id}/
```

### Delete Invoice

```
DELETE /api/invoices/{id}/
```

### Send Invoice

```
POST /api/invoices/{id}/send/
```

Only works on draft invoices. Transitions status to "sent".

### Mark Invoice Paid

```
POST /api/invoices/{id}/mark_paid/
```

Only works on sent or overdue invoices. Transitions status to "paid".

## Authentication

All endpoints require authentication. Include the auth token in the header:

```
Authorization: Token <your-token>
```

## Permissions

- Users can only access their own invoices
- Custom actions enforce status-based rules
```

## Step 8: Confirm and Next Steps

Let the user know:

"I've created the API design for **[Section Title]**:

**Files created:**

- `src/sections/[section-id]/serializers.py` - DRF serializers
- `src/sections/[section-id]/views.py` - DRF viewsets
- `src/sections/[section-id]/urls.py` - Router configuration
- `src/sections/[section-id]/api.md` - API documentation

**Endpoints:**

- `GET /api/[section]/` - List
- `GET /api/[section]/{id}/` - Detail
- `POST /api/[section]/` - Create
- `PUT/PATCH /api/[section]/{id}/` - Update
- `DELETE /api/[section]/{id}/` - Delete
- [Custom actions listed]

**Next steps:**

- If you're building a frontend, you can use the existing React commands (`/design-screen`)
- If you need more custom actions, run this command again
- When all sections are complete, run `/export-product` to generate the export package"

## Important Notes

- Use ModelSerializer for standard CRUD operations
- Create separate serializers for list vs detail views
- Use `@action` decorator for custom endpoints beyond CRUD
- Document all endpoints in api.md
- Include proper permissions for your auth model
- Filter querysets to user's own data when appropriate
