Introduction to caching:

- how it effects performance:
- caching types/level: server(statics), query caching
- benchmark the endpoints you want to intoduce caching in. ( endpooints that performs queries, request to 3d part API .. .etc)
- select a conveniant redis data-structure (hash, list, sorted list)
- strategy to maintain cache consistancey (for example when a resource is created the cache became staild), in non critical data seting expiration key on cache data do the trick.
- benchmark the endpoints after the caching and measaure improvements.

In this article im gonna explaing how i used redis as a cache layer to reduce response latency introduced by database queries and as a result increased the server throughput.

Redis is defined as in-memory data-strucute so first thing you should know exactly what data-structre is most convienet for your problem.
My use case:
The query I'm interested in is the one that load the most recent articles to render the home page. For this I used Redis's `members` data-strucre which is a sorted list. Every member in the list has a corrsponding `score` value which I set the article createdAt date atribute.

Here is the old contoller code

I've seen some tutorials that prefer to extend the mongoose query object and set the code responsible to deal with cahcing in a pre-query hook, but It's kind of complicted if you are not familiar with prototyping and inheritance, but it's less repetative.
However I prefered to seprate the caching code, I created the following module
[
access.js
]

the function `getRecentArticles` first checks redis for articles, if it not there it fallbacks to the database, and cache the result for subseequent reads.
The same goes for `getArticle` the difference is this function is used to retrive one article from the cache.

Then modify the home controller to use `access.recentArticles`.

Whenver you are using caching you must pay attention to the data-consitency between the cache and the database. For example `recentArtilces` will contains invalid data if a user submited a new article.
Now when stratgey to mitigate this is to invalidate the cache, delete or update it whnever a write ooperastin happens, that if it's very crictical to keep the consistency with no delays. However there are use case when the delay is not very critical, as in this use case it's enough to just delete the cache every 2 minutes or so.
[

]

Redis as sessoin store:
Redis is also consided the go-to solution for user's session. Performance wisely you don't want to execute a query on every user's interaction, and if your session is chaning a lot that is going to be a serious performacne hit.

Load testing:
Finally I tested the impact of the changes I introduced by running a load test using apache-benchmark an eay to use cli tool for load testing.

Testing results before using redis:
[

]

Results after Redis:
[

]

Guide lines:
Finally a general guide lined when it comes to cahcing:

- identify the queries you want to cache
- Choose the convient data structre optmized for the operation you need (O(1) access time, sorting, .. etc)
- Keep data consistant. (TTL, Write through, read through)
