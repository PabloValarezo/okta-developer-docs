---
title: Interaction Code grant
meta:
  - name: description
    content: An overview of the Interaction Code grant type for Okta Identity Engine.
---

# Interaction Code grant

<ApiLifecycle access="ie" /><br>
<ApiLifecycle access="Limited GA" /><br>

## Overview

To enable a more customized user authentication experience, Okta has introduced an addition to the [OAuth 2.0 and OpenID Connect](/docs/concepts/oauth-openid) standard called the Interaction Code grant type. This grant type allows apps to manage user interaction with the Okta Authorization Server without a browser. This is useful when the client has a particular way that it wants to interact with the user and doesn’t need to share an authenticated session with other applications.

The Interaction Code flow consists of a series of interactions between the user and the Authorization Server, facilitated by the client. Each interaction is called a remediation step and corresponds to a piece of user data required by the Authorization Server. The client obtains these remediation steps from the Identity Engine component of the Authorization Server and prompts the user for the required data to continue the flow.

For example, a user could start an authentication flow by entering only a username, and this flow would prompt the client to request more information, or remediation, as required. Remediation is the direct communication between the client and the Authorization Server without a browser redirect. An example of a remediation step is the client prompting the user for a password or to add a second factor and then sending that information directly to the Authorization Server.

The Interaction Code grant is intended for developers who want a step-by-step remediation user experience without redirecting to an Authorization Server. This grant type enables developers to include [Identity Engine features](https://help.okta.com/en/oie/okta_help_CSH.htm#csh-features), such as passwordless authentication and progressive profiling. See [Redirect authentication vs. embedded authentication](/docs/concepts/redirect-vs-embedded/) for Identity Engine authentication deployment models and [Identity Engine deployment guides](/docs/guides/oie-intro/) for detailed deployment steps.

## The Interaction Code flow

The Interaction Code flow is similar to the [OAuth 2.0 Authorization Code flow with PKCE](/docs/concepts/oauth-openid/#authorization-code-flow-with-pkce). All clients are required to pass along a client ID, as well as a Proof Key for Code Exchange (PKCE), to keep the flow secure. Confidential clients such as web apps must also pass a client secret in their authorization request. The user can start the authorization request with minimal information, relying on the client to facilitate the interactions with the Okta Authorization Server to progressively authenticate the user. The series of interactions, which could include multifactor authentication steps, is secured using the `interaction_handle`. After successfully completing the remedial interactions, the client receives an `interaction_code` that the client can then redeem for tokens at the [`/token`](/docs/reference/api/oidc/#token) endpoint.

The following table describes the parameters introduced for the Interaction Code grant type flow:

| Interaction Code grant parameter           | Description   |
| --------------------------------           | -----------   |
| `interaction_code` |  The `interaction_code` is a one-time use, opaque code that the client can exchange for tokens using the Interaction Code grant type. This code enables a client to redeem a completed Identity Engine interaction for tokens without needing access to the authorization server’s session. |
| `interaction_handle` | The `interaction_handle` is an opaque, immutable value that is provided by the Okta Authorization Server. The client can use the `interaction_handle` to interact with the Okta Authorization Server directly. The client is responsible for saving the `interaction_handle` and using it for the duration of the transaction. Any public/confidential client that is configured to use the Interaction Code grant type can obtain an `interaction_handle`. If all the remediation steps are successfully performed, an `interaction_code` is returned as part of the success response.            |

The following sequence of steps is a typical Interaction Code flow with the Identity Engine:

<!--
See http://www.plantuml.com/plantuml/uml/

@@startuml
skinparam monochrome true
actor "Resource Owner (User)" as user
participant "Client" as client
participant "Authorization Server (Okta)" as okta
participant "Resource Server (Your App)" as app

user -> client: Start auth with user info
client -> client: Generate PKCE code verifier & challenge
client -> okta: Authorization request w/ code_challenge, scopes, and user info
okta -> client: Sends interaction_handle in 200 error response (for required interaction)
okta <-> client: Remedial interaction w/ interaction_handle
client <-> user: Remedial interaction
okta -> client: If the remedial interaction is successful, send interaction_code
client -> okta: Send interaction_code, client ID, code_verifier to /token
okta -> okta: Evaluates PKCE code
okta -> client: Access token (and optionally refresh token)
client -> app: Request with access token
app -> client: Response
@enduml

 -->

![Interaction Code flow sequence diagram](/img/authorization/interaction-code-grant-flow.png)

* The Interaction Code flow can start with minimal user information. For example, the user (resource owner) may only provide the client app with their username. Alternatively, the client can also begin the flow without any user information for a passwordless or a social sign-in experience.

* The client app generates the PKCE code verifier & code challenge.

* The client app begins interaction with the Authorization Server, providing any context it may have, such as a login hint, as well as sending the code challenge in a request for authorization of certain scopes to the Okta Authorization Server.

  > **Note:** A confidential client authenticates with the Authorization Server while a public client (like the Sign-In Widget) identifies itself to the Authorization Server. Both must provide the PKCE code challenge.

* The Authorization Server sends the `interaction_handle` parameter in the body of a 200 error response to the client app.

  > **Note:** The `interaction_handle` is used to continue the interaction directly with the Authorization Server. This is why the client, either confidential or public, needs to be registered with the Authorization Server to perform this direct interaction.

* The client sends the `interaction_handle` to the Authorization Server in Identity Engine and in return, the Authorization Server sends any required remediation steps to the client. The client begins an interactive flow with the Authorization Server and the resource owner, handling any type of interaction required by the Authorization Server (the remedial information is provided by the user to the client).

* When the remediation steps are completed, the Authorization Server sends back a success response and includes the `interaction_code` to the client.

  > **Note:** The Interaction Code has a maximum lifetime of 60 seconds.

* The client sends the code verifier and the `interaction_code` to the `/${authServerId}/token` endpoint to exchange for tokens.

  > **Note:** The `interaction_code` indicates that the client (and user) went through all of the necessary interactions and received a success response from Identity Engine.

* The Authorization Server authenticates the client and validates the `interaction_code`. If the the code is valid, the Authorization Server sends the tokens (access, ID, and/or refresh) that were initially requested.

* The client makes a request with the access token to your app.

* Your app sends a response to the client.