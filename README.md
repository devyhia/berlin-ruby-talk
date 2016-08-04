## Agenda:
#### Show Rails App
- `rails new RailsApp --database=postgresql`
- `rake db:create`
- Handle `root` in `ApplicationController`.
- Log access
- Log errors
- logger.info
- ActiveRecord
 - `rails g model User name:string`
 - `rake db:migrate`
 - Count or insert ​_(and check the log)_​

#### Show Sinatra App
- Create App
  ```
  require 'sinatra/base'
  class Talk < Sinatra::Application
      set :environment, "development"

      get "/" do
        "Hello World!"
      end

      run!
  end
  ```
- Test Access log
- Test logger.info
- Test ActiveRecord
 - Install Sinatra ActiveRecord `gem install sinatra-activerecord`.
 - Hook database to the Sinatra App:
  ```
  set :database, {
      database: "RailsExample_development",
      adapter: "postgresql",
      host: "localhost",
      encoding: "unicode",
      pool: 5,
      username: "postgres"
  }
  ```

#### Try to log
- It's all working in $stdout!! But where is the log file?

#### Doesn't work? Search on Google.
- Follow official Sinatra documentation. http://recipes.sinatrarb.com/p/middleware/rack_commonlogger
- Follow official Access Log + Error.
https://spin.atomicobject.com/2013/11/12/production-logging-sinatra/
- After deep thought ... maybe redirect $stdout & $stderr ...? Brilliant idea!
  - Except that it breaks Passenger in production!
- Okay! I log on my own. Write in my own file. `logger.info` is working!! wohoooo!!
  - You would need to override the default RACK logger everywhere. Models? Controllers? Sidekiq Workers? You name it ... Not very maintainable.
- Okay, I override `env["rack.logger"]` ... But should I do that in a configure block? Or on every request? Maybe on every single request.
  - Oh it's all working now.
- But wait! Where is my ActiveRecord stuff?
  - Plug it in :) `ActiveRecord::Base.logger = Logger.new("log/#{settings.environment}.log")`
- Did I forget something?
  - Cassandra? Yes, you need to plug that.
  - Oh Sidekiq? RabbitMQ? You need to plug these, as well.
```
::Logger.class_eval { alias :write :'<<' }
access_logger = ::Logger.new("#{settings.root}/log/#{settings.environment}.log")
error_logger = ::File.new("#{settings.root}/log/#{settings.environment}.log","a+")
error_logger.sync = true
```
```
configure do
  use ::Rack::CommonLogger, access_logger
  ActiveRecord::Base.logger = access_logger
end
```
```
before {
  env["rack.errors"] = error_logger
  env["rack.logger"] = access_logger
}
```

#### Organize the log
- Well, now you have a bulk of crappy meaningless text in your log file. Congratulations!
  - You might consider writing logs to different files? Could be a good idea. Now you have all of these log files.
    - `log/development.access.log`
    - `log/development.error.log`
    - `log/development.sql.log`
    - `log/development.debug.log`
    - `log/development.cassandra.log`
    - `log/development.etc.log`

#### SemanticLogger?
- But wait ... Why you need all of this? You can use the SemanticLogger ... And, have:
  - `SemanticLogger["SQL"]`
    - That works!
  - `SemanticLogger["Access"]`
    - That works!
  - `SemanticLogger["Error"]`
    - Ops!! This takes a file (or, technically, any object that exposes `<<` or `write` functions).
      - Create your own class, and pass it `env["rack.error"]` ... hehe!!

#### What the fuck!!
- Wtf! Wait a sec ... Why is this so fucking complicated? That's why, at Minodes, we created the `sinatra-logger` gem.
  - Just one line and you're up and running: `logger filename: "log/#{settings.environment}.log", level: :info`
- Logging in your next awesome Sinatra API should be a painless experience! :)
- Thank you!
