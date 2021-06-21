Intoduction to Redis - Part-2

This is another good example on how to utilize Redis to add feature that if we have used the database, it would negatively affect the performance.

## Top articles

The feature I'm trying to introduce in the blog is to serve the top 10 most-viewed articels. Well a simple solution is to add a counter attribute on the Article's model and increament that counter on every reading. To get the top views we will then just sort by the counter descendentely and slice the first 10 items. Easy .. and can be done in a few lines ? Yes. Sure. But no, we won't approach the problem like that because I like to make people suffer .. . joking, becuase it's for learning purposes we are faking scenarios here, assume there is a great traffic on the blog our readers can't get enough, they are reading and reading, non-stop all the day, our database server is already on fire catching up with _more important_ data, and we do not want to hit the database to increament an article views counter on every read. **Remember** these are just assumbtion, I don't know that much about database engines and how much they can handle.

## Redis to the resuce

I hope you've read [the previous part]() of this blog, because this sounds like a sorting problem again, and we are going to throw some Redis magic on it again. So obviously we will use the [sorted-set](), and you guess what the score value should represent this time ? .. The score value will be the views count. Remember we already have a list with every article in the database, and now we will make another similar list with different score? this is very redundant. This time the set members are just the articles' ids.

```
// cosider this reprsent Redis sorted-set
// a member is an article's id and the socre is the views counter:
"blogs:views": [ { member: 123abc, socre: 2 }, { member: 456def, socre: 10 } ]

```

So the functionality we need to code :
1- a function to increament the score in the `blogs:views` when an article is viewed.
2- a function to retrive the top N **ID** `blogs:views`.
3- query the db for those IDs.
4- CACHE the result for subsequent read.

```javascript
const redisClient = require('../redis')

const incArtilceViews = async (artilceId) => {
  const KEY_ARTICLES_VIEWS = 'blogs:views'
  redisClient.zincrby(KEY_ARTICLES_VIEWS, 1, artilceId, (err, rep) => {
    /* error hndling */
  })
}
```

This is the first bulding block, we will use this function in the controller code that retrieve article, the `zincrby` [command](https://redis.io/commands/zincrby) increaments the score by 1, simple.

```javascript
const getArticlePage = async (req, res, next) => {
  try {
    const art = await access.getArticle(req.params.id)
    if (!art) return next(404)

    art.content = marked(art.content) // render markdown

    // increment this artilce views
    incArtilceViews(art.id)

    return res.render('article', { article: art, comments })
  } catch (err) {
    next(err)
  }
}
```

Now we are sure we are counting, we will use `zrevrange` [command]() to get the top N IDs:

```javascript
const zrevrangeAsync = promisify(redisClient.zrevrange).bind(redisClient)
// return articles' ids list ordered by views count
const mostViewdArticlesIds = async (limit) => {
  const REDIS_KEY_ARTICLES_VIEWS = 'blogs:views'
  const LENGTH = limit || 10
  const articlesIds = await zrevrangeAsync(
    REDIS_KEY_ARTICLES_VIEWS,
    0,
    LENGTH - 1
  )
  logger.info('getMostViewed: result', articlesIds)
  return articlesIds
}
```

calling `mostViewedArticles(limit: Number)` with `limit=5` will returns a list of 5 ids, these represent the 5 most viewed articles, If the limit argument is left, then it default to 10.

OK. Now we can count views and get most-viewed _articles' IDS_, one last step is to convert those ID's to their corresponding articles:

```javascript
const getMostViewedArticles = async (length) => {
  // get new top articles id's
  const articlesIds = await mostViewdArticlesIds(LENGTH)
  // CAUTION: they are not sorted
  const articlesRandomOrder = await Article.find({ _id: { $in: articlesIds } })
  // order the articles list retrived from to db to match the correctly ordered id's list
  topArticles = articlesIds.map((id) =>
    articlesRandomOrder.find((article) => article.id === id)
  )

  return topArticles
}
```

When we query mongodb for example the IDs [id1, id2, id3], it's not guranted that mongo will return the items in the same order as in the query, the items will be in a random order, and so I re-ordered them to match the correct order we already got from Redis.
Now we can use this function in the conroller and call it a day, but there is one last trick to pull on this, we don't want to even query those few ID's, we would like to cache the top articles also. So a slight change to the function above is and you'll probably recognize this pattered :
First: check if top articles already in cache, else: execute the query and finally cache the result for subsequent reads.
I'm using [Redis List](https://redis.io/topics/data-types-intro#redis-lists) to cache the top articles list, The list items are ordered in the order they are pushed in.
We will use [rpush]() to insert new item and [lrange]() to retrieve items.

```javascript
const lrange = promisify(redisClient.lrange).bind(redisClient)
const rpush = promisify(redisClient.rpush).bind(redisClient)

// get most viewed articles
const getMostViewedArticles = async (length) => {
  const REDIS_KEY_MOST_VIEWED = 'blogs:mostviewed'
  const REDIS_MOST_VIEWED_TTL = 60 // refresh every minute
  const LENGTH = length || 10

  // first get previous cached top articles
  // the list is not changed siginficientely in short times
  const cachedArticles = await lrange(REDIS_KEY_MOST_VIEWED, 0, LENGTH - 1)
  let topArticles = []
  if (cachedArticles.length) {
    logger.info('getMostViewed: cache hit')
    cachedArticles.forEach((a) => {
      const doc = JSON.parse(a)
      topArticles.push(new Article(doc))
    })
    return Promise.resolve(topArticles)
  }

  // cache is empty, or expired
  // get new top articles
  const articlesIds = await mostViewdArticlesIds(LENGTH)
  // CAUTION: they are not sorted
  const articlesRandomOrder = await Article.find({ _id: { $in: articlesIds } })
  // order
  topArticles = articlesIds.map((id) =>
    articlesRandomOrder.find((article) => article.id === id)
  )

  // update cache
  logger.info('getMostViewed: updating cache')
  topArticles.forEach((article) => {
    logger.info('Addin article to mostviewd cache ', article.title)
    rpush(REDIS_KEY_MOST_VIEWED, JSON.stringify(article))
  })
  // set expiration key
  redisClient.expire(REDIS_KEY_MOST_VIEWED, REDIS_MOST_VIEWED_TTL)

  return Promise.resolve(topArticles)
}
```

Phew !, that's it now, we can easily just call this function from conrollers or whatever and get a list of N articles instances representing top articles.

This is the final code now, with the code from [part1]():

<script></script>

Make sure you get your head around this before proceeding to next part.

## Code smell

The feature is done .. Heck! I'm done too, it's 2:22 am and I have a long journey to Khartoun tomorrow _I know I know it's technically today_, but this last function smells soo much I can't sleep. It was better before we throw the caching-related code in it. So I should just refactor it into two functions? Probably, but this not the only function that's have multiple responsabilites, `recentArticles()` also do the same stinky thing. I have a better idea for this re-curring pattern. And also I really regret not using `promisify` from the begging this callback pattern is tedious.
OK. I give up, I'm going to sleep and on [part 3]() I'll _resolve_ this, I _promise_.
