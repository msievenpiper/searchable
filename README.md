Searchable, the search method for Laravel
==========================================

Searchable is a trait for Laravel that adds a simple search function to Eloquent Models.

Searchable allows you to perform searches on the models table, and is configurable for columns 
and even relationships.

# Installation

Searchable requires laravel 4.2 or higher to function (yes it will work with 7.x), 
as well as a PHP version higher than 5.4.0.

Run the composer command: 
```bash
composer require nicolaslopezj/searchable
```

Or simply add the package to your `composer.json` file and run `composer update`.

```
"nicolaslopezj/searchable": "1.*"
```

Be sure to check the releases page to make sure you get the correct version for your project.

# Usage

Add the trait to your model and set your search rules like the example below.

```php
use Nicolaslopezj\Searchable\SearchableTrait;

class User extends Model
{
    use SearchableTrait;

    /**
     * Searchable rules.
     *
     * @var array
     */
    protected $searchable = [

        /**
         * Columns and their priority in search results.
         * Columns with higher values are more important.
         * Columns with equal values have equal importance.
         *
         * @var array
         */
        'columns' => [
            'first_name' => 10,
            'last_name' => 10,
            'bio' => 2,
            'email' => 5,
        ]

    ];
    
    ...
}
```

```php
// Simple search
$users = User::search('John')->get();
```

## Searching with relations

```php
use Nicolaslopezj\Searchable\SearchableTrait;

class User extends Model
{
    use SearchableTrait;

    /**
     * Searchable rules.
     *
     * @var array
     */
    protected $searchable = [
        /**
         * Columns and their priority in search results.
         * Columns with higher values are more important.
         * Columns with equal values have equal importance.
         *
         * @var array
         */
        'columns' => [
            'users.first_name' => 10,
            'users.last_name' => 10,
            'users.bio' => 2,
            'users.email' => 5,
            'posts.title' => 2,
            'posts.body' => 1,
        ],
        'joins' => [
            'posts' => ['users.id','posts.user_id'],
        ],
    ];

    public function posts()
    {
        return $this->hasMany('Post');
    }

}
```

Now you can search your model.

```php
// Simple search
$users = User::search('John')
            ->get();

// Make sure if you are getting your relations back for search results you call 
// with() after search()
$users = User::search('John')
            ->with('posts')
            ->get();
```

## Paginating Search Results

You can still use laravel's paginate method on queries.

```php
// Search with relations and paginate
$users = User::search('Hello World!')
            ->with('posts')
            ->paginate(20);
```

## Applying Additional Queries

You can still use all your normal query methods on your models without issue. 

```php
// Search only active users
$users = User::where('status', 'active')
            ->search('John')
            ->paginate(20);
```

## Custom Threshold

The default threshold for acceptance is calculated via the sum
of all relevant ratings for each result found divided by the number of search columns. 
However, if you like to change this to a lower or higher rating you can pass this through the 
`search` method as an additional param.

```php
$threshold = 0; // Set custom threshold for search results
// Search with lower relevance threshold
$users = User::where('status', 'active')
            ->search('John', $threshold)
            ->paginate(20);
```

The above, will return all users in order of relevance.

## Entire Text search

By default searchable will break any string into it's component "words" and search for each
within the columns specified. However if you need to look for an exact match to a sentence 
or phrase you can pass an additional params to the `search` method.

For example:
```php
// Prioritize matches containing "John Doe" above matches that contain only "John" or "Doe".
$users = User::search("John Doe", $threshold, true)->get();
```

By passing true through as the third params you get an exclusive word search, 
this sorts results that contain the phrase/sentence in the provided string first.

If you explicitly want to search for full text matches only, you can disable multi-word 
splitting entirely by setting the fourth parameter to true.

```php
// Do not include matches that only matched "John" OR "Doe".
$users = User::search("John Doe", $threshold, true, true)->get();
```

# How does it work?

Searchable builds a query that search through your model using Laravel's Eloquent.
Here is an example query

#### Eloquent Model:
```php
use Nicolaslopezj\Searchable\SearchableTrait;

class User extends Model
{
    use SearchableTrait;

    /**
     * Searchable rules.
     *
     * @var array
     */
    protected $searchable = [
        'columns' => [
            'first_name' => 10,
            'last_name' => 10,
            'bio' => 2,
            'email' => 5,
        ],
    ];

}
```

#### Search:
```php
$search = User::search('John Doe', null, true)->get();
```

#### Result:
```sql
select `users`.*, 

-- If third parameter is set as true, it will check if the column starts with the specified string.
-- If the string is an exact match, we take the relevance weight (10) and multiply this by 30.
(case when first_name LIKE 'John Doe%' then 300 else 0 end) + 

-- For each column searchable generates 3 additional queries
-- These queries check the "words" contained in the search string are with the column and assigns a value to each sceanrio

-- The first checks if the column is equal to any of the words within the search string
-- if then it adds relevance weight (10) * 15
(case when first_name LIKE 'John' || first_name LIKE 'Doe' then 150 else 0 end) + 

-- The second checks if the column starts with the word, if it does we add the relevance weight * 5
(case when first_name LIKE 'John%' || first_name LIKE 'Doe%' then 50 else 0 end) + 

-- The third checks if the column contains the word in an position, if so it adds relevance weight * 1
(case when first_name LIKE '%John%' || first_name LIKE '%Doe%' then 10 else 0 end) + 

-- This then repeats for each seach column and calculates based on the relevance weight assigned to each column

as relevance  -- This is the full sum of relevance based every column
from `users` 
group by `id` 

-- Selects only the rows that have more than
-- the threshold set/calculated
having relevance > 6.75 

-- Orders the results by relevance
order by `relevance` desc
```

## Important Considerations
This is not optimized for big searches, but sometimes you just need to make it simple 
(Although it is not slow).

Remember to use indexed columns where appropriate and caching using laravel's 
cache facade to improve performance of larger queries.

## Contributing

Anyone is welcome to contribute. Fork, make your changes, and then submit a pull request.

[![Support via Gittip](https://rawgithub.com/twolfson/gittip-badge/0.2.0/dist/gittip.png)](https://gratipay.com/nicolaslopezj/)
