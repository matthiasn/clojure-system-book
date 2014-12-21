# Server-side: TwitterClient

## Architectural Overview

The purpose of the TwitterClient application is to maintain a streaming connection with the Twitter Streaming API, restart this connection if necessary, persist received tweets and make tweets available on a Redis Pub/Sub.

There can only be one instance of this application at any one time because Twitter does not allow you to start multiple clients at the same time.