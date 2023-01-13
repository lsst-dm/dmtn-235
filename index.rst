:tocdepth: 1

.. sectnum::

Abstract
========

The authentication service for the Rubin Science Platform (Gafaelfawr_) uses tokens to authenticate users.
Each token is associated with a list of scopes, which are used to make authorization decisions.
This tech note lists the scopes currently in use by the Science Platform, defines them, and discusses the services to which each scope grants access.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

This document will be updated when new scopes are added, their meanings are changed, or the services to which they grant access change.

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are DMTN-234_, which describes the high-level design; DMTN-224_, which describes the implementation; and SQR-069_, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

.. _DMTN-234: https://dmtn-234.lsst.io/
.. _DMTN-224: https://dmtn-224.lsst.io/
.. _SQR-069: https://sqr-069.lsst.io/

.. _purpose:

Purpose of scopes
=================

Scopes are used for "coarse-grained" access control: whether a user can access a specific component or API at all, or whether the user is allowed to access administrative interfaces for a service.
"Fine-grained" access control decisions made by services, such as whether a user with general access to the service is able to run a specific query or access a specific image, are instead made based on the user's group membership.

How token scopes are assigned and how they are used for authorization is discussed in DMTN-234_, which presents the design for the Science Platform authentication service.

.. _DMTN-234: https://dmtn-234.lsst.io/

Scope naming
============

A scope name may contain any alphanumeric ASCII character or colon (``:``), hyphen (``-``), underscore (``_``), or period (``.``).
Beyond that, scopes are arbitrary labels.
Each Science Platform deployment can define whatever scopes it wishes.

However, the best practice recommendation is to use either ``<system>:<component>`` or ``<verb>:<data-class>`` naming for scopes.
``admin:token`` is an example of the former.
``exec:notebook`` or ``read:tap`` are examples of the latter.

The first convention is generally used for more internal token scopes, such as access to operations inside the authentication system itself.
The second convention is used for access to data products and astronomy services.

Current scopes
==============

The following scopes are currently in use:

``admin:provision``
    Grants access to the API used to provision home directories and other resources for new users.
    This allows running a privileged container with arbitrary user data (although the container itself is not under the control of the API client).
    Provisioning is currently provided by the moneypenny_ service.

    This scope is granted to deployment administrators and to the JupyterHub service so that it can provision new users on initial lab spawn.

    If SQR-066_ is implemented, this scope will be retired and replaced with an ``admin:notebook`` scope with slightly different capabilities.

.. _moneypenny: https://github.com/lsst-sqre/moneypenny
.. _SQR-066: https://sqr-066.lsst.io/

``admin:token``
    Grants token administrator powers.
    Users authenticated with a token with this scope can view, create, modify, and delete tokens for any user.
    Administrators (as configured in Gafaelfawr_) are automatically granted this scope when they authenticate.

    This scope effectively grants full administrative access to the Science Platform and all of its services, since even if the user lacks other scopes, they can use this scope to create a new token with any scopes they wish.

    This scope is used to create tokens for bot users by mobu_, noteburst_, and times-square_.
    It is also used by administrators to impersonate users for debugging purposes.

.. _mobu: https://github.com/lsst-sqre/mobu
.. _noteburst: https://noteburst.lsst.io/
.. _times-square: https://github.com/lsst-sqre/times-square

``exec:admin``
    Grants access to various privileged services that should only be used by deployment administrators.
    This is a bit of a grab bag for administrative access by humans to services that don't need to allow access from other services.

    Currently, this grants access to the admin APIs of cachemachine_, mobu_, noteburst_, sherlock_, and times-square_, and to the admin routes for the Portal Aspect.
    We may replace this scope with one or more scopes starting with ``admin:`` to keep all admin scopes similarly named.

.. _cachemachine: https://github.com/lsst-sqre/cachemachine
.. _sherlock: https://github.com/lsst-sqre/sherlock

``exec:internal-tools``
    Grants access to project-internal tools.
    Examples include observatory logging tools, internal monitoring tools, and tools to inspect data that has not yet been released to general users.

``exec:notebook``
    Allows the user to spawn a lab in the Notebook Aspect.
    This in turn allows arbitrary command execution within an unprivileged JupyterLab_ pod.

.. _JupyterLab: https://jupyterlab.readthedocs.io/en/stable/

``exec:portal``
    Allows the user to perform operations in the Portal Aspect.

    This grants access only to the Portal itself, not to any of the underlying services that the Portal may query on the user's behalf.
    To fully use all Portal functionality, the user will also need ``read:image`` and ``read:tap`` scopes, and possibly others.
    The Portal requests those scopes if they're available, but does not require them to access the Portal itself.

``read:alertdb``
    Grants access to receive alert packets and schemas from the alert archive database.

``read:image``
    Grants access to retrieve images accessible via the Science Platform.
    Currently, this controls access to HiPS (see DMTN-230_), SODA image cutout (see DMTN-208_), and the DataLink ``/api/datalinker/links`` route (as implemented by datalinker_).

    Following the guidelines in :ref:`purpose`, there is a single scope for image access that controls whether the user can download images at all.
    Access to specific images, such as access controls by data release, will be handled via groups.

.. _DMTN-230: https://dmtn-230.lsst.io/
.. _DMTN-208: https://dmtn-208.lsst.io/
.. _datalinker: https://github.com/lsst-sqre/datalinker

``read:tap``
    Grants access to perform queries in the TAP service.

``user:token``
    Can create and modify tokens for the same user as the token that has this scope (as opposed to ``admin:token``, which allows any operation on tokens for any user).
    This scope is automatically granted to users when they authenticate.
    It exists as a separate scope primarily so that users can choose not to grant it to user tokens that they create, so that their programmatic tokens cannot themselves create new tokens.

Expected future scopes
======================

``admin:notebook``
    Grants access to the Notebook Aspect lab controller, if implemented according to the design in SQR-066_.
    This must exist as a separate scope so that it can be granted to the JupyterHub service.

``write:tap``
    Write access to personal and group database tables accessible by the TAP service.

It's not yet clear whether the anticipated client/server Butler service (see DMTN-176_, DMTN-169_, and DMTN-182_) will need a separate scope or will reuse one of the existing scopes plus the ``write:tap`` scope.

.. _DMTN-169: https://dmtn-169.lsst.io/
.. _DMTN-176: https://dmtn-176.lsst.io/
.. _DMTN-182: https://dmtn-182.lsst.io/

Creating new scopes
===================

Many authorization systems discover too late that they've allowed scopes to proliferate to the point where they become confusing and difficult to keep track of.
For example, granting additional scopes to users makes the token management UI more complex for the user.
When the user is creating new tokens, they are expected to pick the scopes that token should have so that it does not have excessive access.
Ideally, the number of scopes they're presented with should be no more than 10 and should be obvious and self-explanatory.

To avoid a confusing proliferation of scopes, the Rubin Science Platform only creates new scopes when there is a clear and compelling need.
Specifically,

#. there exist two users who should receive different levels of access to the same deployment in a way that cannot be represented by the existing scopes, and
#. this access control difference must be done with scopes and not groups.

As discussed in :ref:`purpose`, scopes control access to a service in its entirety, or to the administrative API as opposed to the user API of the service.
Groups are used for all other access control.
Groups must be interpreted by each service (or by another service to which the first service delegates access control decisions).
Scopes are enforced by the authentication layer, before the service ever sees the request, since they determine access to the service in the first place.

Developers of Science Platform services who, after considering the above factors, still believe a new scope is warranted should raise the issue with the SQuaRE team.
