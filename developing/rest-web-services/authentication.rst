RESTful web service authentication
==================================

Indicia's web services provide support for a variety of different types of clients,
including:

  * Websites with online recording and reporting facilities.
  * Mobile applications.
  * Other databases which synchronise data with the Indicia instance.
  * Users of the data.
  * Developers working on site building or report writing.

In order to meet the needs of the different types of client, there are several
options for approaches to authentication.

  * A website or application might wish to authenticate as the registered website on the
    warehouse and will therefore gain access to all records contributed  via that website
    or shared with the website for reporting purposes.
  * A mobile application user might authenticate as that specific user/application
    combination to gain access to just their records.
  * A remote database which synchronises with the warehouse needs to identify itself in
    order for the warehouse to ensure the correct filtered set of records is made available
    to it.
  * A user of the data needs to identify the filtered set of data they have permission to
    view.
  * A developer will want to be able to simulate all these options without the
    authentication process hindering the simplicity of development.

Given the above requirements, Indicia provides 3 types of identification that
can be used when accessing the web services:

#. The warehouse has a list of website registrations in the websites table. This describes
   each of the online recording websites that contribute records to the warehouse.
   Authentication of web service requests can be achieved by providing a website_id and
   using the website's password as the secret for authentication. This gives the client
   access to the records belonging to the given website registration as well as any which
   are shared with the website via warehouse configuration.

   .. tip::

     Before giving a client the website_id and password for a website registration,  bear
     in mind this grants them full permissions to read and write that website's data which
     will not be appropriate in many cases. It also means you cannot disable that client's
     access without also disabling your website or application's access since they share
     the same authentication details. A better way is to create a "client"  for your data
     users in the configuration file as described below and create a project which enables
     them to access the website's records without a filter.

#. The warehouse also has a list of user accounts which can be used for
   authentication. Authentication of web service requests can be achieved by
   providing a user_id and website_id and using the user's warehouse password as
   the secret for authentication. This gives the client access to the user's
   records which they've contributed to the website identified by the given
   website_id. The user must of course be a member of the website. Optionally,
   provide a filter_id for a filter linked to the user which has
   defines_permissions=t (e.g. a filter granting verification rights) to give
   the client access to the filtered set of records.
#. The REST API has a list of clients in its configuration file which
   are being given access to the web services. Clients listed in the configuration file
   can be other systems (e.g. other online recording databases) or could equally be a user
   of the data. Provide the client ID and use the configured secret for authentication.
   Each client has a number of projects defined in the configuration which define filtered
   access to records for a given website, for example a project might be the Odonata
   records available to the iRecord website registration.

The following methods of authentication using these 3 categories of client user
are available for the REST API:

oAuth2
------

The web services support the password grant flow only. By default this is
possible over an https connection only though in development environments you
can configure the REST API's `http_authentication_methods` setting to change
this. Note that a client ID is automatically recognised for every registered
website on the warehouse, in the format "website_id:<id>" replacing <id>
with the numerical website ID.

Example:

#. POST to index.php/services/rest/token with the following POST data, replacing
   the values in <> as appropriate:
   grant_type=password&username=<user>&password=<password>&client_id=website_id:<n>
#. The response is a JSON object with a value for access_token, e.g.:
   ``{access_token:"12345",token_type:"bearer",expires_in:7200}``
#. The bearer token type indicates we can supply the access token in subsequent
   requests by setting the Authorization header to "Bearer <access_token>",
   e.g. call index.php/services/rest/reports with the header set to
   "Authorization: Bearer 12345".

.. tip::

  Accessing via oAuth in this way grants can be configured to grant access only to the
  user's records on the associated website registration given in the token request. This is
  done by setting the 'limit_to_own_data' option for the reports resource as illustrated in
  the Rest API's example configuration file. In this instance, in order to grant access to
  a wider set of records you can create a filter (in the filters table) and link it to the
  user, with defines_permissions=t. This grants the records identified by the filter to the
  user when using that filter ID. The ID of the filter to use can be passed in a query
  parameter in the URL called filter_id.

HMAC
----

This approach to authentication relies on the client process using a shared
secret to build a hash value using the URL plus all the data values supplied in
the request. The hash (HMAC, or keyed-hash message authentication code) is
provided with the request but not the secret. The server side can then hash the
request's data with the secret (which it also knows) to generate the HMAC. If
they match then the request is authentic. Although not as widely recognised as
oAuth2, this approach does provide some protection when using http rather than
https since the secrets are never passed between the client and server. It also
has the advantage of being genuinely stateless and therefore RESTful.

In more detail:

#. The requesting entity creates a HMAC-SHA1 value of the complete request url
   (including parameters). The hash value uses the user password as the shared secret.
#. The requesting entity adds an Authorization header to the request containing the
   following string [user type]:[user identifier]:HMAC:[hmac] where:

     * [user_type] is a token which identifies whether a registered website (WEBSITE_ID),
     * warehouse user account (USER_ID) or client defined in the REST API's configuration
       file (USER).
     * [user identifier] is the requesting client's identifier, either the website_id,
       user_id or client ID as described above.
     * [hmac] is the HMAC-SHA1 value computed in (1)

   When identifying as a warehouse user it is also necessary to provide the
   website ID in the authentication string as follows, since a single user
   account can have access to several website registrations::

      USER_ID:[user id]:WEBSITE_ID:[website id]:HMAC:[hmac]

.. tip::

   When identifying as a warehouse user it is possible for them to have their
   permissions extended by having an administrator create a filter for them
   which has the defines_permissions flag set to true, just as you can with
   oAuth authentication. The ID of the filter to use can be passed in a query
   parameter in the URL called filter_id.

#. The receiving entity recomputes the HMAC-SHA1 in the same manner as (1) and any
   authorisation failure is returned as HTTP 401 Unauthorized.

This authentication should provide suitable protection against tampering and sufficient
level of authentication providing the shared secret is sufficiently long.

The following example PHP snippet illustrates the code required for authentication against
the REST API as a client described in the REST API's configuration file:

.. code-block:: php

  <?php
  $shared_secret = 'mypassword';
  $userId = 'ME';
  $url = 'http://www.example.com/rest/projects';
  $session = curl_init();
  // Set the POST options.
  curl_setopt ($session, CURLOPT_URL, $url);
  curl_setopt($session, CURLOPT_HEADER, false);
  curl_setopt($session, CURLOPT_RETURNTRANSFER, true);
  // Create the authentication HMAC
  $hmac = hash_hmac("sha1", $url, $shared_secret, $raw_output=FALSE);
  curl_setopt($session,
      CURLOPT_HTTPHEADER,
      array("Authorization: USER:$userId:HMAC:$hmac")
  );
  // Do the request
  $response = curl_exec($session);
  $httpCode = curl_getinfo($session, CURLINFO_HTTP_CODE);
  $curlErrno = curl_errno($session);
  // Check for an error, or check if the http response was not OK.
  if ($curlErrno || $httpCode != 200) {
    echo "Error occurred accessing $url<br/>";
    echo "Rest API Sync error $httpCode<br/>";
    if ($curlErrno) {
      echo 'Error number: '.$curlErrno;
      echo 'Error message: '.curl_error($session);
    }
    throw new exception('Request to server failed');
  }
  $data = json_decode($response, true);
  ?>

Direct authentication
---------------------

HMAC authentication never require's the user's secret or password to be passed
across the connection between the client and server so is inherently secure and
it does not require a secure connection (https) to ensure the authentication
details cannot be sniffed. When a secure connection is available over https, or
when developing code so security is not a concern, it can be simpler to pass
a password to the authentication process directly without calculating an HMAC.
Note that the default configuration of a warehouse is to disallow directly
passing a password or secret to the REST API authentication so this needs to be
changed in the REST API's configuration where appropriate. See
:doc:`../../administrating/warehouse/modules/rest-api` for more information.

When using direct authentication, the process is the same as for HMAC but you
set the password or client shared secret in the authentication string
as in the following example (using the token SECRET instead of HMAC)::

  USER_ID:[user id]:WEBSITE_ID:[website id]:SECRET:[user password]
