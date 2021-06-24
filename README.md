<img width="100%" height="150px" src="https://images.unsplash.com/photo-1586769852836-bc069f19e1b6?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=750&q=80" style="object-position:center 50%">

# DynamoDB CRUD Tutorial (ENG)

<img class="icon" width="50px" height="50px" src="https://amazon-dynamodb-labs.com/images/Amazon-DynamoDB.png"/>

# Intro

Although there are many ways of making queries on DynamoDB, in this tutorial we will learn how to query and model data in two different ways: first one with **one table per entity,** which is much similar to traditional modeling data systems(SQL), and the **other one with only one table per project.** For this second one **we will user our Partition Key(PK) to identify our entities and our SortKey(SK) to find our object data ID.**

This second example is widely used by the community, even though it has its cons. One partition of DynamoDB has a storage limit of 10GB of data, therefore every entity will have a max of 10G of data before we need to migrate for the first data modeling method.

For both examples we will be showing all CRUD operations with **Dynamoose ORM.**

# Download DynamoDB Local

1)Go to this AWS link and download your dynamoDB file

[Implantar o DynamoDB localmente em seu computador](https://docs.aws.amazon.com/pt_br/amazondynamodb/latest/developerguide/DynamoDBLocal.DownloadingAndRunning.html)

2. Within the file`s terminal type:

```jsx
java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -sharedDb -port 4567
```

3. This is the code to connect dynamoose to your local dynamoDB

```jsx
const dynamoose = require("dynamoose");
dynamoose.aws.sdk.config.update({
  accessKeyId: "AKID",
  secretAccessKey: "SECRET",
  region: "us-east-1",
});
dynamoose.aws.ddb.local("http://localhost:4567");
```

# Dynamoose

## Install

```jsx
yarn add dynamoose
```

## Creating dynamoose configuration file

In order to use dynamoDB localy, you can put any strings as params, but when you use it in production make sure to pass on your IAM user credentials.

```jsx
const dynamoose = require("dynamoose");

dynamoose.aws.sdk.config.update({
  accessKeyId: "AKID",
  secretAccessKey: "SECRET",
  region: "us-east-1",
});
```

## Creating Models (One Entity per table)

```jsx
const dynamoose = require("dynamoose");
const { v4: uuidv4 } = require("uuid");
const userSchema = new dynamoose.Schema(
  {
    // By default the hashKey will be the first key in the Schema object.
    //hashKey is commonly called a partition key in the AWS documentation.
    id: {
      type: String,
      required: true,
      hashKey: true,
      default: uuidv4,
    },
    login: {
      type: String,
      required: true,
      set: (value) => `${value.toLowerCase().trim()}`,
    },
    name: String,
    age: Number,
    cpf: String,
    phone: String,
    password: { type: String, required: true },
  },
  { saveUnknown: true, timestamps: true }
);

module.exports = dynamoose.model("User", userSchema);
```

## Reference other models as a type(One Entity per table)

```jsx
const dynamoose = require("dynamoose");
const User = require("./User");
const { v4: uuidv4 } = require("uuid");
const gameSchema = new dynamoose.Schema({
  id: {
    type: String,
    required: true,
    hashKey: true,
    default: uuidv4,
  },
  name: { type: String, required: true },
  state: String,
  user: { type: Array, schema: [User] },
});
module.exports = dynamoose.model("Game", gameSchema);
```

## Creating Models (One table per project)

```jsx
const dynamoose = require("dynamoose");
const { v4: uuidv4 } = require("uuid");
const userSchema = new dynamoose.Schema(
  {
    // By default the hashKey will be the first key in the Schema object.
    //hashKey is commonly called a partition key in the AWS documentation.
    table: {
      type: String,
      required: true,
      hashKey: true,
      default: "Users",
    },
    id: {
      type: String,
      required: true,
      // By default the rangeKey won't exist.
      //rangeKey is commonly called a sort key in the AWS documentation.
      rangeKey: true,
      default: uuidv4,
    },
    name: String,
    age: Number,
    login: {
      type: String,
      required: true,
      set: (value) => `${value.toLowerCase().trim()}`,
    },
    cpf: String,
    phone: String,
    password: { type: String, required: true },
  },
  { saveUnknown: true, timestamps: true }
);

module.exports = dynamoose.model("ProjectNameTable", userSchema);
```

## Reference other models as a type (One table per project)

```jsx
const dynamoose = require("dynamoose");
const User = require("./UserOneTableModel");
const { v4: uuidv4 } = require("uuid");
const gameSchema = new dynamoose.Schema({
  // By default the hashKey will be the first key in the Schema object.
  //hashKey is commonly called a partition key in the AWS documentation.
  table: {
    type: String,
    required: true,
    hashKey: true,
    default: "Game",
  },
  id: {
    type: String,
    required: true,
    // By default the rangeKey won't exist.
    //rangeKey is commonly called a sort key in the AWS documentation.
    rangeKey: true,
    default: uuidv4,
  },
  name: { type: String, required: true },
  state: String,
  user: { type: Array, schema: [User] },
});
module.exports = dynamoose.model("ProjectNameTable", gameSchema);
```

# Create

### Creating a user(One table per project)

```jsx
async function createUser(req, res) {
  const { login, password, cpf, name, phone, age } = req.body;
  try {
    const user = await User.create({
      login,
      password,
      cpf,
      name,
      age,
      phone,
    });
    console.log(user);
    res.send(user);
  } catch (error) {
    res.status(500).send(error);
    console.error(error);
  }
}
```

### Creating a Game(One table per project)

```jsx
async function createGame(req, res) {
  const { state, user, name } = req.body;
  //user={id,table:"Users"}
  try {
    const userForInsert = new User(user);
    const userExists = await User.get(userForInsert);
    if (!userExists)
      return res.status(500).send(`User ${user.id} does not exist`);
    const game = await Game.create({
      state,
      name,
      user: [userForInsert],
    });
    console.log(game);
    res.send(game);
  } catch (error) {
    res.status(500).send(error);
    console.error(error);
  }
}
```

### Creating many users at once(One table per project)

```jsx
async function createUsers(req, res) {
  const { usersList } = req.body;
  // usersList = [
  //   {
  //     login1,
  //     password1,
  //     cpf1,
  //     name1,
  //     age1,
  //     phone1
  //   },
  //   {
  //     login2,
  //     password2,
  //     cpf2,
  //     age2,
  //     name2,
  //     phone2
  //   },
  // ];
  try {
    const users = await User.batchPut(usersList);
    console.log(users);
    res.send(users);
  } catch (error) {
    res.status(500).send(error);
    console.error(error);
  }
}
```

### Creating many Games at once(One table per project)

```jsx
async function createGames(req, res) {
  const { gamesList } = req.body;
  // {
  //
  //  gamesList: [
  //     {
  //       state: "Launched",
  //       name: "Battlefield3",
  //       user: { id: "25419715-37b6-4e42-81d4-8cb4b332187f", table: "Users" },
  //     },
  //     {
  //       state: "Launched",
  //       name: "COD",
  //       user: { id: "25419715-37b6-4e42-81d4-8cb4b332187f", table: "Users" },
  //     },
  //   ],
  // };
  try {
    for (let counter = 0; counter < gamesList.length; counter++) {
      const game = gamesList[counter];
      const userForInsert = new User(game.user);
      const userExists = await User.get(userForInsert);
      console.log(userExists);
      if (!userExists)
        return res.status(500).send(`User ${game.user.id} does not exist`);
      game.user = [userForInsert];
    }

    const games = await Game.batchPut(gamesList);
    console.log(games);
    res.send(games);
  } catch (error) {
    res.status(500).send(error);
    console.error(error);
  }
}
```

### Creating a User(One Entity per table)

```jsx
async function createUser(req, res) {
  const { login, password, cpf, name, phone, age } = req.body;
  try {
    const user = await User.create({
      login,
      password,
      cpf,
      age,
      name,
      phone,
    });
    res.send(user);
  } catch (error) {
    res.status(500).send(error);
    console.error(error);
  }
}
```

### Creating a Game(One Entity per table)

```jsx
async function createGame(req, res) {
  const { state, user, name } = req.body;
  //user={id:id}
  try {
    const userForInsert = new User(user);
    const userExists = await User.get(userForInsert);
    if (!userExists) return res.status(500).send(`User ${user} does not exist`);
    const game = await Game.create({
      state,
      name,
      user: [userForInsert],
    });
    console.log(game);
    res.send(game);
  } catch (error) {
    res.status(500).send(error);
    console.error(error);
  }
}
```

### Creating many users at once(One Entity per table)

```jsx
async function createUsers(req, res) {
  const { usersList } = req.body;
  // usersList = [
  //   {
  //     login1,
  //     password1,
  //     cpf1,
  //     name1,
  //     age1,
  //     phone1
  //   },
  //   {
  //     login2,
  //     password2,
  //     cpf2,
  //     age2,
  //     name2,
  //     phone2
  //   },
  // ];
  try {
    const users = await User.batchPut(usersList);
    console.log(users);
    res.send(users);
  } catch (error) {
    res.status(500).send(error);
    console.error(error);
  }
}
```

### Creating many Games at once(One Entity per table)

```jsx
async function createGames(req, res) {
  const { gamesList } = req.body;
  // {
  //
  //  gamesList: [
  //     {
  //       state: "Launched",
  //       name: "Battlefield3",
  //       user:{id:id1},
  //     },
  //     {
  //       state: "Launched",
  //       name: "COD",
  //       user: {id:id2 },
  //     },
  //   ],
  // };
  try {
    for (let counter = 0; counter < gamesList.length; counter++) {
      const game = gamesList[counter];
      const userForInsert = new User(game.user);
      const userExists = await User.get(userForInsert);
      console.log(userExists);
      if (!userExists)
        return res.status(500).send(`User ${game.user.id} does not exist`);
      game.user = [userForInsert];
    }

    const games = await Game.batchPut(gamesList);
    console.log(games);
    res.send(games);
  } catch (error) {
    res.status(500).send(error);
    console.error(error);
  }
}
```

# Populate(One Entity per table)

```jsx
async function FindUserByGame(req, res) {
  const { gameId } = req.body;
  // gameId={id:id1}
  try {
    const searchGame = new Game(gameId);
    const game = await Game.get(searchGame);
    if (!game) return res.status(500).send("Game not found");
    const withoutPopulate = game.toJSON();
    const withPopulate = await game.populate();
    return res.send({ withPopulate, withoutPopulate });
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

# Populate(One table per project)

```jsx
async function FindUserByGame(req, res) {
  const { gameId } = req.body;
  // gameId={id:id1,table:"Game"}
  try {
    const searchGame = new Game(gameId);
    const game = await Game.get(searchGame);
    if (!game) return res.status(500).send("Game not found");
    const withoutPopulate = game.toJSON();
    const withPopulate = await game.populate();
    return res.send({ withPopulate, withoutPopulate });
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

# Delete

## Deleting a user (One Entity per table)

```jsx
async function deleteUser(req, res) {
  try {
    const { id } = req.params;
    await User.delete(id);
    res.status(200).send("user deleted");
  } catch (error) {
    res.status(404).send("user not found");
  }
}
```

## Deleting a user (One table per project)

```jsx
async function deleteUser(req, res) {
  try {
    const { id } = req.params;
    await User.delete({ id, table: "Users" });
    res.status(200).send("user deleted");
  } catch (error) {
    res.status(404).send("user not found");
  }
}
```

## Deleting many at once (One Entity per table)

```jsx
async function deleteUsers(req, res) {
  try {
    const { usersList } = req.body;
    //usersList=[id1,id2]

    await User.batchDelete(usersList);
    res.status(200).send("users deleted");
  } catch (error) {
    res.status(404).send("users not found");
  }
}
```

## Deleting many at once(One table per project)

```jsx
async function deleteUsers(req, res) {
  try {
    const { usersList } = req.body;
    //usersList=[{id:id1,table:"Users"},{id:id2,table:"Users"}]

    await User.batchDelete(usersList);
    res.status(200).send("users deleted");
  } catch (error) {
    res.status(404).send("users not found");
  }
}
```

# Update

## Update by ID(One Entity per table)

update user`s name

```jsx
async function updateName(req, res) {
  try {
    const { name, id } = req.body;
    const user = await User.update({ name, id });
    res.status(200).send(user);
  } catch (error) {
    console.log(error);
    res.status(500).send(error);
  }
}
```

## Update by ID(One table per project)

update user`s name

```jsx
async function updateName(req, res) {
  try {
    const { name, id } = req.body;
    const user = await User.update({ name, id, table: "Users" });
    res.status(200).send(user);
  } catch (error) {
    console.log(error);
    res.status(500).send(error);
  }
}
```

## $SET(One Entity per table)

set found user`s name to Bob

```jsx
async function setName(req, res) {
  try {
    const { id } = req.body;
    const user = await User.update({ id }, { $SET: { name: "Bob" } });
    res.status(200).send(user);
  } catch (error) {
    console.log(error);
    res.status(500).send(error);
  }
}
```

## $SET(One table per project)

set found user`s name to Bob

```jsx
async function setName(req, res) {
  try {
    const { id } = req.body;
    const user = await User.update(
      { id, table: "Users" },

      { $SET: { name: "Bob" } }
    );
    res.status(200).send(user);
  } catch (error) {
    console.log(error);
    res.status(500).send(error);
  }
}
```

## $ADD(One Entity per table)

Adds 32 on the user`s age field

```jsx
async function addAge(req, res) {
  try {
    const { id } = req.body;
    const user = await User.update({ id }, { $ADD: { age: 32 } });
    res.status(200).send(user);
  } catch (error) {
    console.log(error);
    res.status(500).send(error);
  }
}
```

## $ADD(One table per project)

Adds 32 on the user`s age field

```jsx
async function addAge(req, res) {
  try {
    const { id } = req.body;
    console.log(id);
    const user = await User.update(
      { id, table: "Users" },
      { $ADD: { age: 32 } }
    );
    res.status(200).send(user);
  } catch (error) {
    console.log(error);
    res.status(500).send(error);
  }
}
```

## $REMOVE(One Entity per table)

deletes user`s name field

```jsx
async function removeName(req, res) {
  try {
    const { id } = req.body;
    const user = await User.update({ id }, { $REMOVE: ["name"] });
    res.status(200).send(user);
  } catch (error) {
    console.log(error);
    res.status(500).send(error);
  }
}
```

## $REMOVE(One table per project)

deletes user`s name field

```jsx
async function removeName(req, res) {
  try {
    const { id } = req.body;
    const user = await User.update(
      { id, table: "Users" },
      { $REMOVE: ["name"] }
    );
    res.status(200).send(user);
  } catch (error) {
    console.log(error);
    res.status(500).send(error);
  }
}
```

# Querys

### Get(One table per project)

```jsx
async function findById(req, res) {
  const { id } = req.body;
  try {
    const user = await User.get({ table: "Users", id });
    res.send(user.toJSON());
  } catch (error) {
    res.status(500).send(error);
    console.error(error);
  }
}
```

### Get(One Entity per table)

```jsx
async function findById(req, res) {
  const { id } = req.body;
  try {
    const user = await User.get(id);
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
  }
}
```

### Where(One table per project)

### Getting users with where operator

```jsx
async function WhereExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("name")
      .eq("Will")
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Where(One Entity per table)

### Getting users with where operator

```jsx
async function WhereExample(req, res) {
  try {
    const condition = new dynamoose.Condition().where("name").eq("Will");
    const user = await User.scan(condition).exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Limit(One table per project)

### Getting users with where operator limiting search to 10 document reads

```jsx
async function LimitExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("name")
      .eq("Will")
      .limit(10)
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Limit(One Entity per table)

### Getting users with where operator limiting search to 10 document reads

```jsx
async function LimitExample(req, res) {
  try {
    const condition = new dynamoose.Condition().where("name").eq("Will");
    const user = await User.scan(condition).limit(3).exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Atributes(One Entity per table)

### Getting id , name and age from user

```jsx
async function AtributesExample(req, res) {
  try {
    const condition = new dynamoose.Condition().where("name").eq("Will");
    const user = await User.scan(condition)
      .attributes(["id", "name", "age"])
      .exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Atributes(One table per project)

### Getting id , name and age from user

```jsx
async function AtributesExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("name")
      .eq("Will")
      .attributes(["id", "name", "age"])
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Count(One Entity per table)

### Getting all users with name Will

```jsx
async function CountExample(req, res) {
  try {
    const condition = new dynamoose.Condition().where("name").eq("Will");

    const numberOfUsers = await User.scan(condition).count().exec();
    return res.send(numberOfUsers);
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Count(One table per project)

### Getting all users with name Will

```jsx
async function CountExample(req, res) {
  try {
    const numberOfusers = await User.query("table")
      .eq("Users")
      .where("name")
      .eq("Will")
      .count()
      .exec();
    res.send(numberOfusers);
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### OrderBy(One table per project)

### Order users by SK ascending or descending(only two options by default)

```jsx
async function OrderByExample(req, res) {
  try {
    const user = await User.query("table").eq("Users").sort("ascending").exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### OrderBy(One Entity per table)

Once there is no operator sort for Scan, we will use a native js sort method instead

```jsx
async function OrderByExample(req, res) {
  try {
    const user = await User.scan().exec();
    user.sort((a, b) => a - b);
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### ALL(One Entity per table)

### Getting all users even if need to get more then one read capacity

```jsx
async function AllExample(req, res) {
  try {
    const user = await User.query("table").eq("Users").all().exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### ALL(One table per project)

### Getting all users even if need to get more then one read capacity

```jsx
async function AllExample(req, res) {
  try {
    const user = await User.scan().all().exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Contains(One table per project)

### Getting all users that contain "Will" in their name

```jsx
async function ContainsExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("name")
      .contains("Will")
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Contains(One Entity per table)

### Getting all users that contain "Will" in their name

```jsx
async function ContainsExample(req, res) {
  try {
    const user = await User.scan().where("name").contains("Will").exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### And(One Entity per table)

```jsx
async function AndExample(req, res) {
  try {
    const user = await User.scan()
      .where("name")
      .contains("Will")
      .and()
      .where("age")
      .eq(32)
      .exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### And(One table per project)

```jsx
async function AndExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("name")
      .contains("Will")
      .and()
      .where("age")
      .eq(32)
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Equal(One table per project)

```jsx
async function EqualExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("age")
      .eq(32)
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Equal(One Entity per table)

```jsx
async function EqualExample(req, res) {
  try {
    const user = await User.scan().where("age").eq(32).exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Or(One Entity per table)

```jsx
async function OrExample(req, res) {
  try {
    const user = await User.scan()
      .where("name")
      .contains("Will")
      .or()
      .where("age")
      .eq(32)
      .exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Or(One table per project)

Once or condition does not work on Query type, and this model of one table per project needs query to not waste unnecessary reads on other tables with scan, we will user a native js filter method to make this work.

```jsx
async function OrExample(req, res) {
  try {
    const user = await User.query("table").eq("Users").exec();
    const filteredUsers = user.filter((u) => u.name === "Will" || u.age === 60);
    res.send(filteredUsers);
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Not(One table per project)

```jsx
async function NotExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("age")
      .not()
      .eq(32)
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Not(One Entity per table)

```jsx
async function NotExample(req, res) {
  try {
    const user = await User.scan().where("age").not().eq(32).exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Exists(One Entity per table)

Search for all data objects that the field age exists

```jsx
async function ExistsExample(req, res) {
  try {
    const user = await User.scan().where("age").exists().exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Exists(One table per project)

Search for all data objects that the field age exists

```jsx
async function ExistsExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("age")
      .exists()
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Less then(One table per project)

```jsx
async function LessThenExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("age")
      .lt(32)
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Less then(One Entity per table)

```jsx
async function LessThenExample(req, res) {
  try {
    const user = await User.scan().where("age").lt(32).exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Less then or Equal(One Entity per table)

```jsx
async function LessThenOrEqualExample(req, res) {
  try {
    const user = await User.scan().where("age").le(32).exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Less then or Equal(One table per project)

```jsx
async function LessThenOrEqualExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("age")
      .le(32)
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Greater(One table per project)

```jsx
async function GreaterExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("age")
      .gt(32)
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Greater(One Entity per table)

```jsx
async function GreaterExample(req, res) {
  try {
    const user = await User.scan().where("age").gt(32).exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Beggins with(One Entity per table)

```jsx
async function BeginsWithExample(req, res) {
  try {
    const user = await User.scan().where("name").beginsWith("W").exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Beggins with(One table per project)

```jsx
async function BeginsWithExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("name")
      .beginsWith("W")
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### IN(One table per project)

return all users that have Bob or Will in their names

```jsx
async function INExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("name")
      .in(["Will", "Bob"])
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### IN(One Entity per table)

return all users that have Bob or Will in their names

```jsx
async function INExample(req, res) {
  try {
    const user = await User.scan().where("name").in(["Will", "Bob"]).exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Between(One Entity per table)

```jsx
async function BetweenExample(req, res) {
  try {
    const user = await User.query("table")
      .eq("Users")
      .where("age")
      .between(32, 82)
      .exec();
    res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

### Between(One table per project)

```jsx
async function BetweenExample(req, res) {
  try {
    const user = await User.scan().where("age").between(32, 82).exec();
    return res.send(user.toJSON());
  } catch (error) {
    console.error(error);
    return res.status(500).send(error);
  }
}
```

# Official Dynamoose documentation

[https://dynamoosejs.com/](https://dynamoosejs.com/)

# Other Languages for this tutorial

## Português

[felipegenef/DynamodbCrudTutorial-PT-](https://github.com/felipegenef/DynamodbCrudTutorial-PT-)

## English

[felipegenef/DynamodbCrudTutorial-ENG-](https://github.com/felipegenef/DynamodbCrudTutorial-ENG-)
