# Встраивание Sinatra в EventMachine

EventMachine это очень полезный инструмент и иногда вам нужно добавить
веб-интерфейс поверх него. Да, EM действительно поддерживает это из коробки, но
это может быть уродливым и трудным для работы. Почему бы не использовать то,
что все уже знают и любят, как, например, Sinatra?

Below is a (working) code-sample for running a simple HelloWorld Sinatra app
within EventMachine. I've also provided a simple example of deferring tasks
within your Sinatra call.

Ниже приведен (рабочий) пример кода для запуска простого приложения HelloWorld
Sinatra в EventMachine. Я также представил простой пример отсрочки задач в
рамках вашего вызова Sinatra.

```ruby
require 'eventmachine'
require 'sinatra/base'
require 'thin'


# Этот пример показывает, как внедрить Sinatra в ваше EventMachine приложение.
# Это очень полезно, если для приложения требуется какой-то интерфейс API, и вы
# не хотите использовать предоставленный веб-сервером EM.

def run(opts)

  # Запуск реактора
  EM.run do

    # определяем некоторые значения по умолчанию для приложения
    server  = opts[:server] || 'thin'
    host    = opts[:host]   || '0.0.0.0'
    port    = opts[:port]   || '8181'
    web_app = opts[:app]

    # создать базовое отображение, на которое будет установлено наше
    # приложение. Если у меня есть следующие маршруты:
    #
    #   get '/hello' do
    #     'hello!'
    #   end
    #
    #   get '/goodbye' do
    #     'see ya later!'
    #   end
    #
    # Тогда Вы получите следующее:
    #
    #   mapping: '/'
    #   routes:
    #     /hello
    #     /goodbye
    #
    #   mapping: '/api'
    #   routes:
    #     /api/hello
    #     /api/goodbye
    dispatch = Rack::Builder.app do
      map '/' do
        run web_app
      end
    end

    # ПРИМЕЧАНИЕ о том, что мы должны использовать EM-совместимый веб-сервер. Тут может быть больше, но это те, которые в настоящее время доступны.
    unless ['thin', 'hatetepe', 'goliath'].include? server
      raise "Need an EM webserver, but #{server} isn't"
    end

    # Запустите веб-сервер. Обратите внимание, что вы можете запускать другие
    # задачи в своем экземпляре EM.
    Rack::Server.start({
      app:    dispatch,
      server: server,
      Host:   host,
      Port:   port,
      signals: false,
    })
  end
end

# Наше простое hello-world приложение
class HelloApp < Sinatra::Base
  # threaded - False: Примет запросы на процесс реактора
  #            True:  Запросит очередь на фоновый поток
  configure do
    set :threaded, false
  end

  # Запрос работает на реакторной нити (c thread установленном как false)
  get '/hello' do
    'Hello World'
  end

  # Запрос работает на процессе реактора (с thread установленном в false) и немедленно возвращается. Отсроченная задача не задерживает ответ от веб-сервиса.
  get '/delayed-hello' do
    EM.defer do
      sleep 5
    end
    'I\'m doing work in the background, but I am still free to take requests'
  end
end

# стартуем приложение
run app: HelloApp.new
```

Вы можете запустить это просто командой:

```bash
ruby em-sinatra-test.rb # em-sinatra-test.rb это имя файла с кодом который выше
```

Вы должны также быть в состоянии проверить, что это работает правильно со
следующей командой `ab`:

```bash
ab -c 10 -n 100 http://localhost:8181/delayed-hello
```

Если это завершается меньше чем за секунду, то вы имеете успешно настроенный
Sinatra с запуском с EM и вы берете запросы на event-loop и отсрочиваете задачи
в фона. Если проверка отнимает больше, чем около 1 секунды, то вы, скорее
всего, берете запросы на заднем плане, что происходит, когда EM, заполняет
очередь, вы не можете обработать свои запросы sinatra (и это плохо!).
Удостоверьтесь, что вы имеете `thread` установленным как `false` и затем
попробуйте еще раз.

