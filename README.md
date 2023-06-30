![Letus Splash](https://github.com/withjam/letus/raw/main/screens/letus_cropped.jpg)

|||
|---|---|
| ![Letus Screen](https://github.com/withjam/letus/raw/main/screens/posts.png) | ![Letus Filtering](https://github.com/withjam/letus/raw/main/screens/ignoreSettings2.png) |
![Letus Screen](https://github.com/withjam/letus/raw/main/screens/posts2.png) | ![Letus Filtering](https://github.com/withjam/letus/raw/main/screens/ignoreSettings4.png) |
## Project Structure

* `LetusApp` is an Expo project for easy mobile development.  _see the [expo docs](https://docs.expo.io/) if you are new to expo_
* `LetusFunctions` are a set of Azure functions to create a serverless API. _see the [azure functions docs](https://docs.microsoft.com/en-us/azure/azure-functions/functions-develop-local) if you are new to Azure function development_

## Dependencies

* This project relies on Redis Graph - a [Redis Module](https://redis.io/modules). By default, the project expects to use a cloud instance of the database which can be obtained for free at [RedisLabs](https://redislabs.com/)
* This project uses [GCP](https://cloud.google.com/) to both [classify text](https://cloud.google.com/natural-language/docs/classify-text-tutorial) and [analyze sentiment](https://cloud.google.com/natural-language/docs/analyzing-sentiment).  For local development, you should be able to use the free tier for both of these APIs
* This project uses [Azure Funtions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-develop-local) which can be accessed directly in your local development.  If you wish to publish these functions to the cloud, you must create your own [Azure account](https://azure.microsoft.com/en-us/free/). _Tip: There is a very useful [VS Code Extension](https://docs.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=csharp) to help manage your azure projects and functions._
* This project uses [Google Identity](https://developers.google.com/identity/protocols/oauth2) to authenticate users via web-based **Oauth 2.0** Follow the [Expo Google Auth](https://docs.expo.io/guides/authentication/#google) documentation to create your **ClientID**, **JavaScript origin**, and **authorized redirect URI**. _Tip: follow instructions for the [Expo Go](https://expo.io/client) app as they are the easiest steps.  If you are not using the Expo Go app during development, follow the steps based on your dev environment._

### Recomended

* NodeJS runtime (version based on [Azure functions support](https://docs.microsoft.com/en-us/azure/azure-functions/functions-versions)
* VS Code
* VS Code Azure Functions extension

## Development Setup

* Clone this repository

### LetusApp (Expo)

* `cd LetusApp`
* `npm install --global expo-cli`
* `expo install`
* create a `.env.local` file to store your local configuration
* set the value of `LETUS_API_URL` to the location of your API (can use `.env` if local functions)
* set the value of `GOOGLE_WEB_CLIENT_ID` to use a Web OAUTH credential created in your Google Identity account.
* set value of firebase config in the `.env` or `.env.local` file
  * FIREBASE_API_KEY=
  * FIREBASE_AUTH_DOMAIN=
  * FIREBASE_PROJECT_ID=
  * FIREBASE_STORAGE_BUCKET=
  * FIREBASE_MEASSGE_SENDER_ID=
  * FIREBASE_APP_ID=
* `expo start`
* press `i` to launch iOS simulator or `a` for android.  (note: android simulator requires Android Studio and sdk setup)

### LetusFunctions

* Azure functions can be run locally.  The recommended option is to use [VS Code Extension](https://docs.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=csharp) and then use `f5` to launch in debug mode
* Configure the following environment variables (either in `local.settings.json` or in Azure)
  * `REDIS_HOST` The host of the Redis Graph database
  * `REDIS_PORT` The port used by the Redis Graph database
  * `REDIS_PASS` Password for the default Redis user
  * `REDIS_GRAPH` The name of the graph to use
  * `GCP_API_KEY` Google Cloud Platform API Key used for Language processing (`AnalyzeSentiment` and `ClassifyText`)
* Add your `serviceAccount.json` file to `LetusFunctions/shared` to configure the API's firebase-admin

### Authentication

* Letus uses Firebase for authentication in both the Expo app and the Azure functions
* [Create a project for Firebase Auth (web)](https://firebase.google.com/docs/auth/web/start)

### Redis Graph (on RedisLabs)

* [Create a Redis Graph instance](https://docs.redislabs.com/latest/modules/redisgraph/redisgraph-quickstart/)

### Express local setup (local Expo Go App only)

You can run just the Expo Go App localy to use, and develop, the react native app. By accessing existing (free) firebase auth and existing (free) Azure functions deployed for DEV only.

* Copy the contents of `env.dev.example` into your `.env` file
* Launch the app via `expo start`
* Register a new user via the mobile simulator and you're in!
## Redis Graph Commands

_Note:  App was developed using free Redis Enterprise Cloud database with **Redis Graph** module: redis-12183.c251.east-us-mz.azure.cloud.redislabs.com_

![Redis Insight](https://github.com/withjam/letus/raw/main/screens/redisInsight.png)

* `GetPosts` - The core of Letus.  Uses cypher to traverse the current user's network and return the relevant, most recent posts:
  ```
  MATCH (me:Person {userid: $userid})
    OPTIONAL MATCH (me)-[:ignores]->(ign:IgnoreSetting)
    WITH me, ign
    MATCH (poster:Person)-[:posted]->(post:Post)
    WHERE (poster = me OR (poster)-[:friended]-(me))
    AND (NOT (post)-[:inCategory]->(:Category {name:ign.category}) AND NOT (post)<-[:posted]-(:Person {userid:ign.poster}))
    WITH post, poster, me
    OPTIONAL MATCH (post)-[:hasComment]->(comment:Comment)<-[:commented]-(commenter:Person)
    WHERE commenter = me OR (commenter)-[:friended]-(me)
    WITH me, post, poster, collect(comment) as comments, collect(commenter) as commenters
    ORDER BY post.created DESC
    SKIP $skip
    LIMIT $limit
    RETURN post, poster, comments, commenters
    ```
* `CreatePost` - Creates a new post with additional information from NLP processing.
    ```
    MATCH (me:Person {userid:$userid}) ${categories
    .map(
    (cat, index) =>
        'MERGE (cat' + index + ':Category {name: $cat' + index + '})'
    )
    .join(
    ' '
    )}
    MERGE (sentiment:Sentiment {name: $sentiment})
    CREATE (me)-[:posted]->(post:Post {text:$text,created:$now})
    MERGE (post)-[:hasSentiment]->(sentiment) ${categories
    .map((cat, index) => 'MERGE (post)-[:inCategory]->(cat' + index + ')')
    .join(' ')}
    RETURN post
    ```
* `AddComment` - Add your comment to a connection's Post.
    ```
    MATCH (me:Person), (post:Post)
    WHERE me.userid = $userid
    AND ID(post) = $onPost
    CREATE (me)-[:commented]->(comment:Comment {text:$text,created:$now})
    CREATE (post)-[:hasComment]->(comment)
    RETURN post
    ```
*  `AddFriend` - Add a new connection from your direction. Note: single-direction `:friended` relationships determine pending friend requests
    ```
    MATCH (me:Person { userid: $userid })
    MATCH (them:Person { userid: $themid })
    MERGE (me)-[:friended]->(them)
    RETURN me, them
    ```
* `GetFriendRequests` - Return the list of `Person` nodes who have `:friended` you but are not friended by you.
    ```
    MATCH (me:Person {userid: $userid})
    WITH me
    MATCH (them:Person)-[:friended]->(me)
    WHERE NOT (me)-[:friended]->(them)
    RETURN them
    ```
* `AddIgnoreSetting` - Add a new IgnoreSetting to a User
    ```
    MATCH (me:Person { userid: $userid })
    MERGE (me)-[:ignores]->(ignore:IgnoreSetting {poster:$themid, category: $category, sentiment: $sentiment})
    RETURN me, ignore
    ```
* `FindFriends` - Search for unfriended People by name
  ```
  MATCH (them:Person)
  MATCH (me:Person {userid:$userid})
  WHERE NOT them.userid = $userid
  AND them.name STARTS WITH $name
  AND NOT (me)-[:friended]->(them)
  RETURN them
  ```

## NLP Commands
Using the free tier of [GCP Natural Language](https://cloud.google.com/natural-language), we apply both sentiment analysis and text classification to all posts made in the system.

* `AnalyzeSentiment`
    ```
    const sslCreds = getApiKeyCredentials();
    const client = new language.LanguageServiceClient({ sslCreds });

    const document = {
      content: text,
      type: 'PLAIN_TEXT',
    };

    const [result] = await client.analyzeSentiment({ document: document });
    sentiment = mapSentiment(result.documentSentiment.score || 0);
    ```

* `ClassifyText` - Returns a list of matching [NLP Categories](https://cloud.google.com/natural-language/docs/categories)
    ```
    let content = text;
    // GCP requires 20 words
    // we pad with prepositions if between 10-19 words for max coverage
    if (len < 20) {
      content = [...content.split(' '), ...preps.slice(0, 20 - len)].join(' ');
    }
    const sslCreds = getApiKeyCredentials();
    const client = new language.LanguageServiceClient({ sslCreds });

    const document = {
      content,
      type: 'PLAIN_TEXT',
    };

    const [result] = await client.classifyText({ document: document });
    categories = result.categories || [];
    ```