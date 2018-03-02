﻿Working with entity instances
================================

Creating an entity instance
-------------------------------------------

Creating an entity instance in Pony is similar to creating a regular object in Python::

   customer1 = Customer(login="John", password="***",
                        name="John", email="john@google.com")

When creating an object in Pony, all the parameters should be specified as keyword arguments. If an attribute has a default value, you can omit it.

All created instances belong to the current :ref:`database session <db_session_ref>`. In some object-relational mappers, you are required to call an object's ``save()`` method in order to save it. This is inconvenient, as a programmer must track which objects were created or updated, and must not forget to call the ``save()`` method on each object.

Pony tracks which objects were created or updated and saves them in the database automatically when current ``db_session`` is over. If you need to save newly created objects before leaving the ``db_session`` scope, you can do so by using :ref:`flush() <flush_ref>` or :ref:`commit() <commit_ref>` functions.


Loading objects from the database
-----------------------------------------------------

Getting an object by primary key
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The simplest case is when we want to retrieve an object by the primary key. To accomplish this in Pony, the user simply needs to put the primary key in square brackets, after the class name. For example, to extract a customer with the primary key value of 123, we can write::

   customer1 = Customer[123]

The same syntax also works for objects with composite keys; we just need to list the elements of the composite primary key, separated by commas, in the same order that the attributes were defined in the entity class description::

   order_item = OrderItem[order1, product1]

Pony raises the ``ObjectNotFound`` exception if object with such primary key doesn't exist.


Getting one object by unique combination of attributes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If we want to retrieve one object not by its primary key, but by another combination of attributes, we can use the ``get()`` method of an entity. In most cases, it is used when we want to get an object by the secondary unique key, but it can also be used to search by any other combination of attributes. As a parameter of the ``get()`` method, we state names of the attributes and their values. For example, if we want to receive a product under the name "Product 1", and we believe that database has only one product under this name, we can write::

   product1 = Product.get(name="Product1")

If no object is found, ``get()`` returns ``None``. If multiple objects are found, ``MultipleObjectsFoundError`` exception is raised.

We may want to use the ``get()`` method with primary key when we want to get ``None`` instead of ``ObjectNotFound`` exception if the object does not exists in database.

Method ``get()`` can also receive a lambda function as a single positioning argument, similar to the method ``select()``, which we will discuss later; but nevertheless, it will return an instance of an entity, and not an object of the ``Query`` type.


Getting several objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In order to retrieve several objects from a database, we use the ``select()`` method, which is present on every entity. Its argument is a lambda function, which carries a single parameter, symbolizing an instance of an object in the database. Inside this function, we can write conditions, by which we want to carry out the search. For example, if we want to find all products with the price higher than 100, we can write::

    products = Product.select(lambda p: p.price > 100)

This lambda function will not be executed in Python. Instead, it will be translated to a SQL query:

.. code-block:: sql

    SELECT "p"."id", "p"."name", "p"."description",
           "p"."picture", "p"."price", "p"."quantity"
    FROM "Product" "p"
    WHERE "p"."price" > 100

The ``select()`` method returns an instance of the :ref:`Query <query_object_ref>` class. If you start iterating over this object, the SQL query will be sent to the database and you will get the sequence of entity instances. For example, this is how we can print out all product names and prices for our products::

    for p in Product.select(lambda p: p.price > 100):
        print p.name, p.price

If we don't want to iterate over a query, but need just to get a list of objects, we can do it this way::

    product_list = Product.select(lambda p: p.price > 100)[:]

Here we get a full slice ``[:]`` from the query. This is an equivalent of converting a query to a list::

    product_list = list(Product.select(lambda p: p.price > 100))



Transfer parameters to the query
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Inside the lambda function, it is possible to use variables declared previously. In such cases, the query will put the values of those variables as a parameter. One important advantage of declarative query syntax in Pony is that it offers full protection from SQL-injections, as all the data related to external parameters is properly escaped.

For example, if we want to find products with the price higher than ``x``, then we can simply write::

    x = 100
    products = Product.select(lambda p: p.price > x)

The SQL query which will be generated will look this way:

.. code-block:: sql

    SELECT "p"."id", "p"."name", "p"."description",
           "p"."picture", "p"."price", "p"."quantity"
    FROM "Product" "p"
    WHERE "p"."price" > ?

This way the value of``x`` will be passed as the SQL query parameter, which completely eliminates the risk of SQL-injection.


Sorting query results
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If we need to sort objects in a certain order, we can use the method ``order_by()`` of the ``Query`` object. For example, if we want to display names and prices of all products with price higher than 100 in a descending order, we can do it this way::

    Product.select(lambda p: p.price > 100).order_by(desc(Product.price))

The methods of the ``Query`` object modify the SQL query which will be sent to the database. For the example above the following SQL will be generated:

.. code-block:: sql

    SELECT "p"."id", "p"."name", "p"."description",
           "p"."picture", "p"."price", "p"."quantity"
    FROM "Product" "p"
    WHERE "p"."price" > 100
    ORDER BY "p"."price" DESC

The ``order_by()`` method can also receive a lambda function as a parameter::

   Product.select(lambda p: p.price > 100).order_by(lambda p: desc(p.price))

Using the lambda function inside the ``order_by`` method allows using advanced sorting expressions. For example, this is how we can sort our customers by the total price of their orders in the descending order::

    Customer.select().order_by(lambda c: desc(sum(c.orders.total_price)))

In order to sort the result by several attributes, we need to separate them by a comma. For example, if we want to sort products by price in descending order, while displaying products with similar prices in alphabetical order, we can do it this way::

    Product.select(lambda p: p.price > 100).order_by(desc(Product.price), Product.name)

The same query, but using lambda function will look this way::

    Product.select(lambda p: p.price > 100).order_by(lambda p: (desc(p.price), p.name))

Note that according to Python syntax, if we return more than one element from lambda, we need to wrap them into the parenthesis.


Limiting the number of selected objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It is possible to limit the number of objects returned by a query by using the :py:meth:`limit()<Query.limit>` Query method, or by the more compact Python slice notation. For example, this is how we can get the 10 most expensive products::

    Product.select().order_by(lambda p: desc(p.price))[:10]

The result of a slice is not a query object, but a final list of entity instances.

You can also use the :py:meth:`Query.page` method as a convenient way of pagination the query results::

    Product.select().order_by(lambda p: desc(p.price)).page(1)


Traversing relationships
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In Pony you can traverse relationships easily::

    order = Order[123]
    customer = order.customer
    print customer.name

Pony tries to minimize the number of queries sent to the database. In the example above, if a requested ``Customer`` object was already loaded to the cache, Pony will return the object from the cache without sending a query to the database. But, if an object was not loaded yet, Pony still will not send a query immediately. Instead, it will create a "seed" object first. The seed is an object which has only the primary key initialized. Pony does not know how this object will be used, and there is always the possibility that only the primary key is needed.

In the example above, Pony get the object from database in the third line in, when we access the name attribute. By using the "seed" concept, Pony achieves high efficiency and solves the "N+1" problem, which is a weakness of many other mappers.

Traversing is possible in the "to-many" direction as well. For example, if we have a ``Customer`` object and we want to loop through its orders, we can do it this way::

    c = Customer[123]
    for order in c.orders:
        print order.state, order.price


Updating an object
-------------------------------------------------

When you assign new values to object attributes, you don't need to save each updated object manually. Changes will be saved in the database automatically on leaving the ``db_session`` scope.

For example, in order to increase the number of products by 10 with a primary key of 123, we can execute the following code::

    Product[123].quantity += 10

If we need to change several attributes of the same object, we can do so separately::

    order = Order[123]
    order.state = "Shipped"
    order.date_shipped = datetime.now()

or in a single line, using the ``set()`` method::

    order = Order[123]
    order.set(state="Shipped", date_shipped=datetime.now())

The ``set()`` method can be convenient when you need to update several object attributes at once from a dictionary::

    order.set(**dict_with_new_values)

If you need to save the updates to the database before the current database session is finished, you can use the :ref:`flush() <flush_ref>` or :ref:`commit() <commit_ref>` functions.

Pony always saves the changes accumulated in the ``db_session`` cache automatically before executing the following methods: ``select()``, ``get()``, ``exists()``, ``execute()`` and ``commit()``.

In future, Pony is going to support bulk update. It will allow updating multiple objects on the disk without loading them to the cache::

    update(p.set(price=price * 1.1) for p in Product
                                    if p.category.name == "T-Shirt")



Deleting an object
------------------------------------

When you call the ``delete()`` method of an entity instance, Pony marks the object as deleted. The object will be removed from the database during the following commit.

For example, this is how we can delete an order with the primary key equal to 123::

    Order[123].delete()

In future, Pony is going to support bulk deletes. It will allow the deletion of multiple objects without loading them to the cache::

    delete(p for p in Product if p.category.name == "Floppy disk")


Cascade delete
~~~~~~~~~~~~~~~~~~~~~~~~

When Pony deletes an instance of an entity it also needs to delete its relationships with other objects. The relationships between two objects are defined by two relationship attributes. If another side of the relationship is declared as a ``Set``, then we just need to remove the object from that collection. If another side is declared as ``Optional``, then we need to set it to ``None``. If another side is declared as ``Required``,  we cannot just assign ``None`` to that relationship attribute. In this case, Pony will try to do a cascade delete of the related object. This default behavior can be changed using the :ref:`cascade_delete <attribute_cascade_delete_ref>` option of an attribute. By default this option is set to ``True`` if another side of the relationship is declared as ``Required`` and ``False`` for all other relationships.

``True`` means that Pony always does cascade delete even if the other side is defined as ``Optional``. ``False`` means that Pony never does cascade delete for this relationship. If the relationship is defined as ``Required`` at the other end and ``cascade_delete=False`` then Pony raises the ``ConstraintError`` exception on deletion attempt.

Cascade delete for one-to-many relationship example. Raises the ``ConstraintError`` exception on an attempt to delete a group which has related students::

    class Group(db.Entity):
        major = Required(str)
        items = Set("Student", cascade_delete=False)

    class Student(db.Entity):
        name = Required(str)
        group = Required(Group)


Cascade delete for one-to-one relationship example. Deletes a related instance of ``Passport`` when deleting the instance of ``Person``::

    class Person(db.Entity):
        name = Required(str)
        passport = Optional("Passport", cascade_delete=True)

    class Passport(db.Entity):
        number = Required(str)
        person = Required("Person")




Entity class methods
----------------------

.. class:: Entity

   .. py:method:: []

      Returns an entity instance selected by its primary key. Raises the ``ObjectNotFound`` exception if there is no such object. Example::

          p = Product[123]

      For entities with a compound primary key, use a comma between the primary key values::

          order_id = 123
          product_id = 456
          item = OrderItem[123, 456]

      If object with the specified primary key was already loaded into the ``db_session`` cache, Pony returns the object from the cache without sending a query to the database.

   .. py:method:: describe()

      Returns a string with the entity declaration. Example::

          >>> from pony.orm.examples.estore import *
          >>> print OrderItem.describe()

          class OrderItem(Entity):
              quantity = Required(int)
              price = Required(Decimal)
              order = Required(Order)
              product = Required(Product)
              PrimaryKey(order, product)

   .. py:method:: drop_table(with_all_data=False)

      Drops the table which is associated with the entity in the database. If the table is not empty and ``with_all_data=False``, the method raises the ``TableIsNotEmpty`` exception and doesn't delete anything. Setting the ``with_all_data=True`` allows you to delete the table even if it is not empty.

      If you need to delete an intermediate table created for many-to-many relationship, you have to call the method ``drop_table`` of the relationship attribute of the entity class(not instance)::

          class Product(db.Entity):
              tags = Set('Tag')

          class Tag(db.Entity):
              products = Set(Product)

          Product.tags.drop_table(with_all_data=True) # removes the intermediate table

   .. py:method:: exists(lambda[, globals[, locals])
                  exists(**kwargs)

      Returns ``True`` if an instance with the specified condition or attribute values exists and ``False`` otherwise. Examples::

          Product.exists(price=1000)

          Product.exists(lambda p: p.price > 1000)

   .. _entity_get_ref:

   .. py:method:: get(lambda[, globals[, locals])
                  get(**kwargs)

      Used for extracting one entity instance from the database. If the object with the specified parameters exists, then returns the object. Returns ``None`` if there is no such object. If there are more than one objects with the specified parameters, raises the ``MultipleObjectsFoundError: Multiple objects were found. Use select(...) to retrieve them`` exception. Examples::

          Product.get(price=1000)

          Product.get(lambda p: p.name.startswith('A'))

   .. py:method:: select_by_sql(sql, globals=None, locals=None)
   .. py:method:: get_by_sql(sql, globals=None, locals=None)

      If you find that you cannot express a query using the standard Pony queries, you always can write your own SQL query and Pony will build an entity instance(s) based on the query results. When Pony gets the result of the SQL query, it analyzes the column names which it receives from the database cursor. If your query uses ``SELECT * ...`` from the entity table, that would be enough for getting the necessary attribute values for constructing entity instances. You can pass parameters into the query, see :ref:`Using the select_by_sql() and get_by_sql() methods <entities_raw_sql_ref>` for more information.

   .. py:method:: get_for_update(lambda,[globals[, locals], nowait=False)
                  get_for_update(**kwargs, nowait=False)

      Similar to :ref:`get() <entity_get_ref>`, but locks the row in the database using the ``SELECT ... FOR UPDATE`` SQL query. If ``nowait=True``, then the method will throw an exception if this row is already blocked. If ``nowait=False``, then it will wait if the row is already blocked.

      If you need to use ``SELECT ... FOR UPDATE`` for multiple rows then you should use the method :py:meth:`for_update()<Query.for_update>` of ``Query`` object.

   .. py:method:: load()
                  load(args)

      Loads all lazy and non-lazy attributes, but not collection attributes, which were not retrieved from the database yet. If an attribute was already loaded, it won't be loaded again. You can specify the list of the attributes which need to be loaded, or it's names. In this case Pony will load only them::

          obj.load(Person.biography, Person.some_other_field)
          obj.load('biography', 'some_other_field')

   .. py:method:: select()
                  select(lambda[, globals[, locals])

      Selects objects from the database in accordance with the condition specified in lambda, or all objects if lambda function is not specified.

      The ``select`` method returns an instance of the :ref:`Query <query_object_ref>` class. Entity instances will be retrieved from the database once you start iterating over the ``Query`` object. Example::

          Product.select(lambda p: p.price > 100 and count(p.order_items) > 1)[:]

      The query above returns all products with a price greater than 100 and which were ordered more than once.

   .. py:method:: select_random(limit)

      Select ``limit`` random objects. This method uses the algorithm that can be much more effective than using ``ORDER BY RANDOM()`` SQL construct. The method uses the following algorithm:

      1. Determine max id from the table.

      2. Generate random ids in the range (0, max_id]

      3. Retrieve objects by those random ids. If an object with generated id does not exist (e.g. it was deleted), then select another random id and retry.

      Repeat the steps 2-3 as many times as necessary to retrieve the specified amount of objects.

      This algorithm doesn't affect performance even when working with a large number of table rows. However this method also has some limitations:

      * The primary key must be a sequential id of an integer type.

      * The number of "gaps" between existing ids (the count of deleted objects) should be relatively small.

      The ``select_random()`` method can be used if your query does not have any criteria to select specific objects. If such criteria is necessary, then you can use the ``random()`` method of the Query object.


Entity instance methods
----------------------------

.. class:: Entity

   .. py:method:: get_pk()

      Returns the value of the primary key of the object.

      .. code-block:: python

          >>> c = Customer[1]
          >>> c.get_pk()
          1

      If the primary key is composite, then this method returns a tuple consisting of primary key column values.

      .. code-block:: python

          >>> oi = OrderItem[1,4]
          >>> oi.get_pk()
          (1, 4)

   .. py:method:: delete()

      Deletes an entity instance. The object will be marked as deleted and will be deleted from the database on the operation ``flush()`` which is issued automatically on committing the current transaction, exiting from the most outer ``db_session`` or before sending next query to the database.

   .. py:method:: set(**kwargs)

      Assign new values to several object attributes at once::

          Customer[123].set(email='new@example.com', address='New address')

      This method also can be convenient when you want to assign new values from a dictionary::

          d = {'email': 'new@example.com', 'address': 'New address'}
          Customer[123].set(**d)

   .. py:method:: to_dict(only=None, exclude=None, with_collections=False, with_lazy=False, related_objects=False)

      Returns a dictionary with attribute names and its values. This method can be used when you need to serialize an object to JSON or other format.

      By default this method doesn't include collections (to-many relationships) and lazy attributes. If an attribute's values is an entity instance then only the primary key of this object will be added to the dictionary.

      ``only`` - use this parameter if you want to get only the specified attributes. This argument can be used as a first positional argument. You can specify a list of attribute names ``obj.to_dict(['id', 'name'])``, a string separated by spaces: ``obj.to_dict('id name')``, or a string separated by spaces with commas: ``obj.to_dict('id, name')``.

      ``exclude`` - this parameter allows you to exclude specified attributes. Attribute names can be specified the same way as for the ``only`` parameter.

      ``related_objects`` - by default, all related objects represented as a primary key. If ``related_objects=True``, then objects which have relationships with the current object will be added to the resulting dict as objects, not their primary keys. It can be useful if you want to walk the related objects and call the ``to_dict()`` method recursively.

      ``with_collections`` - by default, the resulting dictionary will not contain collections (to-many relationships). If you set this parameter to ``True``, then the relationships to-many will be represented as lists. If ``related_objects=False`` (which is by default), then those lists will consist of primary keys of related instances. If ``related_objects=True`` then to-many collections will be represented as lists of objects.

      ``with_lazy`` - if ``True``, then lazy attributes (such as BLOBs or attributes which are declared with ``lazy=True``) will be included to the resulting dict.

      For illustrating the usage of this method we will use the eStore example which comes with Pony distribution. Let's get a customer object with the id=1 and convert it to a dictionary:

      .. code-block:: python

          >>> from pony.orm.examples.estore import *
          >>> c1 = Customer[1]
          >>> c1.to_dict()

          {'address': u'address 1',
          'country': u'USA',
          'email': u'john@example.com',
          'id': 1,
          'name': u'John Smith',
          'password': u'***'}

      If we don't want to serialize the password attribute, we can exclude it this way:

      .. code-block:: python

          >>> c1.to_dict(exclude='password')

          {'address': u'address 1',
          'country': u'USA',
          'email': u'john@example.com',
          'id': 1,
          'name': u'John Smith'}

      If you want to exclude more than one attribute, you can specify them as a list: ``exclude=['id', 'password']`` or as a string: ``exclude='id, password'`` which is the same as ``exclude='id password'``.

      Also you can specify only the attributes, which you want to serialize using the parameter ``only``:

      .. code-block:: python

          >>> c1.to_dict(only=['id', 'name'])

          {'id': 1, 'name': u'John Smith'}

          >>> c1.to_dict('name email') # 'only' parameter as a positional argument

          {'email': u'john@example.com', 'name': u'John Smith'}

      By default the collections are not included to the resulting dict. If you want to include them, you can specify ``with_collections=True``. Also you can specify the collection attribute in the ``only`` parameter:

      .. code-block:: python

          >>> c1.to_dict(with_collections=True)

          {'address': u'address 1',
          'cart_items': [1, 2],
          'country': u'USA',
          'email': u'john@example.com',
          'id': 1,
          'name': u'John Smith',
          'orders': [1, 2],
          'password': u'***'}

      By default all related objects (cart_items, orders) are represented as a list with their primary keys. If you want to see the related objects instances, you can specify ``related_objects=True``:

      .. code-block:: python

          >>> c1.to_dict(with_collections=True, related_objects=True)

          {'address': u'address 1',
          'cart_items': [CartItem[1], CartItem[2]],
          'country': u'USA',
          'email': u'john@example.com',
          'id': 1,
          'name': u'John Smith',
          'orders': [Order[1], Order[2]],
          'password': u'***'}

      If you need to serialize more than one entity instance, or if you need to serialize an instance with its related objects, you can use the :py:func`to_dict` function from the `pony.orm.serialization` module.

   .. py:method:: flush()

      Saves the changes made to this object to the database. Usually Pony saves changes automatically and you don't need to call this method yourself. One of the use cases when it might be needed is when you want to get the primary key value of a newly created object which has autoincremented primary key before commit.

Entity hooks
-----------------------------------


Serializing entity instances
-------------------------------------------


Serialization with pickle
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Pony allows pickling entity instances, query results and collections. You might want to use it if you want to cache entity instances in an external cache (e.g. memcache). When Pony pickles entity instances, it saves all attributes except collections in order to avoid pickling a large set of data. If you need to pickle a collection attribute, you must pickle it separately. Example:

.. code-block:: python

    >>> from pony.orm.examples.estore import *
    >>> products = select(p for p in Product if p.price > 100)[:]
    >>> products
    [Product[1], Product[2], Product[6]]
    >>> import cPickle
    >>> pickled_data = cPickle.dumps(products)

Now we can put the pickled data to a cache. Later, when we need our instances again, we can unpickle it:

.. code-block:: python

    >>> products = cPickle.loads(pickled_data)
    >>> products
    [Product[1], Product[2], Product[6]]

You can use pickling for storing objects in an external cache for improving application performance. When you unpickle objects, Pony adds them to the current ``db_session`` as if they were just loaded from the database. Pony doesn't check if objects hold the same state in the database.

Serialization to a dictionary and to JSON
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Another way to serialize entity instances is to use the :py:meth:`to_dict() <Entity.to_dict>` method of an entity instance or :py:func:`to_dict` and :py:func:`to_json` functions from the ``pony.orm.serialization`` module. The instance's :py:meth:`to_dict() <Entity.to_dict>` method returns a key-value dictionary structure for a specific entity instance. Sometimes you might need to serialize not only the instance itself, but also the instance's related objects. In this case you can use the ``to_dict()`` function described below.

.. py:function:: to_dict

   This function is used for serializing entity instances to a dictionary. It can receive an entity instance or any iterator which returns entity instances.  The function returns a multi-level dictionary which includes the objects passed to the function, as well as the immediate related objects. Here is a structure of the resulting dict:

   .. code-block:: python

       {
           'entity_name': {
               primary_key_value: {
                   attr: value,
                   ...
               },
               ...
           },
           ...
       }

   Let’s use our online store model example (see the `ER diagram <https://editor.ponyorm.com/user/pony/eStore>`_) and print out the result of the ``to_dict()`` function. Note that you need to import the to_dict function separately:

   .. code-block:: python

       from pony.orm.examples.estore import *
       from pony.orm.serialization import to_dict

       print to_dict(Order[1])

       {
           'Order': {
               1: {
                   'id': 1,
                   'state': u'DELIVERED',
                   'date_created': datetime.datetime(2012, 10, 20, 15, 22)
                   'date_shipped': datetime.datetime(2012, 10, 21, 11, 34),
                   'date_delivered': datetime.datetime(2012, 10, 26, 17, 23),
                   'total_price': Decimal('292.00'),
                   'customer': 1,
                   'items': ['1,1', '1,4'],
                   }
               }
           },
           'Customer': {
               1: {
                   'id': 1
                   'email': u'john@example.com',
                   'password': u'***',
                   'name': u'John Smith',
                   'country': u'USA',
                   'address': u'address 1',
               }
           },
           'OrderItem': {
               '1,1': {
                   'quantity': 1
                   'price': Decimal('274.00'),
                   'order': 1,
                   'product': 1,
               },
               '1,4': {
                   'quantity': 2
                   'price': Decimal('9.98'),
                   'order': 1,
                   'product': 4,
               }
           }
       }

   In the example above the result contains the serialized ``Order[1]`` instance as well as the immediate related objects: ``Customer[1]``, ``OrderItem[1, 1]`` and ``OrderItem[1, 4]``.


.. py:function:: to_json

   This function uses the output of :py:func:`to_dict` and returns its JSON representation.



Saving objects in the database
--------------------------------------

Order of saving objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Usually Pony saves objects in the database in the same order as they are created or modified. In some cases Pony can reorder SQL INSERT statements if this is required for saving objects. Let's consider the following example:

.. code-block:: python

    from pony.orm import *

    db = Database('sqlite', ':memory:')

    class TeamMember(db.Entity):
        name = Required(str)
        team = Optional('Team')

    class Team(db.Entity):
        name = Required(str)
        team_members = Set(TeamMember)

    db.generate_mapping(create_tables=True)
    sql_debug(True)

    with db_session:
        john = TeamMember(name='John')
        mary = TeamMember(name='Mary')
        team = Team(name='Tenacity', team_members=[john, mary])

In the example above we create two team members and then a team object, assigning the team members to the team. The relationship between TeamMember and Team objects is represented by a column in the TeamMember table:

.. code-block:: sql

    CREATE TABLE "Team" (
      "id" INTEGER PRIMARY KEY AUTOINCREMENT,
      "name" TEXT NOT NULL
    )

    CREATE TABLE "TeamMember" (
      "id" INTEGER PRIMARY KEY AUTOINCREMENT,
      "name" TEXT NOT NULL,
      "team" INTEGER REFERENCES "Team" ("id")
    )

When Pony creates ``john``, ``mary`` and ``team`` objects, it understands that it should reorder SQL INSERT statements and create an instance of the ``Team`` object in the database first, because it will allow using the team id for saving TeamMember rows:

.. code-block:: sql

    INSERT INTO "Team" ("name") VALUES (?)
    [u'Tenacity']

    INSERT INTO "TeamMember" ("name", "team") VALUES (?, ?)
    [u'John', 1]

    INSERT INTO "TeamMember" ("name", "team") VALUES (?, ?)
    [u'Mary', 1]


Cyclic chains during saving objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now let's say we want to have an ability to assign a captain to a team. For this purpose we need to add a couple of attributes to our entities: ``Team.captain`` and reverse attribute ``TeamMember.captain_of``

.. code-block:: python

    class TeamMember(db.Entity):
        name = Required(str)
        team = Optional('Team')
        captain_of = Optional('Team')

    class Team(db.Entity):
        name = Required(str)
        team_members = Set(TeamMember)
        captain = Optional(TeamMember, reverse='captain_of')

And here is the code for creating entity instances with a captain assigned to the team:

.. code-block:: python

    with db_session:
        john = TeamMember(name='John')
        mary = TeamMember(name='Mary')
        team = Team(name='Tenacity', team_members=[john, mary], captain=mary)

When Pony tries to execute the code above it raises the following exception::

    pony.orm.core.CommitException: Cannot save cyclic chain: TeamMember -> Team -> TeamMember

Why did it happen? Let's see. Pony sees that for saving the ``john`` and ``mary`` objects in the database it needs to know the id of the team, and tries to reorder the insert statements. But for saving the ``team`` object with the ``captain`` attribute assigned, it needs to know the id of ``mary`` object. In this case Pony cannot resolve this cyclic chain and raises an exception.

In order to save such a cyclic chain, you have to help Pony by adding the :ref:`flush() <flush_ref>` command:

.. code-block:: python

    with db_session:
        john = TeamMember(name='John')
        mary = TeamMember(name='Mary')
        flush() # saves objects created by this moment in the database
        team = Team(name='Tenacity', team_members=[john, mary], captain=mary)

In this case, Pony will save ``john`` and ``mary`` objects in the database first and then will issue SQL UPDATE statement for building the relationship with the ``team`` object:

.. code-block:: sql

    INSERT INTO "TeamMember" ("name") VALUES (?)
    [u'John']

    INSERT INTO "TeamMember" ("name") VALUES (?)
    [u'Mary']

    INSERT INTO "Team" ("name", "captain") VALUES (?, ?)
    [u'Tenacity', 2]

    UPDATE "TeamMember"
    SET "team" = ?
    WHERE "id" = ?
    [1, 2]

    UPDATE "TeamMember"
    SET "team" = ?
    WHERE "id" = ?
    [1, 1]

