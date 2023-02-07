+++
title = "Use Bayesian Averages to Improve Rating Sorting in your Elasticsearch Index"
date = "2022-06-08"
tags = ["php", "elasticsearch"]
+++

Disclaimer: This post is a copy of my [JoliCode blog post](https://jolicode.com/blog/use-bayesian-averages-to-improve-rating-sorting-in-your-elasticsearch-index).

*EDIT*: There were two typos in this article. In the formula we were wrong about the position of one of the arguments which made it wrong, we invite you to check again the formula so you can fix it too. And in the Elasticsearch painless script, a subtlety of the Java language will make a division of a double variable by a long variable outputting a long rounded (badly indeed) so you have to be sure all your variables are doubles. If you never read the article, it is now correct and you don't have to worry about it!

Our modern life is dominated by ratings, whether we go shopping, to the restaurant and even when looking for a spa. But how do we make sure these ratings are accurate?
How could you, with an Elasticsearch cluster, sort by these ratings?

In this post we will work with a list of `Book`. When using the average rating sorting, we will get strange results: a `Book` with only one vote and a 10 average rating on top of the listing followed by another one with 100 votes and an average rating of 9,2 - this doesn’t make any sense.

This is why you would want to use some complex formula to take everything into account. In this post we’ll present to you what the Bayesian average is and how you can use it with a database and Elasticsearch to improve your sorting.

## Bayesian average, what?

> A **Bayesian average** is a method of estimating the mean of a population using outside information, especially a pre-existing belief, which is factored into the calculation. This is a central feature of Bayesian interpretation. This is useful when the available data set is small.

From [https://en.wikipedia.org/wiki/Bayesian_average](https://en.wikipedia.org/wiki/Bayesian_average).

I’m feeling the same as you are now: that quote didn’t help me understand and I was even more intrigued when reading this; what helped me was reading a practical usage of this method.

In our case, we have a `Book` resource that can be rated from 1 to 10. So all the users of our website can rate that `Book` and when the users are searching in the catalog, we want the default sorting to be based on this rating.

That's why we want to weigh this average by the number of voters of all the `Book`. Here is a more comprehensive formula:

![Bayesian average formula](/media/original/2022/average_formula.png)

- “b” annotation refer to book related values;
- “ab” annotation refer to “all book” related values;
- “v” indicate the count of votes, so “Bv” will be the count of votes for a specific book, and “ABv” will be the count of votes for all the books;
- “c” indicate the count of related object;
- “a” will correspond to the average of the related object.

We did [a sheet](https://docs.google.com/spreadsheets/d/1wz2gXRsEhU_u2l30NPFF3RHXgvBv9BR7L0BAH4veEUQ/edit?usp=sharing) to see how that formula will behave based on several factors. So you can understand how the formula will distribute your elements based on their ratings.

## Implementation

Now let’s apply this to our application! Here, we will use a Symfony application which allows us to bypass a lot of bootstrap and go directly to the interesting part. We have a database that contains users, books and ratings. Also, We have a `book` index in an Elasticsearch cluster that reflects our database books but normalized, with a extra field as following in our mapping:

```yaml
mappings:
  dynamic: false
  properties:
    # other properties ...
    bayesianAverage:
      type: float
```

`bayesianAverage` will be used to sort the `book` index later on. We will need to fill that value. During index creation, we transform the Book entity into an Elasticsearch document, and we compute the Bayesian average.

```php
class BookTransformer implements EntityTransformer
{
    private function createModel(Book $book, array $normalized): BookModel
    {
        $model = new BookModel();
        // some assignations to fill my model

        $distribution = $this->ratingsDistribution->get($book->getId());
        
        $ratings = new RatingModel();
        $ratings->set1($distribution[1]); // Count of "1" ratings for this book
        $ratings->set2($distribution[2]); // Count of "2" ratings for this book...
        $ratings->set3($distribution[3]);
        $ratings->set4($distribution[4]);
        $ratings->set5($distribution[5]);
        $ratings->set6($distribution[6]);
        $ratings->set7($distribution[7]);
        $ratings->set8($distribution[8]);
        $ratings->set9($distribution[9]);
        $ratings->set10($distribution[10]);
        $ratings->setCount(array_sum($distribution));
        $model->setRatings($ratings);

        $model->setBayesianAverage($this->bayesianAverage->compute($ratings));

        return $model;
    }
}
```

Here we have a `RatingsDistribution` service[^1] we use to compile all votes for a given `Book` within an array of integers with all possible ratings! That way we can assign our `Book` model with the number of voters for each rating and the total count of voters. These values will be used later on to calculate the Bayesian average from inside Elasticsearch.

And we are also seeing a `BayesianAverage` service we used to do calculations. Here is what it looks like:

```php
class BayesianAverage
{
    public function allVotesCount(): int
    {
        $result = $this->connection->fetchAssociative('SELECT COUNT(id) as count FROM rating;');

        return (int) $result['count'];
    }

    public function allAverageRating(): float
    {
        $result = $this->connection->fetchAssociative('SELECT AVG(r.rating) as average FROM rating r;');

        return (float) $result['average'];
    }

    public function allCount(): int
    {
        $result = $this->connection->fetchAssociative('SELECT COUNT(b.id) as count FROM book b;');

        return (int) $result['count'];
    }

    public function compute(RatingModel $bookRatings): float
    {
        $bookCount = 0 === $bookRatings->getCount() ? 1 : $bookRatings->getCount();
        $inter = $this->allCount() / ($bookCount + ($this->allVotesCount() / $bookCount));
        $avg = (($bookRatings->get1Count() * 1) + ($bookRatings->get2Count() * 2) + ($bookRatings->get3Count() * 3) + ($bookRatings->get4Count() * 4) + ($bookRatings->get5Count() * 5) + ($bookRatings->get6Count() * 6) + ($bookRatings->get7Count() * 7) + ($bookRatings->get8Count() * 8) + ($bookRatings->get9Count() * 9) + ($bookRatings->get10Count() * 10)) / $bookCount;

        return $inter * $avg + (1 - $inter) * $this->allAverageRating();
    }
}
```

For our implementation we will cache global variables with a 1 hour time to live:

- all `Book`votes count in a `bayesian_all_votes` key;
- all `Book` average rating in a `bayesian_all_average_rating` key;
- `Book` count in a `bayesian_all_count` key.

Then we have a `calculate` method which makes all required calculations to get our Bayesian average for a given `Book` ratings distribution. Now, we are able to fill our index on creation with a correct Bayesian average on all our `Book` models!

## Handling new votes

But it works only on indexation, what happens if a user gives a new vote? All values we have calculated will be wrong and have to be refreshed!

First thing we will solve is updating the related model in Elasticsearch when a user is submitting a new vote on a `Book`. When we have a new vote, we will update the Elasticsearch model with a stored painless script[^2] called `bayesian-update` in our Elasticsearch cluster. It will update the document values and update the `bayesianAverage` field, here is what the script will look like:

```java
// only for updates
if (params.addedRating > 0) {
    ctx._source.ratings[params.addedRating.toString()]++;
    ctx._source.ratings['count']++;
}
if (params.removedRating > 0) {
    ctx._source.ratings[params.removedRating.toString()]--;
    ctx._source.ratings['count']--;
}

// classic Bayesian average calculations
long s1 = ctx._source.ratings['1'];
long s2 = ctx._source.ratings['2'];
long s3 = ctx._source.ratings['3'];
long s4 = ctx._source.ratings['4'];
long s5 = ctx._source.ratings['5'];
long s6 = ctx._source.ratings['6'];
long s7 = ctx._source.ratings['7'];
long s8 = ctx._source.ratings['8'];
long s9 = ctx._source.ratings['9'];
long s10 = ctx._source.ratings['10'];
double count = ctx._source.ratings['count'];

double inter = params.allCount / (count + (params.allVotesCount / count));
double avg = ((s1 * 1) + (s2 * 2) + (s3 * 3) + (s4 * 4) + (s5 * 5) + (s6 * 6) + (s7 * 7) + (s8 * 8) + (s9 * 9) + (s10 * 10)) / count;

ctx._source['bayesianAverage'] = inter * avg + (1 - inter) * params.allAverageRating;
```

This script use the following parameters:
- `allCount` corresponds to the method with the same name in our `BayesianAverage` service, it will contains the number of `Book` we have in our database;
- `allVotesCount` also corresponds to the method with the same name in our `BayesianAverage` service and it will contain the number of votes for all the `Book`;
- `allAverageRating` is also mirrored to the `BayesianAverage` service and contains the global average on all `Book` votes;
- then `addedRating` corresponds to the rating that was added on that `Book`;
- lastly, `removedRating` is here because in my application, a user can update his rating to another value so this field will contain the value that was removed if a user is updating his vote.

Now we can run our script each time there is a new vote by using this request onto the Elasticsearch cluster:

```php
POST book/doc/67433/_update
{
  "script" : {
    "id": "bayesian-update",
    "params": {
      "addedRating": 7,
      "removedRating": 0,
      "allCount": 100,
      "allVotesCount": 30160,
      "allAverageRating": 6.51
    }
  }
}
```

That request will trigger an execution of the `bayesian-update` script on our model and update it automatically, without having to send the complete `Book` model again.

## Keep the Bayesian average up-to-date

Now that new votes are managed, a big problem remains: Bayesian average is based on an average for a specific book, **and** the average for **all books**, so how can we be sure that our value is still relevant in time when more `Book` and votes are added?

In our case, we have a lot of `Book` and thanks to that, the difference in the Bayesian average result will be too small to be significant when new `Book` or new votes are added. That is why we chose to make a daily update on all our `Book` for the `bayesianAverage` field. This time frame should be based on your website traffic and if you need a really precise sorting or if some small variations are allowed. The more documents you have, the less frequently you will need to update since a new document won’t impact the global average that much.

For this update I created a new script called `bayesian-refresh` which is very similar to the first script but without the rating update part (so both `addedRating` and `removedRating` are not required parameters anymore).

To trigger this script, we use the bulk update API in Elasticsearch to run the script on all documents by selecting them by their ids as following:

```php
POST book/_bulk
{"update": {"_id" : 67433}}
{"script" : {"id": "bayesian-refresh", "params" : {"allCount": 100, "allVotesCount": 30160, "allAverageRating": 6.51}}}
{"update": {"_id" : 67434}}
{"script" : {"id": "bayesian-refresh", "params" : {"allCount": 100, "allVotesCount": 30160, "allAverageRating": 6.51}}}
{"update": {"_id" : 67435}}
{"script" : {"id": "bayesian-refresh", "params" : {"allCount": 100, "allVotesCount": 30160, "allAverageRating": 6.51}}}
{"update": {"_id" : 67436}}
{"script" : {"id": "bayesian-refresh", "params" : {"allCount": 100, "allVotesCount": 30160, "allAverageRating": 6.51}}}
{"update": {"_id" : 67437}}
{"script" : {"id": "bayesian-refresh", "params" : {"allCount": 100, "allVotesCount": 30160, "allAverageRating": 6.51}}}
```

Sadly Elasticsearch doesn’t provide a way to trigger a script on all documents of an index so we’re forced to do it that way[^3].

## Conclusion

Thanks to this formula and Elasticsearch we are now able to sort our `Book` listing per ratings based on the whole population of `Book` which makes this sort even more relevant! If you need this, you should consider “freshness” of the votes, in our case it doesn’t matter but in some projects, you will want to put less weight on an older vote. But thanks to  this post you should be able to setup a basic sort based on your newly Bayesian average field 🎉.

```php
GET book/_search
{
  "query": {
    "match_all": {}
  },
 "sort": [
    { "bayesianAverage": "desc" }
  ]
}
```

This post was greatly inspired by: [https://www.evanmiller.org/bayesian-average-ratings.html](https://www.evanmiller.org/bayesian-average-ratings.html) If you want more mathematical details you should take a look ! 😉 Also our formula won’t match that one because we simplified some concepts and removed others, like time, since we don’t need these in our case.

[^1]: An extract of that `RatingsDistribution` class if you are interested about it: [https://gist.github.com/Korbeil/9e00073c29a26f55160abddcc9888757](https://gist.github.com/Korbeil/9e00073c29a26f55160abddcc9888757). Keep in mind that our `Rating` entity is composed of 3 fields: a `Book` relation, an `User` relation and a `rating` (integer from 1 to 10).
[^2]: Painless is a scripting language provided by Elasticsearch to create custom behaviors within Elasticsearch [https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-scripting-painless.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-scripting-painless.html).
[^3]: We could use the Reindex API instead of the Bulk update, but it requires to create a new index, will all the complexity it adds when you have real-time updates happening live.