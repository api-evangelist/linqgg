### LinQ Wallet Public API

This API reference describes the modules available in the API, which are:

- __Geo__: This module provides APIs for checking the location of a user or device.
- __Auth__: This module provides APIs for authenticating users in wallet services.
- __Play__: This module provides APIs for managing playing sessions and money related thinds.
- __Money__: This module provides APIs for managing money, such as transferring money between accounts and paying bills.
- __Sandbox__: Module with helpers to make integrations easier. It works only in test mode.

#### GEO Module

The geo module provides the following APIs:

##### Restrictions of access by IP

An IP restriction service allows you to control who can access an application by determining the user's location based on IP address. The service returns the detected locations with their status: allowed or not.

##### Operations availability by user location

We may restrict access to certain features based on different rules in different locations. To check if the requested feature is allowed in specific location, you have to provide user's coordinates to this API.

#### Auth Module

The auth module provides the following APIs:

##### Game Authentification

A game user can create an anonymous account for the wallet services with this service. Optionally, it can take user details to make the registration process in the wallet smoother.

##### User Authentification

This service authenticates the user with the wallet service and provides an access token, which must be used to make protected API calls. Additionally, it allows users to sign in through the Wallet app using a permanent wallet token.

#### Play Module

Services under Play module introduce a way of registering play sessions that happen on the game side, internally implementing all money-related stuff, like betting and reward spreading. Also, it includes internal integrity and security checks created for better protection against unexpected behavior or any other actions. The Play module provides the following APIs.

##### Sessions Service

It allows managing playing sessions, like initiating, finishing and dissolving sessions. It provides vareous ways of configuration if such sessions, inluding automatic spreading the prize across tournament table.

##### Players Service

It services allows performs actions behind the user (player), and allows to join into the session with specified params, like stake, for example. As well, it allows gian the prize in case autospreading is not used.

#### Money Module

The money module provides the following APIs:

##### Accounts & Balance

A user balance and transaction service allows you to check the current balance of a user's account and apply transactions to the account.

##### Payments & Transfers

An order management service allows you to place replenish orders, transfer orders, and check the status of orders after they have been placed.

#### Sandbox module

Special module with helpers that allows integration of automated tests or any other type of operations, excluding required on real cases user interactions. Note, that all methods inside that module do not work in the production environment. Request to those methods on production will cause errors about not implemented methods.

### Raw proto files

You can find all proto files on the Code tab, but we recommend using the tools provided by buf.build to generate ready-to-use packages from the Assets tab. For instructions, please see the Assets tab.

### API Reference

Service descriptions are spread by packages, which include descriptions of messages and types. See Packages section below for more details.
