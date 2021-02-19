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

## Generalized class for multi-model history APIs

There was a use case where I needed to show tables of data from different modeis ordered by time, and this behavior was repetitive. I build a generalized history class that joins the childrens' classes and makes use of variable serializers to rapidly allow the building of multi-model history routes.

General class:

```python
class GenericHistoryView(generics.ListAPIView):
    permission_classes = (IsAuthenticated, IsValidToAccessOffice, HasViewerPermission)
    authentication_classes = (BearerToken,)
    pagination_class = StandardPagination

    def get_chained_querysets(self):
        # Python doesn't have syntax for mandatory methods to be implemented by child classes, so I force its implementation by
        # raising an exception on the parent class by default. It only stops raising it when the child class implements it.
        raise NotImplementedError()

    def get_queryset(self):
        return sorted(self.get_chained_querysets(), key=lambda item: item.created, reverse=True)
```

Example of child class:

```python
class SensorHistoryView(GenericHistoryView):
    serializer_class = SensorHistorySerializer

    def get_chained_querysets(self):
        sensor_actions = SensorAction.objects.filter(irpd=self.kwargs["pk"])
        sensor_streams = SensorStream.objects.filter(irpd=self.kwargs["pk"])
        sensor_streams_v2 = SensorStreamv2.objects.filter(irpd=self.kwargs["pk"], message_subtype="event").distinct("uuid").order_by("uuid", "created", "arrived")
        sensor_actions_v2 = SensorActionv2.objects.filter(irpd=self.kwargs["pk"]).distinct("uuid").order_by("uuid", "created", "arrived")

        if "date_start" in self.request.query_params and "date_end" in self.request.query_params:
            date_start = dateutil.parser.parse(self.request.query_params["date_start"])
            date_end = dateutil.parser.parse(self.request.query_params["date_end"])

            sensor_actions = sensor_actions.filter(created__gte=date_start, created__lte=date_end)
            sensor_streams = sensor_streams.filter(created__gte=date_start, created__lte=date_end)
            sensor_streams_v2 = sensor_streams_v2.filter(created__gte=date_start, created__lte=date_end)
            sensor_actions_v2 = sensor_actions_v2.filter(created__gte=date_start, created__lte=date_end)

        return chain(sensor_actions, sensor_streams, sensor_streams_v2, sensor_actions_v2)
```

And there was logic on the serializer as well:

```python
class SensorHistorySerializer(serializers.BaseSerializer):
    def to_representation(self, instance):
        if isinstance(instance, SensorStream):
            return {"irpd_stream": SensorStreamSerializer(instance=instance).data}
        if isinstance(instance, SensorAction):
            return{"irpd_action": SensorActionSerializer(instance=instance).data}
        if isinstance(instance, SensorActionV2):
            return{"irpd_action_v2": SensorActionSerializerV2(instance=instance).data}
        if isinstance(instance, SensorStreamV2):
            return {"irpd_stream_v2": SensorStreamSerializerV2(instance=instance).data}
        return None
```

## Wrap up

Even though these code snippets aren't a full project, I believe they show I not only know Django, but I go beyond and create new architectures and constructs to achieve code reusability, clarity and features unavailable in the original framework toolset. 
