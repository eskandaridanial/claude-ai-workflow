I want to build a new service that tracks API rate limit usage across our
internal microservices.

The idea: every service calls a bunch of external APIs (payment providers,
SMS gateways, etc.) and each of those has its own rate limit. Right now
everyone tracks this themselves, inconsistently, and we've had a few
incidents where we got throttled or banned because nobody knew we were
close to the limit.

I want a central place where:
- Services report their calls to an external API
- We can see, per external API, how close we are to the limit
- We get warned before we hit it

This would be a brand new service, nothing existing to build on.

Not sure about a lot of things yet:
- Should services push usage data to this service in real-time, or should
  this service poll something?
- What counts as "close to the limit" — is that a fixed threshold or
  configurable per API?
- Who gets warned, and how? Slack? Email? Just a dashboard?
- Do we need this to be highly available, or is it okay if it's down
  sometimes (since it's not blocking actual API calls, just tracking)?
- Stack-wise I'd lean Java/Spring Boot since that's what we know, but open
  to something simpler if it makes sense.