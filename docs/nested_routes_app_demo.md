# Nested Routes App Demo

## Part 1: Create the app in a container (no local Ruby needed)
* `mkdir library_app && cd library_app`
* Generate the app, mounting the current directory:
* `docker run --rm -v "$PWD":/app -w /app ruby:3.3 sh -c "gem install rails && rails new . --skip-git"`
* On Linux, add `--user "$(id -u):$(id -g)"` so the files aren’t owned by root.
* Point out that the files now exist on your host even though Rails ran in the container.

## Part 2: Set up the dev container
* Create Dockerfile.dev:
```Dockerfile
FROM ruby:3.3
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install
```
* Create compose.yaml:

```yaml
services:
  web:
    build:
      context: .
    dockerfile: Dockerfile.dev
    command: bin/rails server -b 0.0.0.0
    ports: ["3000:3000"]
    volumes: [".:/app"]
```

* `docker compose up --build`

* Open http://localhost:3000 to show the Rails welcome page.

* Mention that `-b 0.0.0.0` is needed for the browser to reach the server inside the container.

* If you hit a stale-server error, run `rm tmp/pids/server.pid`.

* In a second terminal, run Rails commands with `docker compose exec web bin/rails ....` 

* Consider aliasing it to `rails` for the demo.

## Part 3: First resource (Author)

* `docker compose exec web bin/rails generate scaffold Author name:string`

* `docker compose exec web bin/rails db:migrate`

* Show the generated model, controller, views, and migration.

* Visit /authors, create a couple of authors, and show `docker compose exec web bin/rails routes -g author` to explain what `resources :authors` gives you.

## Part 4: Second resource (Book) and the association
* `docker compose exec web bin/rails generate scaffold Book title:string year:integer author:references`

* `docker compose exec web bin/rails db:migrate`

* In `app/models/author.rb`, add `has_many :books, dependent: :destroy`.

* Show that `book.rb` already has `belongs_to :author`.

* Visit `/books` to show it works as a standalone route (the author is picked by ID in the form for now).

* Optionally, open `docker compose exec web bin/rails console` and run `Author.first.books` to show the association.

## Part 5: Nested routes

* In `config/routes.rb`, change to:
```ruby
  resources :authors do
  resources :books
  end
```

* Run `docker compose exec web bin/rails routes -g book` and point out the new `/authors/:author_id/books` paths.

* Update BooksController:
  * Add `before_action :set_author, with @author = Author.find(params[:author_id])`.
  * Scope lookups through the parent: `@author.books` and `@author.books.build`. 
  * Redirect to `[@author, @book]` or `author_books_path(@author)`. 

* Update the book views:
  * Forms: `form_with model: [@author, @book]`.
  * Links: `author_book_path(@author, book)`, `new_author_book_path(@author),` and so on. 
  * Remove the `author-ID` field from the form, since the parent is now implied by the URL. 
  * Update `authors/show.html.erb` to list the author’s books and add a "Add book" link to `new_author_book_path(@author)`.
  * Walk through the flow live: `author`, then their `books`, then add a book.
  * Optional: mention `shallow: true` as a common alternative.

## Part 6: Wrap-up
* Set the application root to `authors#index` in `config/routes.rb`. 

* Recap: one container, a mounted directory, two resources, one association, and nested routes expressing that 
relationship in the URL. 

* Optional extras if time allows: a validation (`validates :title, presence: true`) or showing the SQL in the server
logs.
