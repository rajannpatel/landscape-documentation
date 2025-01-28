(reference-release-notes-what-s-new-in-beta)=
# What's new in beta

Here's what's new in Landscape beta:

## Landscape Server

landscape-server 25.04~beta.3 published 29 November 2024
  * fix: bug preventing account switching in new UI
  * fix: configure a randomly-generated default `cookie-encryption-key` during
    startup if not provided
  * fix: redirect security to untrusted

landscape-server 25.04~beta.2 published 19 November 2024
  * fix: address inconsistent Account object updates on subdomain-aware
    API endpoints

landscape-server 25.04~beta.1 published 15 November 2024
  * feat: grpc server sends the request_id parameter
  * feat: standalone OIDC uses new dashboard flow
  * feat: activity requests are associated with child instance profiles
  * feat: grpc server accepts `SendCommandStatus` and publishes messages
  * feat: hostagent consumer processes command status messages
  * feat: set [api] cookie-encryption-key during standalone install/upgrade
  * feat: process account subdomains in middleware for API service
  * feat(appserver): redirect requests to invalid subdomains to the canonical url
  * feat: Add new API argument to return active and inactive interfaces
  * feat: traditional profiles are not applied to windows machines
  * fix: restrict GET /tags to caller's account
  * fix: handle extra emails with SSO
  * fix: remove serve-dashboard configuration
  * fix: allow parsing boolean config values
  * fix: parse boolean config values
  * fix: allow clearing tags of child instance profiles
  * fix: do not set JWT in local storage
  * fix: accept LANDSCAPE_SESSION_JWT cookie for auth
  * fix: allow standalone bootstrapping during OIDC flow
  * fix: include patch for improper PyCurl CA path in landscape-api client
  * fix: add request id to pending computer
  * fix: update link to script execution doc page
  * fix: route main button on /signup to /create-new-account

