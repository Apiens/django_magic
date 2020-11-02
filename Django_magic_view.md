

# DRF CBV



```
# Intended to make myself better understand magics behind django CBV.
# Summarized the cascading inheritance among
# View -> APIView -> GenericAPIView + Mixins -> Viewset
# Pseudocodes included.
```



## 1. View

```
# View.as_view() -> returns 'view'
# when 'view' is called, it calls dispatch()
# and dispatch() returns proper View.method() based on the http method used for request.

# in urls.py  : View.as_view() -> view
# by server   : view() -> dispatch() -> View.proper_method()
```





## 2. APIView

> Inherit View and add following features

```
1) Forced Caching
    When the APIView has .queryset attribute.
    Forbid objects.queryset() by overriding APIView.queryset._fetch_all()
    it enforce using either .all() or .get_queryset() which caches data

2) Policy Application
    classes for
        -renderer
        -parser
        -authentification
        -throttle
        -permission
        -content_negetiation
        -metadata
        -versioning
    is assgined as attributes with default api setting
    
    methods for policy instantiation and implementations are also included.
    e.g.
    -instantiation: get_permissions, get_throttles
    -implementation: check_permissions, check_throttles
```



## 3. GenericAPIView

> Inherit APIView. 

```
# Help querying and serialization through pre-designed attributes and methods.
# Assign `queryset` and `serializer_class` with proper Model and Serializer 
# Methods such as `get_object`, `get_serializer`, `filter_queryset` will help you handle Model and Serializer.
```



### 3.1 new attributes

```python
queryset = None
serializer_class = None

lookup_field = 'pk'
lookup_url_kwarg = None

filter_backends = api_settings.DEFAULT_FILTER_BACKENDS
pagination_class = api_settings.DEFAULT_PAGINATION_CLASS
```



### 3.2 new methods

```python

    def get_queryset():
        return self.queryset.all()
    def get_obejct():
        queryset = self.filter_queryset(self.get_queryset)
        filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
        obj = get_object_or_404(queryset, **filter_kwargs)
        return obj
    def get_serializer():
        serializer_class = get_serializer_class()
        kwargs.setdefault('context',get_serializer_context())
        return serializer_class
    def get_serialiser_class()
    def get_serializer_context()
    def filter_queryset():
        for backend in list(self.filter_backends):
            queryset = backend().filter_queryset(self.request, queryset, self)
        return queryset
    def paginator():
    def paginate_queryset():
    def get_paginated_response():
```



## 4. Mixins



```
#### Mixins + GenericAPIView => LC_APIView, RUD_APIView ####
# Basically there are 5 mixins: List, Create, Retreive, Update, Destroy.
# these mixins utilize methods defined in GenericAPIView.
# (such as get_object, get_serializer, filter_queryset and paginator.paginate_queryset)
# by inheriting Mixins and GenericAPIView, YourOwnAPIView will have much sexier outlook.
# ListCreateApiView, RetrieveUpdateDestroyApiView are good example of YourOwnAPIView

# Be aware http methods defined in the ListCreateApiView 
# and that they call inherited methods from Mixins.

# You are doing great job if you have looked at what 'get()' is calling in each APIView.
# In ListCreateApiView, get() returns list().
# In RetrieveUpdateDeleteView, get() returns retrieve().
```



### 4.1 Mixins

```python
class ListModelMixin:
    def list():
        queryset = self.filter_queryset(self.get_queryset)
        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)
        serializer = self.get_serializer(queryset,many=True)
        return Response(serializer.data)
    
class CreateModelMixin:
	def create():
        serializer = self.get_serializer()
        serializer.is_valid()
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)

    
class RetrieveModelMixin:
    def retrieve():
        instance = self.get_object()
        serializer = self.get_serializer(instance)
        return Response(serializer.data)
 
class UpdateModelMixin:
    def update():
        partial = kwargs.pop('partial',False)
        instance = self.get_object()
        serializer = self.get_serializer(instance, data=request.data, partial=partial)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        
        if getattr(instance, '_prefetched_objects_cache', None):
            # If 'prefetch_related' has been applied to queryset,
            # i.e. when we eagerloaded data (for whatever purpose),
            # we need to empty the cache.
            # (because we shouldn't update with cached data!)
            instance._prefetched_objects_cache = {}
        
        return Response(serializer.data)
        
    def partial_update(self, request, *args, **kwargs):
        kwargs['partial'] = True
        return self.update(self, request, *args, **kwargs)
    
class DestroyModelMixin:
    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        instance.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```



### 4.2 LC/RUD/APIView

```python
class ListCreateAPIView(mixins.ListModelMixin,
                        mixins.CreateModelMixin,
                        GenericAPIView):
    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
    
    
class RetrieveUpdateDestroyAPIView(mixins.RetrieveModelMixin,
                                   mixins.UpdateModelMixin,
                                   mixins.DestroyModelMixin,
                                   GenericAPIView):
    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def patch(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```





## 5. ViewSet and Router



