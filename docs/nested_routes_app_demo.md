# Nested Routes App Demo

## Part 1: Create the app in a container (no local Ruby needed)
* Create the project working directory:
```bash
mkdir library_app && cd library_app
```
* Generate the app, mounting the current directory:
```bash
docker run --rm -v "$PWD":/app -w /app ruby:4.0.7 sh -c "gem install rails && rails new . --skip-git"
```
* On Linux, add `--user "$(id -u):$(id -g)"` so the files aren’t owned by root.
* Point out that the files now exist on your host even though Rails ran in the container.

## Part 2: Set up the dev container
* Create Dockerfile.dev:
```Dockerfile
FROM ruby:4.0.7
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

* Bring up the Docker container:
```bash
docker compose up --build
```

* Open `http://localhost:3000` to show the Rails welcome page.

* Mention that `-b 0.0.0.0` is needed for the browser to reach the server inside the container.

* If you hit a stale-server error, run `rm tmp/pids/server.pid`.

* In a second terminal, run Rails commands with:
```bash
docker compose exec web bin/rails ...
```
>[!NOTE]
> Note that we are using the container name from the .yaml file, not from Docker Desktop!

## Part 3: Create the first resource (Author)

* Create the Author resource and all of its components/files:
```bash
docker compose exec web bin/rails generate scaffold Author name:string
`

* Run the migration created by the above command so that the database can support the resource:
```bash
docker compose exec web bin/rails db:migrate
```

* Show the generated model, controller, views, and migration files (e.g. `app/models/*.rb`, `app/controllers/*.rb`, `app/views/authors/*.erb`).

* Visit `/authors`, create a couple of authors, and show `docker compose exec web bin/rails routes -g author` to explain what `resources :authors` gives you.

## Part 4: Second the second resource (Book) and the association
* Create the second resource, Book:
```bash
docker compose exec web bin/rails generate scaffold Book title:string year:integer author:references
```

* Run the migration for this resource:
```bash
docker compose exec web bin/rails db:migrate
```

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

* Run `docker compose exec web bin/rails routes -g books` and point out the new `/authors/:author_id/books` paths.

* Update the Books controller (`app/controllers/book_controller.rb`):
  * Add `before_action :set_author` before the existing `set_book` callback, and define a private `set_author` method whose body is `@author = Author.find(params[:author_id])`.
  * Redirect to `[@author, @book]` or `author_books_path(@author)`. 
  * Scope lookups through the parent: `@author.books` and `@author.books.build`:

```ruby
class BooksController < ApplicationController
  before_action :set_author
  before_action :set_book, only: %i[show edit update destroy]

  def index
    @books = @author.books
  end

  def show
  end

  def new
    @book = @author.books.build
  end

  def edit
  end

  def create
    @book = @author.books.build(book_params)

    if @book.save
      redirect_to [@author, @book], notice: "Book was successfully created."
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    if @book.update(book_params)
      redirect_to [@author, @book], notice: "Book was successfully updated."
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @book.destroy
    redirect_to author_books_path(@author), notice: "Book was successfully deleted."
  end

  private

  def set_author
    @author = Author.find(params[:author_id])
  end

  def set_book
    @book = @author.books.find(params[:id])
  end

  def book_params
    params.require(:book).permit(:title, :year)   # :author_id removed
  end
end
```

* Update the book views:
  * In `books/_form.html.erb`, replace the `form_with` to be `<%= form_with model: [@author, @book] do |form| %>`
  * Links: `author_book_path(@author, book)`, `new_author_book_path(@author),` and so on in the *.html.erb files:
```ruby
<%= link_to "Add book", new_author_book_path(@author) %>
<%= link_to "Show", [@author, book] %>
<%= link_to "Edit", edit_author_book_path(@author, book) %>
<%= button_to "Delete", [@author, book], method: :delete %>
<%= link_to "Back to books", author_books_path(@author) %>
```
  * Remove the `author-ID` field from the book form, since the parent is now implied by the URL. 
  * Update `authors/show.html.erb` to list the author’s books and add a "Add book" link to `new_author_book_path(@author)`.
  * Walk through the flow live: `author`, then their `books`, then add a book.
  * Optional: mention `shallow: true` as a common alternative.

## Part 6: Wrap-up
* Set the application root to `authors#index` in `config/routes.rb`. 

* Recap: one container, a mounted directory, two resources, one association, and nested routes expressing that 
relationship in the URL. 

* Optional extras if time allows: a validation (`validates :title, presence: true`) or showing the SQL in the server
logs.

* The rule of thumb is that <%= %> is for tags that should display a value, such as `<%= book.title %>`, and `<% %>` is for control flow like `each`, `if`, and `end`.
