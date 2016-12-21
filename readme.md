# Sequelize

## Learning Objectives

- Explain the purpose of an ORM
- Use sequelize to connect to a DB in JS
- Define a model in sequelize that has getters and setters
- Define the roll of callbacks in Sequelize
- Describe why database are asynchronous (and what asynchronous means)
- Use Sequelize to find models from the DB
- Use Sequelize to perform CRUD on model instances
- Create an API with CRUD functionality using Express and Sequelize
- Map a directory / file structure conducive to DB modeling
- Map Rails / ActiveRecord methods to Express / Sequelize methods

## Today's class...

...is about how we're going to do SQL databases in Node. We're going to cover this by exploring an app called Tunr.

### Sequelize

For Mongo, we used Mongoose to interact with the database. For SQL, we're going to use Sequelize.

Sequelize is an ORM (Object Relational Mapping). It can be used with MySQL, SQLite, and Postgres databases, and more.

### Why don't we just use MongoDB?

Because Mongo isn't a relational database. It stores data as big blobs of JSON. That's useful if you want to mostly store huge chunks of data -- documents, images, and so on.

But we're much more concerned with small bits of data that we want to be able to relate to each other: Artists and Songs, Posts and Comments, Doctors and Patients, and so on.

Doing that is a sonic pain in Mongo.

Sequelize can use **Postgres** -- what we were using yesterday -- which is really nice because it's familiar and we can still poke around in the database using `psql`.

## Let's start at the end

Fork and clone [tunr_node_hbs](https://github.com/den-wdi-2/tunr_node_hbs).

### Setup:
```bash
npm install
createdb tunr_db
node db/migrate.js
node db/seeds.js
nodemon app.js
```

Let's walk through those last few steps.

## Setting up the database

### How is the database created in the Node app?

`createdb` is a PostgreSQL command that creates a new database, in this case one called `tunr_db`.

Then, Sequelize has a really nice way of managing the database tables.  It's in `migrate.js`.

### So we open the db/migrate.js file to see...

Not much.

There's basically just a sleek method called "sync". You can set up your models to "sync" with the database: whenever you run the sync script, Sequelize will look at all of your models and alter their tables to fit the model.

So if you have a User model with an "email" field, the app will make sure the "users" table has an "email" column. Delete the "email" field in the model, and the next time the app starts up it'll remove the "email" column.

### Note:
- The `force: true` tells Sequelize to drop all rows in all existing tables in the database.
- The `process.exit()` tells Node, "OK, stop doing stuff now." Otherwise, Node is going to just keep running and waiting for something to happen until you turn off your server. You can call this *anywhere* in your app.

### TLDR, to set up the database...

...run `node db/migrate.js`

### Wild guess as to how we'll seed the database?

Run `node db/seeds.js` and away you go!

Check `psql` to verify all the data is there.

Better yet, run `nodemon app.js` and go to `localhost:3000`!

## You do:

With your table, take 30 minutes to turn Tunr into an app that does something completely different -- just by changing the names and fields of the models.

This app gives you two models with a has-many and belongs-to relationship. Instead of Artists and Songs, it could just as easily be Movies and Reviews, Breweries and Beers, Kardashians and Significant Others, the list goes on.

*Don't create any new files. Don't create any new functions. Don't delete and create model fields. Just replace words.*

In half an hour, we'll go around the room and ask one person from each table to say how their table changed Tunr.

### As you do this

Write down any Sequelize methods you come across. We'll pool what we found and make a cheat sheet for the class.

### Keep in mind

As you have it in front of you, the app works just fine. If all you're doing is replacing words, that means any errors you encounter will almost certainly be caused by typos. So when you get an error, **don't thrash**. Just `console.log` line-by-line until you find the line where your app breaks, and, thus, the line where you have a typo.

### Afterward

##### What Sequelize methods did you write down?
-----

## Now that you have a decent idea of how this app is layed out...

I'm going to go through Tunr and explain the things you encountered.

## seeds.js

Looking at the top of this file, I've `require`d two big JSON files.

I've saved them into a variable called Seeds, but I could call it anything. *Doesn't matter.*

Now things get weird.

### Asynchronicity

You see all of those `.then`s? This should look familiar from our *asynchronous* calls in jQuery and Angular.

The rest of this file is a bunch of *asynchronous* things happening -- that is, a bunch of events starting at the same time, but not necessarily finishing in the same order.

Javascript starts executing whatever's written on one line, and doesn't wait for that execution to stop before starting the next line.

Sometimes this is good: if you have a Javascript slideshow loading a whole bunch of images, you don't want your entire webpage to hang until the last image is loaded. You want the image loading and the rest of your Javascript to happen asynchronously.

However, there are certainly times during which you want *asynchronous* things to happen *synchronously*. So you chain them together in such a way that events happen in a specific order: one event can't start until the previous event has completed. (Remember those "promise" objects from Angular?)

In this case, the asynchronous events that we're forcing to happen synchronously are CRUD interactions with the database.

In Node, if you were to run:
```js
DB.models.Artist.bulkCreate(Seeds.artists);
DB.models.Artist.findAll();
```
...the second line would start running **before** the first line had stopped. That is, Node would try to retrieve all the artists before they'd been created, which wouldn't work.

(This is **annoying**.)

I didn't *have* to use `.then`. I could have used...

### Nested callbacks

That would result in something like...

```js
DB.models.Artist.bulkCreate(data.artists).then(function(){
  DB.models.Artist.findAll().then(function(artists){
    DB.models.Song.bulkCreate(songs).then(function(){
      process.exit();
    })
  });
});
```

...which is a few steps away from...

![PHP Hadouken](http://i.imgur.com/BtjZedW.jpg)

...which is called:

### Callback Hell

You wake up one day and find you have 47 nested callbacks, have to hit "tab" 47 times for each new line, and your wife has left you and your house has burned down.

It makes no difference from a performance standpoint. It's *real aggravating* from a readability standpoint.

### What's actually going on in this file

1. We create a bunch of Artists from seed data.
- **Then**, from the database, we retrieve all the artists we just created.
- **Then**, using the artists we retrieved, we take the Song seed data, loop through it, and figure out the ID of the artist to which each song belongs. Then we create a bunch of Songs from that updated data.
- **Then** we tell Node to shut off.

### Note:
- If you're going to have chained `then`s, the function inside each `then` needs to `return` the asynchronous function inside it.
- You can chain object methods and properties across multiple lines to make your code more readable. That is:

This:
```js
Object.doSomething().doSomethingElse();
```
...is the same as:
```js
Object
.doSomething()
.doSomethingElse()
```

## Models

...are pretty similar to Mongoose models, only they have different magic words.

Sequelize has [all the data types you would expect](http://docs.sequelizejs.com/en/latest/api/datatypes/). It's kind of annoying to have to write `Sequelize.STRING` instead of just `string`, but such is life.

As you can see from the very useful example in the `Artist` model, you can add custom functions to models in Sequelize just as you did in Mongoose.

The three really different things are...

### No more scheams

We put that structure right inside the model.

### Where associations go

In Mongoose, we could put our associations right inside the schema like this:

```js
var foodSchema = new Schema({
  name: {
    type: String,
    default: ""
  },
  ingredients: [{
    type: Schema.Types.ObjectId,  // NOTE: Referencing
    ref: 'Ingredient'
  }]
});
```

In Sequelize, you must define your associations *after all your models have been loaded*.

In this app, I defined my associations in the `connection` file.

### Where you put instance methods

In Mongoose, we would say something like:

```js
  User.methods.encrypt = function(password) {
    return bcrypt.hashSync(password, bcrypt.genSaltSync(8), null);
  };

```

But in Sequelize, we define them explicitly inside the model:

```js
var model = sequelize.define("artist", {

},
{
  instanceMethods: {
    shout: function(){
      console.log("My name is " + this.name + "!");
    }
  }
}
);
```

## Connecting to the database

At the top of both the migration file and the seeds file a `connection` file is `require`d.

Let's take a look at that file.

### connection.js

The first two lines give us an object representing Sequelize itself, and an object representing Sequelize's connection to my database. The convention for naming these is bizarre and confusing in that it seems to be to use the word "sequelize" for both, one capitalized and the other not. Such is life.

Next, we load the models.

Then, we define the **associations** for the models. Note that I've arranged it this way so the associations are set up after the models have been loaded.

Again, in Sequelize you have to define your associations after all your models have been loaded. I can't put `Artist.hasMany(Song)` in my Artist model because if Node hasn't loaded the Song model yet it doesn't know what it should be linking Artists to.

I end the file by making a big ol' object into which I'll put the things I need later: the two (Ss)equelizes and my models.

## Controllers

...follow pretty much the exact same concepts as the `seeds.js` file with chained "thens".

## Views

...in this case use Handlebars, which we'll cover later.

## Kwiz Kwestions

- List 1 Sequelize method.
  <!-- findById-->
- How do callbacks and asynchronicity factor into Sequelize?
  <!-- Sequelize methods are asynchonous, so you need to provide a callback to say what should happen after a given method is doen.-->
- Where should you declare associations in a Sequelize app? ( Hint: Before or after you require your models? )
  <!-- After you declare your models, ideally right after they're loaded when the app starts.-->
- There are 3 different ways to set up your database tables in a Sequelize app. What are 2?
  <!-- schema.sql
  - `sequelize db:migrate`
  - `sequelize.sync`-->
