# 📚 Book Review

A Laravel 12 application for browsing books and submitting reviews, built as a learning project from a Udemy course.

> **Note:** This is a personal learning project and has no license for redistribution.

## What It Does

-   Browse a list of 100 seeded books with star ratings (★☆) and review counts
-   Search books by title
-   Filter books by 5 preset options: Latest, Popular Last Month, Popular Last 6 Months, Highest Rated Last Month, Highest Rated Last 6 Months
-   View individual book pages with all reviews listed
-   Submit reviews with a rating (1-5) and text (minimum 15 characters)
-   Rate limited to 3 review submissions per hour per IP

## Screenshots / Demo

Run `php artisan serve` and visit `http://localhost:8000`

## Tech Stack

-   **Laravel 12** with PHP 8.2+ (8.2, 8.3, 8.4 all work)
-   **Tailwind CSS** via CDN (no build required for styling)
-   **SQLite** by default
-   **Blade** templates with one custom component (`<x-star-rating>`)

## Key Implementation Details

### Query Scopes in `Book.php`

The `Book` model has custom query scopes for filtering:

```php
scopePopularLastMonth()      // Orders by review count, then rating, min 2 reviews
scopePopularLast6Months()    // Same logic, 6 month window, min 5 reviews
scopeHighestRatedLastMonth() // Orders by avg rating, then review count, min 2 reviews
scopeHighestRatedLast6Months()
```

These use `withCount('reviews')` and `withAvg('reviews', 'rating')` for aggregate data.

### Caching Strategy

-   Book listings cached for 1 hour with key pattern `book:{filter}:{title}`
-   Individual books cached with key `book:{id}`
-   Cache busted automatically via model `booted()` events on Book and Review models

```php
// In Book.php and Review.php
protected static function booted()
{
    static::updated(fn(Book $book) => cache()->forget('book:' . $book->id));
    static::deleted(fn(Book $book) => cache()->forget('book:' . $book->id));
}
```

### Rate Limiting

Defined in `AppServiceProvider.php`:

```php
RateLimiter::for('reviews', function (Request $request) {
    return Limit::perHour(3)->by($request->user()?->id ?: $request->ip());
});
```

Applied via middleware in `routes/web.php`:

```php
Route::resource('books.reviews', ReviewController::class)
    ->scoped(['review' => 'book'])
    ->only(['create', 'store'])
    ->middleware(['throttle:reviews']);
```

### Database Seeder

Seeds 100 books in 3 tiers (33 each with good/average/bad reviews):

```php
Book::factory(33)->create()->each(function ($book) {
    Review::factory()->count(random_int(5, 30))->good()->for($book)->create();
});
// Repeated for average() and bad() factory states
```

### Review Factory States

```php
good()    // rating 4-5
average() // rating 2-5
bad()     // rating 1-3
```

## Project Structure

```
app/
├── Http/Controllers/
│   ├── BookController.php      # index() and show() only
│   └── ReviewController.php    # create() and store() only
├── Models/
│   ├── Book.php                # 14 query scopes, cache busting
│   └── Review.php              # $fillable = ['review', 'rating']
├── Providers/
│   └── AppServiceProvider.php  # Rate limiter definition
resources/views/
├── books/
│   ├── index.blade.php         # Book list with search + filters
│   ├── show.blade.php          # Single book with reviews
│   └── reviews/
│       └── create.blade.php    # Review submission form
├── components/
│   └── star-rating.blade.php   # ★☆ display component
└── layouts/
    └── app.blade.php           # Tailwind CSS classes defined here
```

## Installation

```bash
git clone <repo-url>
cd book-review
composer install
cp .env.example .env
php artisan key:generate
php artisan migrate
php artisan db:seed    # Seeds 100 books with 5-30 reviews each
php artisan serve
```

Or use the shortcut:

```bash
composer setup
```

## Routes

| Method | URI                            | Action                  | Description                   |
| ------ | ------------------------------ | ----------------------- | ----------------------------- |
| GET    | `/`                            | redirect                | Redirects to `/books`         |
| GET    | `/books`                       | BookController@index    | List books with search/filter |
| GET    | `/books/{book}`                | BookController@show     | Single book with reviews      |
| GET    | `/books/{book}/reviews/create` | ReviewController@create | Review form                   |
| POST   | `/books/{book}/reviews`        | ReviewController@store  | Submit review (throttled)     |

---

## 🔒 Security Considerations

### What's Good ✅

1. **CSRF Protection** - `@csrf` is present in the review form
2. **Input Validation** - Reviews validated: `'review' => 'string|min:15|required'`, `'rating' => 'integer|min:1|max:5|required'`
3. **Rate Limiting** - 3 reviews/hour per IP prevents spam
4. **Cascade Deletes** - Reviews deleted when parent book deleted (foreign key constraint)
5. **Mass Assignment Protection** - `Review` uses `$fillable`

### Potential Issues ⚠️

1. **No Authentication** - Anyone can submit reviews anonymously
2. **No `$fillable` on Book Model** - Currently not an issue since there's no book creation route, but add it if you expose one
3. **Cache Key Collision** - `book:{filter}:{title}` could conflict if titles contain special characters. Consider hashing:
    ```php
    $cacheKey = 'books:' . md5($filter . ':' . $title);
    ```
4. **No Validation Error Display** - The review form doesn't show `@error` directives. Add:
    ```blade
    @error('review')
        <p class="text-red-500 text-sm">{{ $message }}</p>
    @enderror
    ```
5. **Integer ID Exposure** - Book IDs are sequential integers. Not a security issue but consider UUIDs for production

### Not Issues (Clarifications)

-   The `LIKE '%title%'` query in `scopeTitle()` is safe because Laravel's query builder uses prepared statements

---

## 💡 Suggestions

### Quick Wins

1. **Add validation error display** in `create.blade.php`
2. **Add pagination** - Currently loads all books at once
3. **Add a "back to book" link** on the review form
4. **Add flash messages** for successful review submission

### If You Continue Building

1. **Add authentication** (Laravel Breeze is easiest)
2. **Associate reviews with users** - Add `user_id` to reviews table
3. **Add ability to edit/delete own reviews**
4. **Add book CRUD** for admins
5. **Add database indexes** on `reviews.book_id` and `reviews.created_at`

---

## ❓ Questions

1. Did the course cover testing? There's a `tests/` folder but I didn't see custom tests
2. Are you planning to add authentication or keep it anonymous?
3. The seeder creates books dated up to 2 years ago - is the time-based filtering working correctly for you?

---

## Laravel Concepts Demonstrated

| Concept                | Where                                                                     |
| ---------------------- | ------------------------------------------------------------------------- |
| Eloquent Relationships | `Book::hasMany(Review)`, `Review::belongsTo(Book)`                        |
| Query Scopes           | 14 scopes in `Book.php` including `scopePopular()`, `scopeHighestRated()` |
| Aggregate Queries      | `withCount()`, `withAvg()`                                                |
| Caching                | `cache()->remember()` with 1 hour TTL                                     |
| Model Events           | `booted()` for cache invalidation                                         |
| Rate Limiting          | Custom `reviews` limiter in AppServiceProvider                            |
| Blade Components       | `<x-star-rating>` anonymous component                                     |
| Resource Controllers   | RESTful routing with `->only()`                                           |
| Nested Resources       | `books.reviews` route with scoped binding                                 |

---

_Built while learning Laravel from a Udemy course._
