# ActiveRecord

Сперва требуется ActiveRecord gem в вашем приложении, после передаём параметры
в ваше соединение с базой данных:

```ruby
require 'rubygems'
require 'sinatra'
require 'active_record'

ActiveRecord::Base.establish_connection(
  :adapter => 'sqlite3',
  :database =>  'sinatra_application.sqlite3.db'
)
```

Теперь вы можете создать и использовать ActiveRecord модели просто как в Rails
(в примере предполагается, что вы уже имеете 'posts' таблицу в базе данных):

```ruby
class Post < ActiveRecord::Base
end

get '/' do
  @posts = Post.all()
  erb :index
end
```

Это будет отрисовано `./views/index.erb`:

```erb
<% @posts.each do |post| %>
  <h1><%= post.title %></h1>
<% end %>
```
