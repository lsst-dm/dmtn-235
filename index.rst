Abstract
========

The authentication service for the Rubin Science Platform (Gafaelfawr_) uses tokens to authenticate users.
Each token is associated with a list of scopes, which are used to make authorization decisions.
This tech note lists the scopes currently in use by the Science Platform, defines them, and discusses the services to which each scope grants access.

This document will be updated when new scopes are added, their meanings are changed, or the services to which they grant access change.

How token scopes are assigned and how they are used for authorization is discussed in DMTN-234_, which presents the design for the Science Platform authentication service.

.. _Gafaelfawr: https://gafaelfawr.lsst.io/
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

``admin:provision``
    Grants access to the API used to provision home directories and other resources for new users.
    This allows running a privileged container with arbitrary user data (although the container itself is not under the control of the API client).
    Provisioning is currently provided by the moneypenny_ service.

    This scope is granted to deployment administrators and to the nublado2 service so that it can provision new users on initial lab spawn.

.. _moneypenny: https://github.com/lsst-sqre/moneypenny

``exec:admin``
    Grants access to various privileged services that should only be used by deployment administrators.
    This is a bit of a grab bag for administrative access by humans to services that don't need to allow access from other services.

    Currently, this grants access to the admin APIs of cachemachine_, mobu_, noteburst_, sherlock_, and times-square_, and to the admin routes for the Portal Aspect.

.. _cachemachine: https://github.com/lsst-sqre/cachemachine
.. _sherlock: https://github.com/lsst-sqre/sherlock

``exec:notebook``
    Allows the user to spawn a lab in the Notebook Aspect.
    This in turn allows arbitrary command execution within an unprivileged JupyterLab_ pod.

.. _JupyterLab: https://jupyterlab.readthedocs.io/en/stable/

``exec:portal``
    Allows the user to perform operations in the Portal Aspect.
    Due to the underlying queries the Portal performs, the user will also need ``read:image`` and ``read:tap`` scopes.

``read:alertdb``
    Grants access to receive alert packets and schemas from the alert archive database.

``read:image``
    Grants access to retrieve images accessible via the Science Platform.
    Currently, this controls access to the HiPS (see DMTN-230_), SODA image cutout (see DMTN-208_), and DataLink (as implemented by datalinker_) services.

    At present, there is a single scope for access to all images.
    In the future, this may be broken into separate scopes for data releases or types of images.

.. _DMTN-230: https://dmtn-230.lsst.io/
.. _DMTN-208: https://dmtn-208.lsst.io/
.. _datalinker: https://github.com/lsst-sqre/datalinker

``read:tap``
    Grants access to perform queries in the TAP service.

``user:token``
    Can create and modify tokens for the same user as the token that has this scope (as opposed to ``admin:token``, which allows any operation on tokens for any user).
    This scope is automatically granted to users when they authenticate.
    It exists as a separate scope primarily so that users can choose not to grant it to user tokens that they create, so that their programmatic tokens cannot themselves create new tokens.
