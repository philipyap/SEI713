# ![](https://ga-dash.s3.amazonaws.com/production/assets/logo-9f88ae6c9c3871690e33280fcf557f33.png) SQLAlchemy in Flask

### 📚 Learning Objectives
*After this lesson, you will be able to:*
- Create Models and migrate them to a Postgres database
- Perform CRUD Operations with SQLAlchemy
- Query it all via an API

## Project Initialization

Let's set up our app! We are going to need to
- Set up a virtual environment
- add our `.gitignore` _(curious what to put in it? Checkout [GitHub's suggested Python gitignore](https://github.com/github/gitignore/blob/master/Python.gitignore))_
- initialize and empty git repository
- Activate the virtual environment
- Upgrade our Pip in the venv
- Install Flask, Flask-Sqlalchemy, and Psycopg2
- Copy our installs into a `requirements.txt` file.
- Create a `models.py` file for our models and an `api.py` for our flask server.

_Not sure of the commands? Follow these:_

```zshell
python3 -m venv flaSQL
echo "lib\nbin\n__pycache__/\ninstance/\n.webassets-cache\n.vscode\n.DS_Store" >> .gitignore
git init
. flaSQL/bin/activate
pip install --upgrade pip
pip3 install flask
pip3 install Flask-SQLAlchemy
pip3 install psycopg2
pip3 freeze > requirements.txt
touch models.py api.py
```

## Model Setup

When using SQLAlchemy in a Flask server, they need to be able to communicate with each other. This setup is going to happen in our `models.py` file. We are doing it in this file because models are often set up initially and then left alone, so the Flask app setup makes sense to be here. 

>If you have app setup that's more sizable, you can put it in a separate file and import what you need from it.

After importing Flask from flask and SQLAlchemy from flask_sqlalchemy, the flask app needs to be created, the database URI set to it, and SQLAlchemy given the newly created and configured app.

```python=
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['SQLALCHEMY_ECHO'] = True
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://localhost/flasql'
db = SQLAlchemy(app)
```

We have three config settings in here. 
1. `SQLALCHEMY_TRACK_MODIFICATIONS` is a feature that is going to be removed on the next major release. This line isn't necessary, but if we don't have it, it'll throw us a warning which is annoying. Also, the python community is big on being explicit which means delaring this to be false even though is defaulted to that behavior is a shibboleth you can use to show how cool you are with the python community.
2. `SQLALCHEMY_ECHO` will print out all the SQL queries it is issuing. This defaults to False, but since I'm a nosey dev, I want to see everything my middleware does, so I set this to true. If you want to keep your console clean and your debugging harder, you can take that line out or embody the python lifestyle and **explicity** set it to False.
3. `SQLALCHEMY_DATABASE_URI` is pretty self explanitory. This URI tells our program everything it needs to know about our databse. This database URI will change when going into production, but for now, localhost is where our postgres database lives. 
> A Database URI is crafted like this:
`[DB_TYPE]+[DB_CONNECTOR]://[USERNAME]:[PASSWORD]@[HOST]:[PORT]/[DB_NAME]`
For macs, the database connector, username, password, and port are all assumed. Note that not all operating systems function that way.

### 📃 The Model Itself

This part looks and functions exactly like plain SQLAlchemy, the only difference is that, instead of importing all the Classes we need, they're all just called from the `db` variable we created.

```python
class User(db.Model):
  __tablename__ = 'users'

  id = db.Column(db.Integer, primary_key=True)
  email = db.Column(db.String, unique=True, nullable=False)
  name = db.Column(db.String, nullable=False)
  bio = db.Column(db.String(150))

  def __repr__(self):
    return f'User(id={self.id}, email="{self.email}", name="{self.name}")'
```

This syntax should be pretty familiar! 

#### Try it yourself!
We're going to have three models:
1. Users _(already defined)_
2. Posts
    - Header: String(150), unique, not null
    - author_id: Foreign Key (references user), on delete set null
    - body: String, not null
3. Tags
    - tag: String(50), unique, not null

Users has a 1:M relationship with Posts (One user has many posts, one post has one author)
Posts has a N:M relationship with Tags (many posts have many tags, many tags are applied to many posts)

Use the SQLAlchemy you've already learned along with the User model we defined above to try to create those relations!

#### The Models—Final Version

The code below will look different from yours. How is it different? Why do you think those changes were applied?


```python
class User(db.Model):
  __tablename__ = 'users'

  id = db.Column(db.Integer, primary_key=True)
  email = db.Column(db.String, unique=True, nullable=False)
  name = db.Column(db.String, nullable=False)
  bio = db.Column(db.String(150))

  posts = db.relationship('Post', backref='author', lazy=True)

  def __repr__(self):
    return f'User(id={self.id}, email="{self.email}", name="{self.name}")'

post_tags = db.Table('post_tags',
    db.Column('tag_id', db.Integer, db.ForeignKey('tags.id'), primary_key=True),
    db.Column('post_id', db.Integer, db.ForeignKey('posts.id'), primary_key=True)
)

class Post(db.Model):
  __tablename__ = 'posts'

  id = db.Column(db.Integer, primary_key=True)
  header = db.Column(db.String(150), unique=True, nullable=False)
  author_id = db.Column(db.Integer, db.ForeignKey('users.id', ondelete='SET NULL'))
  body = db.Column(db.String, nullable=False)

  tags = db.relationship('Tag', secondary=post_tags, lazy='subquery',
        backref=db.backref('posts', lazy=True))

  def __repr__(self):
    return f'User(id={self.id}, email="{self.email}", display_name="{self.display_name}")'

class Tag(db.Model):
  __tablename__ = 'tags'

  id = db.Column(db.Integer, primary_key=True)
  tag = db.Column(db.String(50), unique=True, nullable=False)

  def __repr__(self):
    return f'Tag(id={self.id}, tag="{self.tag}")'
```

Some things to note in these model definitions:
- **backref** vs **back_populates**: _When using `backref`, you only have to put the relationship on ONE class. When using `back_populates`, the relationship needs to be defined on BOTH class models. There is very little difference, but some prefer `back_populates` because it's more explicit and offers some IDE perks_
- **`lazy='subquery'` in Post**: _When defining the relationship, we set `lazy=subquery`. This functionality is similar to `lazy=joined/lazy=False` in that it will load the relationship in the same query, but while `joined/False` loads it with a join statement, `subquery` will perform a second query to populate that field. The pro of `subquery` is that when whenever we ask for a Post, two queries will always be made, but when querying for multiple posts we will not get additional queries._
- **`backref=backref('posts', lazy=True)`**: _If called `backref='posts'`, the relationship on tags will be the same as on pages, which means that the relationship will default to `lazy=subquery`. However, we don't need ALL the posts by default when querying a tag, so we call the backref function with the name and any other relationship parameters we want to establish_


### 🌱 Database Creation

First, we'll want to make sure that we have the database for it to connect to. You can do this either from the command line using `createdb flasql` or going into your `psql` shell and issuing the SQL command `CREATE DATABASE flasql`.

To create the tables, we are going to open an interactive python shell in our virtual environment. The first thing we'll need to do is import our `db` object from `models.py`, then we can run SQLAlchemy.create_all() method to create the tables:

```python
>>> from models import db
>>> db.create_all()
```
Now if you check your database, you'll see FOUR tables!
In this shell, we can seed our database with some folks and posts and tags. First, we'll add three users:
```python
>>> from models import User
>>> users = [User(name='Steve Peters', email='stpets@bigdaddybezos.com', bio='Cranky but cool and caring')]
>>> users.append(User(name='Mike Schull', email='mikey.boi@prettyokay.dev', bio='The reddest raddest dev in town'))
>>> users.append(User(name='Brandi Butler', email='brandi.butler@ga.co', bio='Cats and computers are my jam'))
>>> db.session.add_all(users)
```
None of this will stick in the database until we commit the session. To do that, just type `db.session.commit()` into your python shell.

To make sure our Database works as we want, let's add two posts, one about cats, one about computers. Let's have Brandi write both. You can get creative or copy the ones I wrote below.

```python
>>> from models import Post
>>> db.session.add(Post(
... header='Can Do Cats to decrease WFH stressors', 
... body='Working from home is made better by MORE cats. Just do a google search for cats or visit http://www.placekitten.com.',
... author_id=User.query.filter_by(name="Brandi Butler").one().id
... ))
>>> db.session.add(Post(
... header='Works on My Machine', 
... body='Nothing is more frustrating than trying to get a bug solved, but the person helping you is unable to replicate it. Yikes. Computers suck sometimes.',
... author_id=User.query.filter_by(name="Brandi Butler").one().id
... ))
```

Lastly, we're going to apply the tags 'cats' and 'wisdom' to the first post, and 'computers' and 'wisdom' to the second one.
We accompish this by saving all our posts into a list, then appending the tags to each item.

```python
>>> from models import Tag
>>> posts = Post.query.all()
>>> posts[0].tags += [Tag(tag='cats'), Tag(tag='wisdom')]
>>> posts[1].tags += [Tag(tag='computers'), Tag.query.filter_by(tag='wisdom').one()]
```
We can also add a post by the tag!

```python
>>> computers_tag = Tag.query.filter_by(tag='computers').one()
>>> computers_tag.posts.append(Post(
... header='Python is overrated',
... body='Big snakes can\'t compare to my big intellect! JS 4 lyfe',
... author_id=User.query.filter(User.name.ilike('%Steve%')).one().id,
... tags=[Tag(tag='python')]
... ))
```

And, of course, we commit the session to save it all.
```python
>>> db.session.commit()
```
Once you've commited your session, check your database to make sure that the changes were applied. Once that's confirmed, exit the python shell by typing `quit()`.

## ⏱ Flask Time!

In your `api.py`, we're going to import our flask app from models at the top of the script. Following that, we'll import `jsonify` from flask.

Write a test route to the root route that returns some sort of message.
```python
from models import app
from flask import jsonify

@app.route('/')
def home():
  return jsonify(message='welcome home!')
```

Before we can start our flask server, we need to export our environment and our app. Once we've done that, we can run `flask run`
```bash
export FLASK_ENV=development
export FLASK_APP=api.py
flask run
```

Once you've confirmed that your flask server is running all okay, it's time to test the SQLAlchemy connection. When we hit the home route, instead of sending a message, let's send back the first user we find.

```python
from models import app, User
from flask import jsonify

@app.route('/')
def home():
  first_user = User.query.first()
  print(f'🎀 {first_user}')
  return jsonify(user=first_user)
```

Check out that SWEET error! 
```
Object of type User is not JSON serializable
```
Whaaaa?! When we printed it, it looks like an object, I access the keys like an object! What's the DEAL!? The reason error occurs is because a SQLAlchemy instance is more than just a dictionary, more than just the information from the database; it's chock full of methods and functionality that SQLAlchemy brings with its Base declarative. So how do we send the json back to the front-end?

We're going to add a method to our model classes! This will allow us to call `.as_dict()` on our SQLAlchemy objects and return dictionary version that CAN be serialized. 

Back in `models.py` at the bottom of our User model, define a method called `as_dict()` that will take one parameter, self. There are a few ways to do this, the first is the brute force/hand-crafted way.
```python
def as_dict(self):
  return {
    id: self.id,
    name: self.name,
    email: self.email,
    bio: self.bio
  }
```
This is a totally reasonable and highly functional way of defining this. ESPECIALLY if you want to modify how the front end receives this _(think back to taking out the password in auth when returning an object)_. We'll do that when we get into Post. 

#### ♟Bonus funky Iteration
There is a way to *iterate* over all the columns in the table, and create a dictionary from that. It's inline and *pretty* slick.
```python
def as_dict(self):
    return {c.name: getattr(self, c.name) for c in self.__table__.columns}
```
Let's break this down to see what it's doing.
* **The Inline loop**—This is an instance of a loop essentially returning a value inside an object, creating a new key-value pair for each iteration. The structure is as such: `[some code declaration] for [variable] in [list]`. 
* **What is c?**—So `self.__table__.columns` is clearly a list, because we're iterating through it, but what are each of the items? When calling `type()` on `c` tells us that `c` is an instance of a `sqlalchemy.sql.schema.Column`. It simply prints, for example, `users.id`, but there's more than just that string *(think about our `__repr__` function. Same principle)*. This `sqlalchemy.sql.schema.Column` has a `name` attribute which is a string. We can use this to create the key in the dictionary representation of User.
* **`getattr(self, c.name)`**—This functions the same as `self.id`, but is particularly useful when your key is programatically generated and your object is [not subscriptable](https://stackoverflow.com/questions/216972/what-does-it-mean-if-a-python-object-is-subscriptable-or-not) (like our User object). This function takes two params, the first is the object to query, the second is the name of the key. 

There it is! We can use this single line to create a dictionary representation of our User object, which can then be serialized for JSON useage!

Back in our `api.py`, instead of passing `first_user`, we'll pass `first_user.as_dict()`.

```python
# api.py

from models import app, User
from flask import jsonify

@app.route('/')
def home():
  first_user = []
  return jsonify(user=first_user.as_dict() if first_user else 'No users!')
```

## 🍰 Time to Modularize

If we put all CRUD logic for even one model in our `api.py` file, it's going to start to get unreadable. So we are going to make a series of scripts with CRUD and model specific functions to import into our main server.

In terminal, create a folder called `crud`. In that folder, create three files: `user_crud.py`, `post_crud.py`, and `tag_crud.py`. Nice and self explanitory!

Let's start with full CRUD for our user first. Open `crud/user_crud.py` and import two things at the top of the script:
1. `from flask import jsonify, redirect`
2. `from models import db, User`

Next, we define our first function! We will be writing **5** functions that correspond to our [RESTful Routing](https://docs.google.com/spreadsheets/d/1bT1V8i45ljc-1C4usoWB_nyibZCxxYk8p2fUKF-D2IU/edit?usp=sharing) chart:
| Action | Method | Function Name | Path |
|:------:|:------:|:-------------:|:----:|
| index | GET | `get_all_users()` | /users |
| create* | POST | `create_user()` | /users |
| detail/show | GET | `get_user(id)` | /users/:id |
| update* | PUT | `update_user(id, **kwargs)` | /users/:id|
| delete* | DELETE | `delete_user(id)` | /users/:id |
> The three with asteriks will return a redirect

We will define our GET routes first.

#### 🥇If At first you don't succeed, try except again

First function! We are going to be indexing all the users, so our function will be called `get_all_users()`. In Python, there is built in functionality that—in essence—says, "Run this block of code, and if there are any problems, throw an exception error". It structurally looks like an `if...else...` block, but the keywords are `try` and `exception`.

```python
from employed_alums import gabe

def mock_gabe(can_handle, mock, support):
  try:
    if can_handle(gabe):
      mock(gabe)
    else:
      support(gabe)
    return gabe.happiness_level
  except Exception as error:
    print(f'Error interacting with Gabe')
```
This block of code will **try** to run some functions with our imported gabe instance, but if anything goes wrong, it'll throw an exception which will be stored as the variable `error`, which we can then print to our console and debug.

[More on `try...except...` from the Python docs.](https://docs.python.org/3/tutorial/errors.html#handling-exceptions)

This is good to know, but Flask has a very particular way that it deals with errors.

The GET routes
-

#### Index Users

Let's get all users!

```python
def get_all_users():
    all_users = User.query.all()
    results = [user.as_dict() for user in all_users] 
    return jsonify(results)
```


#### Show User

Our `get_user` function needs an `id` passed to it, but once we have that, it's as simple as calling `User.query.get(id)`.

```python
def get_user(id):
    user = User.query.get(id)
    if user:
      return jsonify(user.as_dict())
    else:
      raise Exception('Error getting user at {}'.format(id))
```

This block of code queries the user by id, then runs a conditional to see if `user` is a truthy value. If it is, it returns the jsonified version of the dictionary representation of our user object *(returned by our `as_dict()` method)*, else it raises an exeption with an error message.

### ⚗️Importing into Flask

To use these functions in our flask server, we need to import them. 
```python
from crud.user_crud import get_all_users, get_user
```

Now we can return the results of these functions when `/users` and `/users/:id` respectively are hit. 

**Before looking at the code below**, try to write these routes out yourself. 
- How do you specify query parameters?
- How would you pass that parameter into a function?

#### Check your `api.py` below
```python
from flask import jsonify
from models import app, User
from crud.user_crud import get_all_users, get_user

# Routes
@app.route('/users')
def user_index_create():
    return get_all_users()

@app.route('/users/<int:id>')
def user_show_put_delete():
    return get_user(id)
```

So what happens when this doesn't work?

#### 🚒 Exception Alert

If there is a problem with this, it will throw what's called an **Exception**. This normally looks like a styled page with flask because it assumes that you'll be doing all this via the browser. This is not super useful for us, as we're writing this 

In our `api.py`, we're going to write a function that will run any time an exception is raised.

```python
from flask import jsonify
from models import app, User
from crud.user_crud import get_all_users, get_user

@app.errorhandler(Exception)
def unhandled_exception(e):
  app.logger.error('Unhandled Exception: %s', (e))
  message_str = e.__str__()
  return jsonify(message=message_str.split(':')[0])
```

The POST routes
-

Coming soon!