# Introduction to Redis:

This a 3 parts articel introducing `Redis` and how it can be used to solve different problems. The first part I'll use Redis to cache the landing page database query.
[The second part]() is another use case where I'll use Redis to calculacte most-viewed articles and retrieve them effecientely without bothering the database too much.
Part-3 I'll refactor the code introduced in the previous parts.
All the examples are implemented in a multi-user blog I made while learning Node.js (Express.js). **Remember** this is only for learning purposes and not by anyway the most efficient way to serve blogs.

## Part 1: Redis for caching

Redis is defined as in-memory data-strucute and it's well known for being balnzingly fast. First thing you should know exactly what [data-structre](https://redislabs.com/redis-enterprise/data-structures/) is most convienet to solve your problem. To make this decision we need to know how your application will ferquently access them (reading, inserting, searching) and the data-structure performance. Redis has a verstaile data-strucure with different performance metrics as we will see.

### My use case:

Here is the contoller code before adding Redis:

<script src="https://gist.github.com/3omer/da4271554d3a050817219d3aa8a64070/095530fc4177acf0194e3c3a20a9a919b091a5b8.js?file=main.js"></script>

The query I'm interested in is the one that load the most recent articles to render the home page. For this I used Redis's `sorted list` data-strucre which is a sorted by `score` value. Every member in the list has a corrsponding `score` value which I set to the article's `createdAt` atribute.

I've seen some [articles]() that extends the mongoose query object and set the code responsible to deal with cahcing in a pre-query hook, but It's kind of complicted if you are not familiar with prototyping and inheritance, eventhough it's less repetative.
However I prefere to seprate the caching code and be more exmplicit about how we access data, I created the following module `access.js` to manage accessing to `Articles` model.

But before everything we need to set-up a redis client as in the following

<script src="https://gist.github.com/3omer/da4271554d3a050817219d3aa8a64070/027943637f2be071a3f406968384a2752e9cf46a.js?file=redis.js"></script>

<script src="https://gist.github.com/3omer/da4271554d3a050817219d3aa8a64070/027943637f2be071a3f406968384a2752e9cf46a.js?file=access.js"></script>

The [caching strategy](https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one/) i'm using is called `Cache-Aside` as implemented in `getRecentArticles()`. It's simple, `getRecentArticles()` first checks redis for articles, if it is not there it fallbacks to the database, and cache the result for subsequent reads.

Then modify the home controller to use `getRecentArticles()` instead of hitting the DB on every request.

<script src="https://gist.github.com/3omer/da4271554d3a050817219d3aa8a64070/027943637f2be071a3f406968384a2752e9cf46a.js?file=main.js"></script>

Whenver you are using caching you must pay attention to the data-consitency between the cache and the database. For example `recentArtilces()` will contains invalid data if one user submited a new article.
Now one obvious way to mitigate this is to update the cache or erase it whenever a user submit an article, orrr .. we can serve the staled data for 5 minutes then jsut erase the cache and call it a day? I hope no user will notice the recent articles list doesn't contain articles posted in the last 5 minuts ? :), I aslo hope you can see where i'm going with this.
If your application is not very critical about data-consistency you can just set a `TTL` -time to live- on data to expire the cache and force the application to refresh it from the DB again.
Otherwise, if data consistency is very critcial for your app to function, then maybe you should use a different strategy.

## Redis as sessoin store:

Redis is also consided the go-to solution for user's session. Performance wisely you don't want to execute a query on every user's interaction, and if the client's session is chaning a lot that is going to be a serious performacne damage.

## Load testing:

Finally I tested the impact of the changes I introduced -caching + session store- by running a load test using `apache-benchmark` an easy to use tool for load testing. These results are from the landing page only.

- ### Testing results before using redis:

  These are the server throughput readings

  <table class="ui celled table">
    <thead>
      <tr><th># conccurent req. </th>
      <th>Before Redis</th>
      <th>After Redis</th>
    </tr></thead>
    <tbody>
      <tr>
        <td data-label="# conccurent req.">100</td>
        <td data-label="Before Redis">163.41</td>
        <td data-label="After Redis">303.83</td>
      </tr>
      <tr>
        <td data-label="# conccurent req.">120</td>
        <td data-label="Before Redis">146.15</td>
        <td data-label="After Redis">336.46</td>
      </tr>
      <tr>
        <td data-label="# conccurent req.">160</td>
        <td data-label="Before Redis">137.31</td>
        <td data-label="After Redis">324.15</td>
      </tr>
      <tr>
        <td data-label="# conccurent req.">200</td>
        <td data-label="Before Redis">129.90</td>
        <td data-label="After Redis">335.71</td>
      </tr>
    </tbody>
  </table>

## Guide lines:

Finally a general guide lined when it comes to cahcing:

- identify the queries you want to cache
- Choose the convient data structre optmized for the operation you need (time complexity eg. O(1) access time, sorting, .. etc)
- Keep data consistant. (TTL, Write through, read through)
