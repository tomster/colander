Extending Colander
==================

You can extend Colander by defining a new :term:`type` or by defining
a new :term:`validator`.

.. _defining_a_new_type:

Defining a New Type
-------------------

A new type is a class with two methods:: ``serialize`` and
``deserialize``.  ``serialize`` converts a Python data structure (an
:term:`appstruct`) into a serialization (a :term:`cstruct`).
``deserialize`` converts a serialized value (a :term:`cstruct`) into a
Python data structure (a :term:`appstruct`).

Here's a type which implements boolean serialization and
deserialization.  It serializes a boolean to the string ``true`` or
``false`` or the special :attr:`colander.null` sentinel; it then
deserializes a string (presumably ``true`` or ``false``, but allows
some wiggle room for ``t``, ``on``, ``yes``, ``y``, and ``1``) to a
boolean value.

.. code-block::  python
   :linenos:

   from colander import null

   class Boolean(object):
       def serialize(self, node, appstruct):
           if appstruct is null:
               return null
           if not isinstance(appstruct, bool):
              raise Invalid(node, '%r is not a boolean')
           return appstruct and 'true' or 'false'

       def deserialize(self, node, cstruct):
           if not isinstance(cstruct, basestring):
               raise Invalid(node, '%r is not a string' % cstruct)
           value = cstruct.lower()
           if value in ('true', 'yes', 'y', 'on', 't', '1'):
               return True
           return False

Note that the ``deserialize`` method of a type does not need to
explicitly deserialize the :attr:`colander.null` value.
Deserialization of the null value is dealt with at a higher level
(with the :meth:`colander.SchemaNode.deserialize` method); a type will
never receive an :attr:`colander.null` value as a ``cstruct`` argument
to its ``deserialize`` method.

Here's how you would use the resulting class as part of a schema:

.. code-block:: python
   :linenos:

   import colander

   class Schema(colander.MappingSchema):
       interested = colander.SchemaNode(Boolean())

The above schema has a member named ``interested`` which will now be
serialized and deserialized as a boolean, according to the logic
defined in the ``Boolean`` type class.

Note that the only two real constraints of a type class are:

- it must deal specially with the value :attr:`colander.null` within
  ``serialize``, translating it to a type-specific null value.

- its ``serialize`` method must be able to make sense of a value
  generated by its ``deserialize`` method and vice versa, except that
  the ``deserialize`` method needn't deal with the
  :attr:`colander.null` value specially even if the ``serialize``
  method returns it.

The ``serialize`` method of a type accepts two values: ``node``, and
``appstruct``.  ``node`` will be the schema node associated with this
type.  It is used when the type must raise a :exc:`colander.Invalid`
error, which expects a schema node as its first constructor argument.
``appstruct`` will be the :term:`appstruct` value that needs to be
serialized.

The deserialize and method of a type accept two values: ``node``, and
``cstruct``.  ``node`` will be the schema node associated with this
type.  It is used when the type must raise a :exc:`colander.Invalid`
error, which expects a schema node as its first constructor argument.
``cstruct`` will be the :term:`cstruct` value that needs to be
deserialized.

A type class does not need to implement a constructor (``__init__``),
but it isn't prevented from doing so if it needs to accept arguments;
Colander itself doesn't construct any types, only users of Colander
schemas do, so how types are constructed is beyond the scope of
Colander itself.

The :exc:`colander.Invalid` exception may be raised during
serialization or deserialization as necessary for whatever reason the
type feels appropriate (the inability to serialize or deserialize a
value being the most common case).

For a more formal definition of a the interface of a type, see
:class:`colander.interfaces.Type`.

.. _defining_a_new_validator:

Defining a New Validator
------------------------

A validator is a callable which accepts two positional arguments:
``node`` and ``value``.  It returns ``None`` if the value is valid.
It raises a :class:`colander.Invalid` exception if the value is not
valid.  Here's a validator that checks if the value is a valid credit
card number.

.. code-block:: python
   :linenos:

   def luhnok(node, value):
       """ checks to make sure that the value passes a luhn mod-10 checksum """
       sum = 0
       num_digits = len(value)
       oddeven = num_digits & 1

       for count in range(0, num_digits):
           digit = int(value[count])

           if not (( count & 1 ) ^ oddeven ):
               digit = digit * 2
           if digit > 9:
               digit = digit - 9

           sum = sum + digit

       if not (sum % 10) == 0:
           raise Invalid(node, 
                         '%r is not a valid credit card number' % value)
        
Here's how the resulting ``luhnok`` validator might be used in a
schema:

.. code-block:: python
   :linenos:

   import colander

   class Schema(colander.MappingSchema):
       cc_number = colander.SchemaNode(colander.String(), validator=lunhnok)

Note that the validator doesn't need to check if the ``value`` is a
string: this has already been done as the result of the type of the
``cc_number`` schema node being :class:`colander.String`. Validators
are always passed the *deserialized* value when they are invoked.

The ``node`` value passed to the validator is a schema node object; it
must in turn be passed to the :exc:`colander.Invalid` exception
constructor if one needs to be raised.

For a more formal definition of a the interface of a validator, see
:class:`colander.interfaces.Validator`.

