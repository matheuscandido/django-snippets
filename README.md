# Django Snippets

Most Django projects I worked on are closed source. Since I can't share code I don't own, I'll exibit here some interesting code snippets that go beyond the ordinary Django skillset of Router, Models and Views that's expected from a back-end Python developer. I hope these are useful to assess my skill and mastery of the framework.

The code is inspired on code I wrote on closed-source projects

On some of these examples I make use of Django Rest Framework (DRF), which provides set of classes that speed up development of APIs, covering some common use-cases of Django views and serializers.

### Writing Generalized Views for Shared Behavior

The specific behavior can be expressed on child classes.

```python
class SensorListCreateView(GenericListCreateSensorViewWithPermissions):
    serializer_class = FullSensorSerializer


class SensorPaginatedListCreateView(GenericListCreateSensorViewWithPermissions):
    serializer_class = FullSensorSerializer
    pagination_class = DefaultListPagination


class SimplePivotListCreatePivotView(GenericListCreateSensorViewWithPermissions):
    serializer_class = SensorSimpleSerializer
```

While the generalized behavior is shared in a parent class:

```python
class GenericListCreateSensorViewWithPermissions(generics.ListCreateAPIView):
    permission_classes_by_method = {
        "GET": (IsAuthenticated, IsValidToAccessSensor,),
        "POST": (IsAuthenticated, IsValidToAccessSensor, IsReviewer)
    }
    authentication_classes = (BearerToken,)
    serializer_class = FullSensorSerializer
    model = Sensor
    model_name = "sensor"

    def get_permissions(self):
        # This is an innovation of mine, since the default DRF doesn't allow for different permission-sets based on HTTP method
        return [permission() for permission in self.permission_classes_by_method[self.request.method]]

    def get_queryset(self):
        office_id = self.kwargs['office']

        if self.request.user.is_superuser:
            return self.model.objects.filter(office=office_id).order_by('name')

        office_administrator = Office.objects.get(id=office_id).administrator
        if self.request.user == office_administrator:
            return self.model.objects.filter(office=office_id).order_by('name')

        access_permissions = AccessPermission.objects.filter(user=self.request.user, office__isnull=False).exclude(level=0).values(self.model_name.lower())
        return self.model.objects.filter(id__in=access_permissions, office=office_id).order_by('name')

    def post(self, request, *args, **kwargs):
        # some code here that adds to the POST cycle, overriding default behavior
        return super(GenericListCreateSensorViewWithPermissions, self).post(request, args, kwargs)
```

I make use of overriding default DRF classes extensively, to add functionality that is absent on the upstream feature set.
