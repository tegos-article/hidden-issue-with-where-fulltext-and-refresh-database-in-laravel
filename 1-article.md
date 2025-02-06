## Understanding `whereFulltext` in Laravel

The `whereFulltext` method in Laravel allows developers to perform full-text searches on columns indexed with FULLTEXT in MySQL and PostgreSQL([docs](https://laravel.com/docs/11.x/queries#full-text-where-clauses)). This method is useful for searching large datasets efficiently, especially for text-based searches. However, while it enhances search performance, it has some caveats, particularly when used in tests with `RefreshDatabase`.

## The Role of `RefreshDatabase` in Testing

`RefreshDatabase` is a commonly used Laravel testing trait that ensures the database is migrated and reset before each test. It works by wrapping each test in a database transaction, which is rolled back after the test completes. This ensures database isolation between tests, preventing one test from affecting another([more](https://laravel.com/docs/11.x/database-testing#resetting-the-database-after-each-test)).

However, this transactional approach poses a significant issue when working with full-text indexes.

## The Issue: `FULLTEXT` Indexes and Transactions

Consider the following search implementation using `whereFulltext`:

```php
public function search(string $query): array
{
    return Product::query()
        ->select(['id'])
        ->whereFulltext(['article'], $query)
        ->toBase()
        ->pluck('id')
        ->toArray();
}
```

A basic test for this search method might look like this:

```php
public function test_search_internal_brand_details(): void
{
    $this->actingAsFrontendUser();

    $product = $this->createProduct();
    $article = $product->article;

    $response = $this->postJson(route('api-v2:search.details'), [
        'article' => $article
    ]);

    $response->assertOk();
    $response->assertJsonPath('data.0.article', $product->article);
    $response->assertJsonPath('data.0.brand_id', $product->brand_id);
}
```

This test, however, fails because the search query returns an empty result.

The root cause lies in how MySQL handles FULLTEXT indexes. MySQL does not support FULLTEXT indexes inside transactions ([see MySQL documentation](https://dev.mysql.com/doc/refman/en/innodb-fulltext-index.html#innodb-fulltext-index-transaction)). Since `RefreshDatabase` wraps each test in a transaction, the full-text index is not accessible during the test.

## Workarounds

### Committing the Transaction

One possible solution is to commit the transaction before executing the search:

```php
public function test_search_internal_brand_details(): void
{
    $this->actingAsFrontendUser();
    
    $product = $this->createProduct();
    DB::commit();

    $article = $product->article;
    
    $response = $this->postJson(route('api-v2:search.details'), [
        'article' => $article
    ]);

    $response->assertOk();
    $response->assertJsonPath('data.0.article', $product->article);
    $response->assertJsonPath('data.0.brand_id', $product->brand_id);
    
    Product::query()->truncate();
}
```

While this works, it is not an ideal approach since manually committing and truncating data increases complexity and breaks test isolation.

### Better Approach: Using a Search Repository

A cleaner and more maintainable solution is to abstract the search logic into a repository and mock it in tests.

Define a search repository interface:

```php
interface ProductSearchRepositoryInterface
{
    public function search(string $query): array;
}
```

Implement the repository using `whereFulltext`:

```php
final class ProductSearchDatabaseRepository implements ProductSearchRepositoryInterface
{
    public function search(string $query): array
    {
        return Product::query()
            ->select(['id'])
            ->whereFulltext(['article'], $query)
            ->toBase()
            ->pluck('id')
            ->toArray();
    }
}
```

Then, modify the test to mock the repository:

```php
public function test_search_internal_brand_details(): void
{
    $this->actingAsFrontendUser();
    
    $product = $this->createProduct();

    $productSearchRepository = $this->createMock(ProductSearchRepositoryInterface::class);
    $productSearchRepository->method('search')->willReturn([$product->id]);

    $this->app->instance(ProductSearchRepositoryInterface::class, $productSearchRepository);
    
    $article = $product->article;
    
    $response = $this->postJson(route('api-v2:search.details'), [
        'article' => $article
    ]);

    $response->assertOk();
    $response->assertJsonPath('data.0.article', $product->article);
    $response->assertJsonPath('data.0.brand_id', $product->brand_id);
}
```

By mocking the search repository, the test avoids dealing with FULLTEXT index limitations, making it more reliable and maintainable.

## Conclusion

Using `whereFulltext` in Laravel tests with `RefreshDatabase` can lead to unexpected issues due to MySQL's handling of FULLTEXT indexes inside transactions. While committing transactions manually can resolve the problem, a better approach is to use dependency injection and mock the search logic. This ensures test reliability and maintains proper isolation between test cases.
