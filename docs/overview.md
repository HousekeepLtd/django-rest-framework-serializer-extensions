# Overview
Serializer extensions reduces the need for *very similar* serializers, by
allowing the fields to be defined on a *per-view/per-request* basis. Fields can
be whitelisted, blacklisted, and child serializers can be optionally expanded.
Whatever fields you choose to use, your querysets can be optimized
automatically to make the fewest database calls possible.

Support for [HashIds](https://github.com/davidaurelio/hashids-python) is
[also provided](usage-hashids.md). If you're currently exposing your internal IDs over a public
API, we suggest you consider switching to HashIDs instead.


## Requirements
* Python (3.10, 3.11, 3.12, 3.13)
* [Django](https://github.com/django/django) (4.2, 5.0, 5.2)
* [Django REST Framework](https://github.com/tomchristie/django-rest-framework) (3.14, 3.15, 3.16). Note Django REST Framework 3.16 is the first version to
  officially support Django 5.2.
* [HashIds](https://github.com/davidaurelio/hashids-python) (>1.0)
