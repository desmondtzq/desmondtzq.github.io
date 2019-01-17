---
layout: post
title: "App migration"
tags: [migration]
comments: true
---

Just like changes in the schema of a database, sometimes there will be changes in the app that are not compatible with previous versions of the app, and a "migration" will be required to ensure the latest version of the app can function correctly.

## What is a migration?

In the context of an app, it is simply code that has to be run once in order to update the state of the app to be compatible with the latest version of the app.

There can be multiple reasons why a migration is required. Perhaps previously you have (ab)used SharedPreferences to store data, and now you want to transfer and store it properly in a database. Or you have used the AlarmManager to schedule work to be done, but now in the new version of the app you decided that you don't need it anymore and want to cancel the alarms that were scheduled previously.

Whatever the case may be, you will probably want to manage the changes in the new version of the app, and not worry about the older versions going forward.

## The easy solution

Unlike a database, there is no standard way or tools provided to perform migrations for an app. And because changes like these do not always occur, it can be tempting to write the migration logic right where they are needed. 

```kotlin
class Application {

    override fun onCreate() {
        // ...
        performMigration()
    }

    private fun performMigration() { ... }
}

class Activity {

    override fun onCreate() {
        // ...
        anotherMigration()
    }

    private fun anotherMigration() { ... }
}
```

While it works, these migration will be running more than once, and unnecessarily so. It pollutes the classes with methods that quickly becomes irrelevant as the version of the app increases, and also makes it difficult to test the migrations in isolation. 

## A better solution

Good news is that there is nothing stopping us from learning from how database schema perform their migrations and perform app migrations in a similar manner.

Let's begin by defining what we want:
- all migrations in a single place
- run only once, and only when necessary
- easy to test

### All migrations in a single place

We don't want logic to be scattered all over, but to keep them in a single place, seperate from the everything else. We can start by defining a class that run all the migrations that is needed.

```kotlin
class MigrationManager {

    fun runMigrations() {
        // run all the migrations!
    }
}
```

### Easy to test

We also want to ensure that each piece of logic can be tested in isolation, so we will probably write the logic separately as well. In order for our `MigrationManager` to run all the different logic, we can define an interface for a migration, and have each piece of logic implement the interface.

```kotlin
interface Migration {
    fun run()
}

class MigrationA(private val dependencyA: DependencyA) : Migration {
    override fun run() {
        // logic for A
    }
}

class MigrationB(private val dependencyB: DependencyB) : Migration {
    override fun run() {
        // logic for B
    }
}
```

With each migration in separate classes, we can test each of them seperately in isolation. 

Now our `MigragtionManager` can run all of them easily:

```kotlin
class MigrationManager(private val migrations: List<Migration>) {

    fun runMigrations() {
        migrations.forEach(Migration::run)
    }
}
```

### Run only once, and only when necessary

Now for the last part. To ensure that the migrations run only once when necessary, the migrations will have to be tagged with a version code that indicates the version of the app that the migration is introduced. We can then compare the last saved version code with the migration's version code to decide whether there is a need to run them. In order for this to work, we also have to save the version code of app after the migrations are done.

The psuedo code may illustrate it more clearly:
```
previousVersionCode = get last saved version code from preferences, or 0 if not available

for each migration:
    if migration.versionCode > previousVersionCode, then:
        run migration

save current app version code to preferences
```

## Putting it all together

Actual code can look something like this: 

```kotlin
interface Migration {
    val versionCode: Int

    fun run()
}

class MigrationManager(
    private val migrations: List<Migration>, // list of all migrations ever defined
    private val sharedPreference: SharedPreferences,
    private val appVersionCode: Int // the current app version code
) {

    fun runMigrations() {
        val previousVersionCode = sharedPreference.getInt("latest_version_code", 0)

        migrations
            .takeIf { it.versionCode > previousVersionCode }
            .forEach(Migration::run)

        sharedPreference.edit {
            putInt("latest_version_code", appVersionCode)
        }
    }
}
```

Now we have a mechanism for managing migrations. Hopefully it can help keep the codebase clean and maintanable.
