django-earthdistance
====================

Using PostgreSQL's EarthDistance extension with django and django-ext-*

Earthdistance allows to do fast geolocalized queries without using PostGIS

Usage
---------------

Cube and EarthDistance extensions must be enabled in postgreSQL BD, so logs in
database using pgsql and install extensions:

```
=> create extension cube;
=> create extension earthdistance;
```

Extension is built using [djorm-ext-core](https://github.com/niwibe/djorm-ext-core)
and [djorm-ext-expressions](https://github.com/niwibe/djorm-ext-expressions) packages.

### Filter by rows inside a circle of distance r ###

```python
from django_earthdistance.expressions import DistanceExpression
from django_earthdistance.functions import CubeDistance, LlToEarth
from djorm_expressions.models import ExpressionManager

class MyModel(models.Model):
    latitute = models.FloatField()
    longitude = models.FloatField()

    objects = ExpressionManager()

# Define fields to query in DistanceExpression initialization
# search with lat=0.2546 and lon=-38.25 and distance 1500 meters

MyModel.objects.where(
    DistanceExpression(['latitude', 'longitude']).in_distance(
        1500,
        [0.2546, -38.25]))
```

### Annotate each row returned by a query with distance between two points ###
```python
# annotate_functions can be used with any manager inherited from ExpressionManager
# see https://github.com/niwibe/djorm-ext-expressions and 
# http://www.niwi.be/2012/10/07/sqlexpressions-and-functions-with-django/

MyModel.objects.filter(....).annotate_functions(
    distance=CubeDistance(
        LlToEarth([0.2546, -38.25]]),
        LlToEarth(['latitude', 'longitude']))
```

### Optimizing perfomance with indexes ###

PostgreSQL allow to use GiST indexes with functions results, a good perfomance improvement is to store ll_to_earth results in
an index, ll_to_earth() calculates the position of a point on the surface of the earth (assuming earth is 
perfectly spherical)


```sql
-- Example MyModel table is app_mymodel and points columns are latitude and longitude
CREATE INDEX mymodel_location ON app_mymodel USING gist (ll_to_earth(latitude, longitude));
```

Also, using south is preferred, just add this migration to migrations/ folder and edit it to your needs
```python
class Migration(SchemaMigration):

    def forwards(self, orm):
        cursor = connection.cursor()
        cursor.execute("CREATE INDEX mymodel_location ON app_mymodel USING gist (ll_to_earth(latitude, longitude));")


    def backwards(self, orm):
        # Deleting field 'Venue.coords'
        cursor = connection.cursor()
        cursor.execute("DROP INDEX mymodel_location ON app_mymodel;")
```
