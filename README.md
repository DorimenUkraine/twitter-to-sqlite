# twitter-to-sqlite

[![PyPI](https://img.shields.io/pypi/v/twitter-to-sqlite.svg)](https://pypi.org/project/twitter-to-sqlite/)
[![CircleCI](https://circleci.com/gh/dogsheep/twitter-to-sqlite.svg?style=svg)](https://circleci.com/gh/dogsheep/twitter-to-sqlite)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](https://github.com/dogsheep/twitter-to-sqlite/blob/master/LICENSE)

Save data from Twitter to a SQLite database.

## How to install

    $ pip install twitter-to-sqlite

## Authentication

First, you will need to create a Twitter application at https://developer.twitter.com/en/apps

Once you have created your application, navigate to the "Keys and tokens" page and make note of the following:

* Your API key
* Your API secret key
* Your access token
* Your access token secret

You will need to save all four of these values to a JSON file in order to use this tool.

You can create that tool by running the following command and pasting in the values at the prompts:

    $ twitter-to-sqlite auth
    Create an app here: https://developer.twitter.com/en/apps
    Then navigate to 'Keys and tokens' and paste in the following:

    API key: xxx
    API secret key: xxx
    Access token: xxx
    Access token secret: xxx

This will create a file called `auth.json` in your current directory containing the required values. To save the file at a different path or filename, use the `--auth=myauth.json` option.

## Retrieving tweets by an account

The `user-timeline` command retrieves all of the tweets posted by the specified user account. It defaults to the account belonging to the authenticated user:

    $ twitter-to-sqlite user-timeline twitter.db
    Importing tweets  [#####-------------------------------]  2799/17780  00:01:39

It assumes there is an `auth.json` file in the current director. You can provide the path to your `auth.json` file using `-a`:

    $ twitter-to-sqlite user-timeline twitter.db -a /path/to/auth.json

To load tweets for another user, use `--screen_name`:

    $ twitter-to-sqlite user-timeline twitter.db --screen_name=cleopaws

Twitter's API only returns up to around 3,200 tweets for most user accounts, but you may find that it returns all available tweets for your own user account.

## Retrieve accounts in bulk

If you have a list of Twitter screen names (or user IDs) you can bulk fetch their fully inflated Twitter profiles using the `users-lookup` command:

    $ twitter-to-sqlite users-lookup users.db simonw cleopaws

You can pass user IDs instead usincg the `--ids` option:

    $ twitter-to-sqlite users-lookup users.db 12497 3166449535 --ids

## Retrieving Twitter followers

The `followers` command retrieves details of every follower of the specified account. You can use it to retrieve your own followers, or you can pass a screen_name to pull the followers for another account.

The following command pulls your followers and saves them in a SQLite database file called `twitter.db`:

    $ twitter-to-sqlite followers twitter.db

This command is **extremely slow**, because Twitter impose a rate limit of no more than one request per minute to this endpoint! If you are running it against an account with thousands of followers you should expect this to take several hours.

To retrieve followers for another account, use:

    $ twitter-to-sqlite followers twitter.db --screen_name=cleopaws

See [Analyzing my Twitter followers with Datasette](https://simonwillison.net/2018/Jan/28/analyzing-my-twitter-followers/) for the original inspiration for this command.

## Retrieving Twitter list memberships

The `list-members` command can be used to retrieve details of one or more Twitter lists, including all of their members.

    $ twitter-to-sqlite list-members members.db simonw/the-good-place

You can pass multiple `screen_name/list_slug` identifiers.

If you know the numeric IDs of the lists instead, you can use `--ids`:

    $ twitter-to-sqlite list-members members.db 927913322841653248 --ids

## Retrieving just follower and friend IDs

It's also possible to retrieve just the numeric Twitter IDs of the accounts that specific users are following ("friends" in Twitter's API terminology) or followed-by:

    $ twitter-to-sqlite followers-ids members.db simonw cleopaws

This will populate the `following` table with `followed_id`/`follower_id` pairs for the two specified accounts, listing every account ID that is following either of those two accounts.

    $ twitter-to-sqlite friends-ids members.db simonw cleopaws

This will do the same thing but pull the IDs that those accounts are following.

Both of these commands also support `--sql` and `--attach` as an alternative to passing screen names as direct command-line arguments. You can use `--ids` to process the inputs as user IDs rather than screen names.

The underlying Twitter APIs have a rate limit of 15 requests every 15 minutes - though they do return up to 5,000 IDs in each call. By default both of these subcommands will wait for 61 seconds between API calls in order to stay within the rate limit - you can adjust this behaviour down to just one second delay if you know you will not be making many calls using `--sleep=1`.

## Design notes

* Tweet IDs are stored as integers, to afford sorting by ID in a sensible way
* While we configure foreign key relationships between tables, we do not ask SQLite to enforce them. This is used by the `following` table to allow the `followers-ids` and `friends-ids` commands to populate it with user IDs even if the user accounts themselves are not yet present in the `users` table.
