---
title: "REST in Peace? Django's Framework Problem"
date: 2025-03-25T11:07:12-04:00
draft: false
tags:
    - django
    - open-source
Summary: "The Django REST Framework maintainer has removed access to thousands of community discussions, leaving even other maintainers pleading for read-only access. What does this mean for Django's ecosystem sustainability?"
---
[Django REST Framework](https://www.django-rest-framework.org/), DRF, is a staple in the Django ecosystem with nearly 800,000 dependent repositories. It is the canonical package for nearly every Django application with a REST API. The maintainer has [recently disabled GitHub issues and discussions](https://github.com/orgs/encode/discussions/11) for the repository such that they can no longer be viewed. This has effectively removed access to thousands of discussions and cross-linkages for troubleshooting, bug reports, feature requests, and community-derived workarounds that countless developers could have stumbled upon when facing problems. Years of content the community has produced and relied on have been abruptly removed from the commons to such a degree that *even one of its own maintainers* is [pleading for read-only access](https://github.com/encode/django-rest-framework/pull/9660#issuecomment-2708169762). For contributors, what is the incentive to participate in this area of the open source community knowing your contributions can be wiped away in an instant?

The official response is:
> The issues are still there and we can re-enable them if needed. REST framework will still be accepting security updates and essential versioning changes.   
> ...   
> Currently I'm putting the volume dials to low to create less unnecessary churn.   

In [another thread](https://github.com/encode/django-rest-framework/pull/9660) he states:
> - I feel that the project's discussion working space doesn't present the kind of environment I'd like to be working in.
> - Much of the discussion and issue space isn't being properly directed at the moment and feels to me like its creating a distraction/drag on attention, rather than being genuinely valuable.

People have suggested that he could reinstate the issues in a read-only mode which he's [considering](https://github.com/encode/django-rest-framework/pull/9660#issuecomment-2702105443). 

The motivations here are hardly unusual; a dedicated maintainer creates something so successful that the community overwhelms them with requests, varying in levels of entitlement and relevance. The response, however, is unusual and adds another wrinkle to the issues surrounding open source sustainability. Even more worrying is that despite DRF being a cornerstone of Django development, the archival of more than a decade of community knowledge has generated little discussion in the broader community.

In the background, a competitor to DRF has been growing. [django-ninja](https://github.com/vitalik/django-ninja) has been making waves and offers an async-ready heavily type-driven alternative that will remind users of FastAPI.

Ninja, however, brings with it some operational concerns. It's less popular, which can make some hesitant to adopt it. It has over 300 open issues, almost 60 open pull requests, and is maintained by a single maintainer. The maintainer responds to some issues and merges PRs sporadically, but faith in the project has started to waver. A new fork, [django-shinobi](https://github.com/pmdevita/django-shinobi/) has emerged, which might lead to even more uncertainty from potential users.

There are [several](https://www.djangoproject.com/weblog/2024/oct/08/why-django-supports-the-open-source-pledge/) Django projects facing sustainability problems, and maintainers getting burnt out or overworked isn't something new, but this seems like a critical moment. How are teams evaluating Django supposed to justify it when the ecosystem is failing to sustain even basic REST framework development?