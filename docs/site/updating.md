# Updating the site

The FuraOJ is under active development, so occasionally you may wish to update. This is a fairly simple process.

!>  The FuraOJ development team makes no commitment to backwards compatibility. It's possible that an update migration
    might add, change, or delete data from your install. Always back up before attempting an update. <br> <br>
    If in doubt, feel free to [contact us on Discord](https://discord.gg/TDyYVyd).

First, switch to the site virtual environment, and pull the latest changes.

```
(furaosite) $ git pull origin master
```

Dependencies may have changed since the last time you updated, so install any missing ones now.

```
(furaosite) $ pip3 install -r requirements.txt
```

The database schema might also have changed, so update it.

```
(furaosite) $ ./manage.py migrate
(furaosite) $ ./manage.py check
```

Finally, update any static files that may have changed.

```
(furaosite) $ ./make_style.sh
(furaosite) $ ./manage.py collectstatic
(furaosite) $ ./manage.py compilemessages
(furaosite) $ ./manage.py compilejsi18n
```

That's it! You may wish to condense the above steps into a script you can run at a later time.
