---
title: "Writing a Custom Database in Python"
date: 2018-08-01T16:04:56-04:00
draft: true
tags: ["python"]
menu:
  main:
    parent: tutorials

---

In this post, we'll be discussing about writing our own custom database in Python.

Our database will be quite simple and we will be using LevelDB as our storage engine. LevelDB is a really fast key-value store developed at Google. This is the actual engine that does storing of our data, and it supports the common get, put, delete and iterate operations, and it does it super fast. And our code is going to make a wrapper around this and provide functionality we want incorporated.

To start, we need this library called "[Plyvel](https://plyvel.readthedocs.io/en/latest/installation.html)". This is our python interface to the leveldb core  written in C++.

```
>>> import plyvel
>>> db = plyvel.DB('/tmp/testdb/', create_if_missing=True)`
```
And to store data:
```
>>> db.put(bytes("name"), bytes("goku"))
>>> db.put(bytes("age"), bytes("40"))
```

And to retrieve data:
```python
>>> db.get(bytes("name"))
cryps
```

You can also go through all the data:
```
>>> for key, value in db:
...     print key + " : " + value
```

And our result:
```markdown
  name : goku 
  age  : 40 
```

You will notice I'm using 'bytes()' here because the keys and values have to be in byte characters.

You can read the whole documentation [here](https://plyvel.readthedocs.io/en/latest/user.html).

**Working with our DB:**
So we see that we can store and retrieve stuff. But what can we actually do with just key and value pairs?

Storing data will be a little different than what we are used to with other databases. So let's suppose we want to store data for books in our database.

We have the book **isbn number**, **title**, **author** and **genre**.

You can store this data any way you want but I'm just going to show one way of doing things.

I want my application to mainly center around genre of books so, my key would be structured like:

**book:genre:book_id:title**
**book:genre:book_id:author**

So for each I'd structure the keys string in this format.

Eg.
```
db.put(b'book:fantasy:9788580410631:title', b'Name of the Wind')

db.put(b'book:fantasy:9788580410631:author', b'Patrick Rothfuss')
```

Now if I want to get all the books of one genre from the database: 

```python
def getByGenre(genre):
    prefix = 'book:'+genre
    for key, value in db.iterator(start=bytes(prefix)):
        print value
```

We can then call:
```
>>> getByGenre('fantasy')
```

And we get this result:
```markdown
  Name of The Wind
  Patrick Rothfuss
```

We can do more stuff here:

```python

def getByGenre(genre):
    prefix = 'book:' + genre
    with db.iterator() as it:
        for k, v in it:
            print 'Name: ', v
            k,v = it.next()
            print 'Author: ', v
            
```

This is a very basic example, but basically you can customize this to your requirements. LevelDB is very fast so you can do complex operations, and it will work good for even million of data.
