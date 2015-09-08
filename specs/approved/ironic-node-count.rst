..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==================================================
Count ironic nodes matching capabilities
==================================================

Include the URL of your launchpad blueprint:

https://blueprints.launchpad.net/ironic/+spec/ironic-node-count

This spec talks about adding a new ironic command to count the number of nodes
that match specific capabilities.

Problem description
===================

Currently, the capabilities field in ironic node can be used to store special
information about the node, for example vendor: Foo, multi_ip: True, etc
which is then used at schedule time. It's useful to have counting functions
to count how many nodes are available matching specific capabilities, for
example, vendor=Foo.

Proposed change
===============

A new command needs to be added. The command should take some capabilties,
and count the number of nodes that It's useful to have counting functions
to count how many nodes are available matching specific capabilities, for
example, vendor=Foo.

ironic node-count -p <key=value>, --properties <key=value>

Alternatives
------------
None.

Data model impact
-----------------
None.

State Machine Impact
--------------------
None.

REST API impact
---------------

Each API method which is either added or changed should have the following

* Specification for the method

  * A description of what the method does, suitable for use in user
    documentation.

  * Method type (POST/PUT/GET/DELETE/PATCH)

  * Normal http response code(s)

  * Expected error http response code(s)

    * A description for each possible error code should be included.
      Describe semantic errors which can cause it, such as
      inconsistent parameters supplied to the method, or when a
      resource is not in an appropriate state for the request to
      succeed. Errors caused by syntactic problems covered by the JSON
      schema definition do not need to be included.

  * URL for the resource

  * Parameters which can be passed via the url, including data types

  * JSON schema definition for the body data if allowed

  * JSON schema definition for the response data if any

* Does the API microversion need to increment?

* Example use case including typical API samples for both data supplied
  by the caller and the response

* Discuss any policy changes, and discuss what things a deployer needs to
  think about when defining their policy.

* Is a corresponding change in the client library and CLI necessary?

* Is this change discoverable by clients? Not all clients will upgrade at the
  same time, so this change must work with older clients without breaking them.

Note that the schema should be defined as restrictively as possible. Parameters
which are required should be marked as such and only under exceptional
circumstances should additional parameters which are not defined in the schema
be permitted.

Use of free-form JSON dicts should only be permitted where necessary to allow
divergence in the drivers. In such case, the drivers must expose the expected
content of the JSON dict and an ability to validate it.

Reuse of existing predefined parameter types is highly encouraged.

Client (CLI) impact
-------------------
Typically, but not always, if there are any REST API changes, there are
corresponding changes to python-ironicclient. If so, what does the user
interface look like. If not, describe why there are REST API changes but
no changes to the client.

RPC API impact
--------------
None.

Driver API impact
-----------------
None.

Nova driver impact
------------------
None.

Security impact
---------------
None.

Other end user impact
---------------------
None.

Scalability impact
------------------
Each time this command is called, it will make a call to the ironic db.

Performance Impact
------------------
None.

Other deployer impact
---------------------
Deployers can use this new command to audit their inventory, find out if they
can satisfy special capability based requests, etc.

Developer impact
----------------
None.

Implementation
==============

Assignee(s)
-----------
praneshpg

Work Items
----------
* Add new command ironic node-count to python-ironicclient
* Add backend support for this command
* db code to count nodes based on capabilities passed


Dependencies
============
None.

Testing
=======

Please discuss how the change will be tested. We especially want to know what
tempest tests will be added. It is assumed that unit test coverage will be
added so that doesn't need to be mentioned explicitly, but discussion of why
you think unit tests are sufficient and we don't need to add more tempest
tests would need to be included.

Is this untestable in gate given current limitations (specific hardware /
software configurations available)? If so, are there mitigation plans (3rd
party testing, gate enhancements, etc)?


Upgrades and Backwards Compatibility
====================================
Since this change does not modify any existing functionality or command, no
backwards incompatible change happens.


Documentation Impact
====================
The new command will be documented. No change is required to the developer
documentation.


References
==========
None.
