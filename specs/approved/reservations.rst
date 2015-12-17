..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================
Node Reservations
=================

Include the URL of your launchpad bug with ``rfe`` tag:

https://bugs.launchpad.net/ironic/+bug/1527006

(It is a goal for ironic to work in a standalone fashion, but for purposes of
clarity, we'll assume in this document that nova is the client calling into
ironic.)

Ironic currently doesn't have a state to describe a node that has been picked
up by a scheduler, but not yet been acted upon by the deployer. An example
of such a node is one that is in the nova-compute message queue.

Problem description
===================

Currently, when nova-scheduler picks up a node and puts it in the queue for
nova-compute to process, there is no guarantee that it will be left alone by
further scheduling requests. The node is marked as unavailable only when
nova-compute picks up the node and associates it with an instance_uuid in
ironic.nodes table. This is a race condition that clients calling ironic have
to be aware of and work around.

Let's help the clients by building an interface to mark a node as reserved,
and only returning non-reserved nodes when a list of potential nodes is
requested.

Proposed change
===============

We will add a new resource type "reservation" to ironic. A client may request
a reservation by passing a list of nodes for that reservation. A successful
request for a reservation will create an entry in the database. Furthermore,
when a client asks for eligible nodes (to deploy on), any node that is marked
reserved will not be returned. In other words, only nodes satisfying all the
following conditions will be returned:
  o in AVAILABLE state,
  o have NULL instance_uuid
  o no associated reservation

Reservations may be listed, and fetched or deleted by ID. 

A deployment request for a node must now include the reservation id, which
is proof that the request that created the reservation is the same one that is
using it. This will preclude any client not using the reservation system from
trying to use a reserved node for deployment.

Alternatives
------------

We could expect all the clients using ironic to have a lock/other mechanism to
prevent the race condition described in the problem description section above.

There is another spec under review that discusses adding a claim api[0]. That
can be thought of as having two major parts: selecting a node, and locking a
node. This spec largely describes the latter, and ideally the claim api must
just allow locking (and not HAVE to filter).


Data model impact
-----------------

A new ``reservations`` table will be added, with the following fields:

* `created_at` - DATETIME - the timestamp of creation

* `updated_at` - DATETIME - the timestamp of the last update

* `expires_at` - DATETIME - the timestamp of the time this reservation expires

* `id` - INT - auto-increment ID

* `uuid` - STRING - unique UUID4 for the reservation

* `node_uuid` - UUID4 - foreign key to the `nodes` table

* `instance_uuid` - UUID4 - uuid of the client node requesting this reservation


State Machine Impact
--------------------

No changes, except that a reseration can only be created for a node that is
in AVAILABLE state and not associated with an `instance_uuid`.

REST API impact
---------------

A new resource /v1/reservations will be added, supporting POST, GET and DELETE
operations.

POST will accept a json body consisting of the client `instance_uuid`
requesting the and a list of `node_uuid`. For example::

 POST /v1/reservations

 {
    "instance_uuid": "ef1896b0-a794-425e-9396-e7557ad1bf61",
    "node_uuid": [
        "96b3a7ff-df46-4211-94c3-a5570a0881aa",
        "f985d47a-c858-48ac-b0ae-4f6f433f0c24"
    ]
 }

* This method iterates through the list of node_uuids, and returns the first
  node that is in AVAILABLE state, does not have an `instance_uuid` associated,
  is not in maintenance mode, and does not have a reservation.

* Returns 201 Created on success, with the Reservation object in the response
  body. The reservation object will contain ONE `node_uuid` that is the ironic
  node that could be reserved. Sample::
    {
      "reservation":
        {
          "instance_uuid": <uuid>,
          "created_at": <datetime>,
          "node_uuid": <uuid>,
          "uuid": <uuid>,
          "updated_at": <datetime>
        }
    }

* Returns 409 Conflict if a reservation cannot be created. This could be
  because none of the nodes are free, none of the nodes exist, or any
  combination of the above.
  TODO(praneshp): Should we include a list of reasons a node couldn't be 
                  selected?


GET /v1/reservations

* Return a list of reservation objects.

* Returns 200 on success.

* No error cases.

* Response body::

    {
      "reservations": [
        {
          "instance_uuid": <uuid>
          "created_at": <datetime>,
          "node_uuid": <uuid>,
          "uuid": <uuid>,
          "updated_at": <datetime>
        },
        {
          "instance_uuid": <uuid>
          "created_at": <datetime>,
          "node_uuid": <uuid>,
          "uuid": <uuid>,
          "updated_at": <datetime>
        }
      ]
    }

GET /v1/reservations/<uuid>

* Returns the reservation object for <uuid>.

* Returns 200 on success.

* Returns 404 if a reservation with <uuid> does not exist.

* No additional parameters.

* Empty request body.

* Response body::

    {
      "reservation":
        {
          "uuid": <uuid>,
          "node_uuid": <uuid>,
          "instance_uuid": <uuid>
          "created_at": <datetime>,
          "updated_at": <datetime>
        }
    }

DELETE /v1/reservations/<uuid>

* Deletes the reservation object for <uuid>.

* Returns 204 on success.

* Returns 404 if a reservation with <uuid> does not exist.

* No additional parameters.

* Empty request body.

* Empty response body.

The API version should be bumped for this feature.

Client (CLI) impact
-------------------

The client should also be updated to support this feature. The python library
and the CLI should both support this. The CLI should add the new commands,
which each correspond to the new REST API methods:

* ironic reservation-create <node uuid> <instance uuid>

* ironic reservation-list

* ironic reservation-show <uuid>

* ironic reservation-delete <uuid>

TODO(praneshp): Find out if we need to filter nodes by reservation. This can
be achieved by grepping through reservation-list

RPC API impact
--------------

None

Driver API impact
-----------------

None.

Nova driver impact
------------------

This change likely means changes in the nova driver (and any other client using
ironic), to make use of the reservations api. They should be able to work
without a reservation api, except that they will not be able to deploy on
nodes that are already 'reserved'.

Security impact
---------------

None.

Other end user impact
---------------------

None.

Scalability impact
------------------

None.

Performance Impact
------------------

No huge performance impact for this API alone. Should take O(N) at worst, where
N is the number of nodes the client sends.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee: praneshp

Work Items
----------

* Create new db tables

* Create new API endpoints

* Create new CLI calls

* Document changes

Dependencies
============

Ideally, these changes must be part of the spec discussing the claim api[0],
with an option to just create a reservation without filtering

Testing
=======

Tempest tests should be added for the new API endpoints. DSVM jobs will also
help cover this code once it is being used by Nova.

Upgrades and Backwards Compatibility
====================================

None

Documentation Impact
====================

Docs would be required for the changes described.

References
==========

[0] https://review.openstack.org/#/c/204641/
