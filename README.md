# flaskr 2.0

The [Flask Tutorial](https://github.com/mitsuhiko/flask/tree/master/docs/tutorial)
is great for learning Flask, but it requires the use of SQLite. Some people
learning Flask may not have experience with relational databases, so this
additional requirement becomes an unnecessary distraction.

It seems like Redis would be a much better fit over SQLite for this tutorial. It 
would be necessary to install Redis in the development environment, but it is
getting more popular (Cloud9 and Heroku both have it). I think it's worth the
effort to install Redis than learn SQL.

This was designed for the [Cloud9](http://c9.io) environment.


## Requirements

- pip install redis-collections
- pip install Flask


## Start Services

- sudo service redis-server start


## Additional Reading

- https://redis-collections.readthedocs.org/en/latest/

