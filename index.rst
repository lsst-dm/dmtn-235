###########################################
Token scopes for the Rubin Science Platform
###########################################

.. abstract::

   The authentication service for the Rubin Science Platform (Gafaelfawr_) uses tokens to authenticate users.
   Each token is associated with a list of scopes, which are used to make authorization decisions.
   This tech note lists the scopes currently in use by the Science Platform, defines them, and discusses the services to which each scope grants access.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/

This document will be updated when new scopes are added, their meanings are changed, or the services to which they grant access change.

.. note::

   This is part of a tech note series on identity management for the Rubin Science Platform.
   The primary documents are :dmtn:`234`, which describes the high-level design; :dmtn:`224`, which describes the implementation; and :sqr:`069`, which provides a history and analysis of the decisions underlying the design and implementation.
   See the `references section of DMTN-224 <https://dmtn-224.lsst.io/#references>`__ for a complete list of related documents.

.. _purpose:

Purpose of scopes
=================

Scopes are used for "coarse-grained" access control: whether a user can access a specific component or API at all, or whether the user is allowed to access administrative interfaces for a service.
"Fine-grained" access control decisions made by services, such as whether a user with general access to the service is able to run a specific query or access a specific image, are instead made based on the user's group membership.

Specifically, token scopes are not used to control access to data releases.
Access decisions for specific data releases should be made based on user group membership.

How token scopes are assigned and how they are used for authorization is discussed in :dmtn:`234`, which presents the design for the Science Platform authentication service.

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

``admin:jupyterlab``
    Grants access to inspection and deletion routes on the Nublado (Notebook Aspect) controller (see :sqr:`066`).
    JupyterHub has access to a token with this scope so that it can make lab controller calls that it needs to be able to perform without access to a user token, such as deleting old labs and checking lab status.

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

``admin:userinfo``
    Grants access to view the user information of any user.
    This is separate from ``admin:token`` because some applications, such as the Qserv Kafka bridge, need to obtain user information such as quotas without having access to a user's token.

``exec:admin``
    Grants access to various privileged services that should only be used by deployment administrators.
    This is a bit of a grab bag for administrative access by humans to services that don't need to allow access from other services.

    Currently, this grants access to the admin APIs of mobu_, noteburst_, and times-square_.

.. _sherlock: https://github.com/lsst-sqre/sherlock

``exec:internal-tools``
    Grants access to project-internal tools.
    Examples include observatory logging tools, internal monitoring tools, and tools to inspect data that has not yet been released to general users.

``exec:notebook``
    Allows the user to spawn a lab in the Notebook Aspect.
    This in turn allows arbitrary command execution within an unprivileged JupyterLab_ pod.

    For the time being, this scope is also used to control WebDAV access to the home directory space for the authenticating user, since that home directory space is used primarily by the Notebook Aspect.
    This may eventually be controlled by a different scope if the permissions need to be more granular.

.. _JupyterLab: https://jupyterlab.readthedocs.io/en/stable/

``exec:portal``
    Allows the user to perform operations in the Portal Aspect.

    This grants access only to the Portal itself, not to any of the underlying services that the Portal may query on the user's behalf.
    To fully use all Portal functionality, the user will also need ``read:image`` and ``read:tap`` scopes, and possibly others.
    The Portal requests those scopes if they're available, but does not require them to access the Portal itself.

``exec:portal-admin``
    Grants access to the admin routes of the Portal.
    This can be used to view internal debugging information about the Portal and perform a few administrative actions.
    It is kept separate from ``exec:admin`` so that it can be granted to the Portal development team to allow them to debug production issues.

``read:alertdb``
    Grants access to receive alert packets and schemas from the alert archive database.

``read:image``
    Grants access to retrieve images accessible via the Science Platform.
    Currently, this controls access to HiPS (see :dmtn:`230`), SODA image cutout (see :dmtn:`208`), the DataLink ``/api/datalinker/links`` route (see :dmtn:`238`), and image retrieval from client-server Butler.

    Following the guidelines in :ref:`purpose`, there is a single scope for image access that controls whether the user can download images at all.
    Access to specific images, such as access controls by data release, will be handled via groups.

``read:tap``
    Grants access to perform queries in the TAP service.

``write:files``
    Grants write access to a user's file space.
    Currently, this is used to control access to the user's WebDAV service.
    This does not grant access to other user's files, only to the ones owned by the authenticated user.

``write:sasquatch``
    Grants access to write metrics to the Sasquatch telemetry service (see :sqr:`067`).
    This scope is separate so that it can be granted to service tokens for automated processes (often outside of the Science Platform) that need to record metrics.

``user:token``
    Can create and modify tokens for the same user as the token that has this scope (as opposed to ``admin:token``, which allows any operation on tokens for any user).
    This scope is automatically granted to users when they authenticate.
    It exists as a separate scope primarily so that users can choose not to grant it to user tokens that they create, so that their programmatic tokens cannot themselves create new tokens.

Expected future scopes
======================

``write:tap``
    Write access to personal and group database tables accessible by the TAP service.

It's not yet clear whether the anticipated client/server Butler service (see :dmtn:`176`, :dmtn:`169`, and :dmtn:`182`) will need a separate scope or will reuse existing scopes plus the ``write:tap`` scope.
Currently, read access to images via the Butler is controlled by the ``read:image`` scope.

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
