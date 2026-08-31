# Contest data downloads

The FuraOJ allows contest authors to download contest data. At the time of writing, only submission data can be downloaded.

By default, this feature is disabled. To enable, uncomment the relevant settings in `local_settings.py`.

```python
# Uncomment to allow contest authors to download contest data
DMOJ_CONTEST_DATA_DOWNLOAD = True

# Directory to cache contest data downloads.
# It is the administrator's responsibility to clean up old files.
DMOJ_CONTEST_DATA_CACHE = '/home/dmoj-uwsgi/contestdatacache'

# Path to use for nginx's X-Accel-Redirect feature.
# Should be an internal location mapped to the above directory.
DMOJ_CONTEST_DATA_INTERNAL = '/contestdatacache'

# How often contest data can be exported.
# This applies per contest, not per user.
DMOJ_CONTEST_DATA_DOWNLOAD_RATELIMIT = datetime.timedelta(days=1)
```

Also, uncomment the relevant section in your Nginx configuration if you wish to take
advantage of Nginx's [X-Accel-Redirect](https://www.nginx.com/resources/wiki/start/topics/examples/x-accel/#x-accel-redirect)
feature.

```nginx
# Uncomment if you are allowing contest data downloads and want to serve it faster.
# This location name should be set to DMOJ_CONTEST_DATA_INTERNAL.
location /contestdatacache {
    internal;
    root <path to data cache directory, without the final /contestdatacache>;

    # Default from local_settings.py:
    #root /home/dmoj-uwsgi/;
}
```

These data files are not cleaned up automatically. Although each contest can have at most one data file
stored on the server at a time, you may want to clean up old files. A Cron job should suffice:

```
0 */4 * * * find /home/dmoj-uwsgi/contestdatacache/ -type f -mtime +2 -delete
```

This Cron job will delete files older than 2 days every 4 hours. You are recommended to tweak these
values according to your ratelimit.

You should now find a link on your _Edit Profile_ that allows you to download your data,
along with various configuration options.
