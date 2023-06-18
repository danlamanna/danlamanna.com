---
title: "Building Search DSLs with Django"
date: 2023-06-17T07:16:18-04:00
draft: false
tags:
    - django
    - search
---
Search capabilities span from free text (think Google) to raw data access (think SQL). In between, there's a wide range of options for narrowing a search that are often provided with UI elements. But what if there are too many fields for a UI to search on? Search DSLs can give a user more granular access to searching without exposing an overly complicated interface.

GitHub issues provide a DSL that's accompanied by UI elements. An example query for searching issues would be:
{{< highlight text >}}
is:open author:danlamanna
{{< / highlight >}}

We can create something similar for use in a custom Django application. Taking this example means building a DSL that:
1. Supports filtering by a status of either *open* or *closed*
2. Supports filtering by an author username
3. Implicitly conjoins the status and author filter (AND them together)

## Enter PyParsing

[PyParsing](https://github.com/pyparsing/pyparsing) is a fantastic open source library for creating parsers. We can use it to build the "open or closed" part of this by creating a `pyparsing.ParserElement`.

{{< highlight python >}}
import pyparsing as pp

pp.one_of('open closed')  # ParserElement

pp.one_of('open closed').parse_string('foo')
# ParseException: Expected open | closed, found 'foo'  (at char 0), (line:1, col:1)

pp.one_of('open closed').parse_string('open')
# ParseResults(['open'], {})

# pyparsing ignores whitespace by default
pp.one_of('open closed').parse_string(' open   ')
# ParseResults(['open'], {})
{{< / highlight >}}

This lets us take an input string and return a set of `ParseResults` that we can build database queries from! Since we need to only consider an open or closed status after `is:` we need to create a `Keyword`. A `Keyword` is documented as a "Token to exactly match a specified string as a keyword, that is, it must be immediately followed by a non-keyword character." We can also use a `Literal` for the colon.

{{< highlight python >}}
Is = pp.Keyword('is')
OpenClosed = pp.one_of('open closed')
StatusTerm = Is + pp.Literal(':') + OpenClosed

StatusTerm.parse_string('is:closed')
# ParseResults(['is', ':', 'closed'], {})
{{< / highlight >}}

This is almost perfect, and it demonstrates how easy it is to build up a parser from composable parts. One problem is that our parse results contain the colon even though that won't be necessary for our final query. PyParsing offers a `Suppress` converter for ignoring specific tokens after parsing.

{{< highlight python >}}
StatusTerm = Is + pp.Suppress(pp.Literal(':')) + OpenClosed
StatusTerm.parse_string('is:closed')
# ParseResults(['is', 'closed'], {})
{{< / highlight >}}

Perfect!

## Querying in Django
Django has composable lookup objects for building queries called [Q objects](https://docs.djangoproject.com/en/4.2/topics/db/queries/#complex-lookups-with-q-objects). This means that `Q(foo='bar')` roughly translates to `WHERE foo = 'bar'` and `Q(foo='bar') | Q(baz='qux')` to `WHERE foo = 'bar' OR baz = 'qux'`. Since Q objects operate at the Django ORM layer, they're also resilient to injection attacks. Putting this together with our parser, we can generate `Q` objects from our input string.

This should look like:
| search                      | Q object                                    |
|-----------------------------|---------------------------------------------|
| is:closed                   | Q(status='closed')                          |
| is:closed author:danlamanna | Q(status='closed') & Q(author='danlamanna') |

Our `StatusTerm` currently returns a list of tokens in the `ParseResults` object. Using "parse actions" we can make it return the Q objects we want.

Setting a parse action for a parser element is as simple as:
{{< highlight python >}}
from django.db.models import Q

@StatusTerm.set_parse_action
def status_to_q(results):
    # results = ['is', 'closed']
    return Q(status=results[1])
    
StatusTerm.parse_string('is:closed')
# ParseResults([<Q: (AND: ('status', 'closed'))>], {})
{{< / highlight >}}

Now we can create a similar parser element for the author in our DSL.
{{< highlight python >}}
AuthorTerm = pp.Keyword('author') + pp.Suppress(pp.Literal(':')) + pp.Word(pp.alphanums + '_')

@AuthorTerm.set_parse_action
def author_to_q(results):
    # results = ['author', 'danlamanna']
    return Q(author=results[1])
{{< / highlight >}}

This is similar to the `StatusTerm` with the exception that we accept a PyParsing `Word` for the value instead of an enumerated set of statuses.

These elements are easy to compose together, so now we can build our final parser:
{{< highlight python >}}
dsl = pp.OneOrMore(pp.Or([StatusTerm, AuthorTerm]))
dsl.parse_string('author:danlamanna is:closed')
# ParseResults([<Q: (AND: ('author', 'danlamanna'))>, <Q: (AND: ('status', 'closed'))>], {})
{{< / highlight >}}

## 🤯 
`pp.Or` says match *any* of the parser elements we give e.g. require the string to specify a status OR an author, not both. `pp.OneOrMore` matches one or more of those terms, which is what lets the search specify a status and an author.

Now we can parse any combination of strings that contain at least one of our search terms, and we'll get a list of `Q` objects that can compose with the bitwise AND operator to create a single `Q` object for the Django ORM.

Assuming our model is called `Issue`:
{{< highlight python >}}
import functools, operator

search_str = 'author:danlamanna is:closed'
filter_obj = functools.reduce(operator.and_, 
                              dsl.parse_string(search_str).as_list())
# <Q: (AND: ('author', 'danlamanna'), ('status', 'closed'))>
                              
# fetch all issues from the database that have a status of closed and an author of danlamanna
issues = Issue.objects.filter(filter_obj)
{{< / highlight >}}

This is a simple parser that only works in the most obvious cases, but we could build on it in a number of ways:
- Add validation   
  For instance, maybe we want to prevent users from searching for invalid author names. PyParsing supports adding a parse action for raising errors, see `add_condition`.
- Handle duplicated terms   
  Each DSL might want to handle this differently. Ours ignores duplicated terms without raising any sort of exception.
- Add disjunction (OR) logic   
  Even allow combining AND with OR, see, for instance how `infixNotation` in PyParsing works. This even supports nesting expressions with parentheses!
- Add wildcards   
  There's no reason we can't translate certain tokens as having wildcards and return an ANDed `Q` object e.g. `author:dan*lamanna` => `Q(author__startswith='dan') & Q(author__endswith='lamanna')`

Our final parser, with the example query, looks like this:
{{< highlight python >}}
from django.db.models import Q
import pyparsing as pp
import functools, operator

Is = pp.Keyword('is')
OpenClosed = pp.one_of('open closed')
StatusTerm = Is + pp.Suppress(pp.Literal(':')) + OpenClosed

@StatusTerm.set_parse_action
def status_to_q(results):
    return Q(status=results[1])

AuthorTerm = pp.Keyword('author') + pp.Suppress(pp.Literal(':')) + pp.Word(pp.alphanums + '_')

@AuthorTerm.set_parse_action
def author_to_q(results):
    return Q(author=results[1])

dsl = pp.OneOrMore(pp.Or([StatusTerm, AuthorTerm]))
search_str = 'author:danlamanna is:closed'
filter_obj = functools.reduce(operator.and_,
                              dsl.parse_string(search_str).as_list())

# fetch all issues from the database that have a status of closed and an author of danlamanna
issues = Issue.objects.filter(filter_obj)
{{< / highlight >}}
