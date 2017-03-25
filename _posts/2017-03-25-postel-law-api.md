---
author: jlordiales
comments: true
share: true
date: 2017-03-25
layout: post
title: Why you should follow the robustness principle in your APIs
tags:
- API
- Postel law
- Microservices
---
Microservices are all the rage right now. Everyone is taking their big monoliths
and decomposing them into smaller services with exposed APIs. If you are doing
this right then your services should be completely decoupled and independently
releasable. Yet the way some APIs are designed makes this extremely hard to
accomplish, if not impossible.
Let's take a look at the problem and how to solve it.

# Postel's Law
Postel's Law, also known as the [robustness
principle](https://www.wikiwand.com/en/Robustness_principle) states that you
should be conservative in what you send and liberal in what you accept from
others.  Although this was proposed initially for the specification of the TCP
protocol, it has a very important place in the design and evolution of APIs and
we'll see why with an example.

Imagine we have 2 small, independent services owned by different teams: the user
service and the address service. The address service exposes a small API to
store users' addresses, which initially only consists of the street address.  The
user service can POST a request with a JSON body like `{"street_address": "1234
Fake St."}` and the address service responds with a 201 - Created.

So far so good. However, we quickly realize that street address alone is not
enough information so we want to start sending in city and postal code as well.
Unfortunately, the address service does not follow the robustness principle we
mentioned before because if it happens to receive any field on the request that
it doesn't recognize (anything other than street_address at the moment) it
immediately fails, returning a 400 - Bad Request

The user service team is done with their part of the work needed to start
sending the new fields but they can not release it until the address service is
updated to take in those fields.  We effectively introduced a major coupling
point between the 2 services that adds the need to sync their releases. This is
even worse if we have multiple environments with potentially different versions
of our services deployed and/or multiple clients of the address service.

Of course, one approach to mitigate this would be for the user service to have
this new functionality behind a [feature
flag](https://martinfowler.com/bliki/FeatureToggle.html). Then they can do their
work and release to any environment with the flag disabled. Once the address
service is updated the new feature can be enabled.
Not a big problem, but each feature flag does introduce some extra complexity to
the code. What happens the next time we want to add another field? Will we have
one flag for each field that we want to send to the address service?

Now imagine that both services are running with the latest changes, the feature
flag is enabled and we are storing the data we wanted. It's Saturday 11 PM
and consumers start calling in complaining that their address data is completely
messed up. They are getting data from other people, which is not only a terrible
user experience but a potential privacy disaster.
It turns out that the latest release of the address service with the addition of
the 2 new fields also included a nasty bug that caused the data to be assigned
to the wrong users.
The address team decides to rollback the release immediately to the latest known
functioning version (the one that had only street_address on the API).

There are 2 possible scenarios at this point. Scenario 1 is that they rollback
their service without realizing that the user service is still sending the new
fields. This causes all new requests after the rollback to fail with a 400,
effectively loosing all new data.
Scenario 2 involves someone contacting the user team for them to disable the
feature flag (if it's still there) before doing the rollback.
The result is equally bad in either case. You introduce unnecessary dependencies
between teams and unexpected bugs.

This whole thing could have been avoided if the address service would've
adhered to the robustness principle and simply ignored all unknown fields.
The user service then could've started sending the new fields in all
environments without worrying about when the address service was ready.
Similarly, in the production incident scenario, the address service can safely
rollback and the user service team doesn't even have to know about it. No
coupling, no synchronization between teams, no hassle.

Note that you can still enforce the presence of mandatory fields. The only thing
we've done is to make sure we just ignore the fields we don't care about.

This is very easy to do if you are using Jackson for instance, either globally
by configuring you object mapper like:
{% highlight java %}
new ObjectMapper()
  .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
{% endhighlight %}

or on a class by class basis, with a simple annotation:

{% highlight java %}
@JsonIgnoreProperties(ignoreUnknown = true)
public class Address {
}
{% endhighlight %}

Other languages and serialization/deserialization frameworks should provide
similar options.
If you are using JSON schema to validate your incoming requests you can set
[additionalProperties](http://json-schema.org/latest/json-schema-validation.html#rfc.section.5.18) to true.

# API versioning
Of course this doesn't mean that your service should accept completely invalid
input and try to make sense of it. As they say, keep an open mind but not so
open that your brain falls off.

Adding new optional fields to an API, like we saw in the previous example, is a classic
example of a change that is perfectly backwards compatible. Your service should
be able to handle both the presence and the absence of those fields.
Other times, however, there's a fundamental change in the structure of your API.
In these cases it's a lot harder, if not impossible, to keep backwards
compatibility. 

This is one of the main reasons why it's a good idea to have versioning of your
APIs from day one. You never know how your service will need to evolve in the
future so it's better to start with the assumption that you will need versioning
at some point. 
You have several alternatives to handle API versioning (URL vs Content-Type
header for instance). Which one you choose will depend on your particular needs
and constraints.

# Also important for consumers of APIs
So far we've been assuming that this applies only to providers of an API in what
they accept from their clients. But it is equally important for consumers as
well.

When you consume an external API, most of the time you are not interested in
every single field of the response. By making sure that you ignore any field you
don't have an interest in, you are not only writing less code but you are also
making your service more resilient to changes in that API.

We can go back to the user and address service example from before, but this
time looking at things from the point of view of the user service. In the same
way that the address service provides an endpoint to POST data to, it also
provides an endpoint where clients can GET address details. In the response the
address service includes the street address, city, latitude and longitude.

The user service is only interested in the street address and city, it has no
use for latitude and longitude. But for some reason it still maps every field
of the response. With time the address service adds more fields to the response
because other consumers need those. Every new addition involves extra work for
the user service team, even when they still only care about the original street
address and city.

To make things worse, the address service doesn't really sync with its clients
before doing backwards compatible changes (and why should they?). So the user
service only realizes that they have to adapt to a new response after things
start failing on their side.

# Is it ever worth it?
So why would you want to be really conservative in what you accept and fail all
requests that have any additional field? 
I haven't found a real-life use case where the benefits from doing so outweigh
the problems we discussed before.
I suspect most services that actually do this are doing it because they just
haven't thought about it and they use the default behaviour of their
language/tool. In the case of Jackson this default behaviour is to fail the
deserialization when extra fields are present.

Some people argue that making the request fail early can make it explicit to the
consumers of their API that they are doing something wrong. I don't really find
that argument compelling enough. This sort of issues should be detected through
testing, either [consumer driven
contracts](https://martinfowler.com/articles/consumerDrivenContracts.html) or
integration/E2E tests.

# Conclusion
Postel's law, or the Robustness principle, is essential in the evolution of
APIs. No one can accurately anticipate how your requirements are going to change
or how many different number and types of consumers you are going to have. By
making sure that you are lenient and ignore the fields you don't really care
about for specific requests you will be decoupling your service from your
consumers. This is fundamental in a micro-service environment where the
dependencies between services should be kept to a minimum, especially when
considering releases.
