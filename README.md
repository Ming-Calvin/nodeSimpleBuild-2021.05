# nodeSimpleBuild-2021.05
# Node后端搭建（express+sequlize)

## 1. 概述

​	首先编写一个 Express Web 服务器开始；接着为 MySQL 数据库添加配置；使用 Sequelize创建模型`Tutorial`并编写控制器；自定义所有请求的路由。

| Methods | Urls                     | Actions                              |
| :------ | :----------------------- | :----------------------------------- |
| GET     | api/tutorials            | 获取所有内容                         |
| GET     | api/tutorials/:id        | 获取`id`对应的内容                   |
| POST    | api/tutorials            | 添加新的内容                         |
| PUT     | api/tutorials/:id        | 更新`id`对应的内容                   |
| DELETE  | api/tutorials/:id        | 删除`id`对应的内容                   |
| DELETE  | api/tutorials            | 删除所有内容                         |
| GET     | api/tutorials/published  | 查找 `published`属性值为`true`的内容 |
| GET     | api/tutorials?title=[kw] | 找到标题包含 `'kw'`的内容            |

## 2. 创建 Node.js 应用程序

### 2.1 创建一个文件夹

```
$ mkdir nodejs-express-sequelize-mysql
$ cd nodejs-express-sequelize-mysql
```

### 2.2 使用`package.json`文件初始化 Node.js 应用程序

```
npm init

name: (nodejs-express-sequelize-mysql) 
version: (1.0.0) 
description: Node.js Rest Apis with Express, Sequelize & MySQL.
entry point: (index.js) server.js
test command: 
git repository: 
keywords: nodejs, express, sequelize, mysql, rest, api
author: bezkoder
license: (ISC)

Is this ok? (yes) yes
```

### 2.3 安装必要的模块：`express`，`sequelize`，`mysql2`和`body-parser`

```
npm install express sequelize mysql2 body-parser cors --save
```

安装后的*package.json*文件：

```json
{
  "name": "nodejs-express-sequelize-mysql",
  "version": "1.0.0",
  "description": "Node.js Rest Apis with Express, Sequelize & MySQL",
  "main": "server.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [
    "nodejs",
    "express",
    "rest",
    "api",
    "sequelize",
    "mysql"
  ],
  "author": "bezkoder",
  "license": "ISC",
  "dependencies": {
    "body-parser": "^1.19.0",
    "cors": "^2.8.5",
    "express": "^4.17.1",
    "mysql2": "^2.0.2",
    "sequelize": "^5.21.2"
  }
}
```

## 3. 设置 Express Web 服务器

### 3.1 创建一个新的server.js文件

```js
const express = require("express");
const bodyParser = require("body-parser");
const cors = require("cors");

const app = express();

var corsOptions = {
  origin: "http://localhost:8081"
};

app.use(cors(corsOptions));

// parse requests of content-type - application/json
app.use(bodyParser.json());

// parse requests of content-type - application/x-www-form-urlencoded
app.use(bodyParser.urlencoded({ extended: true }));

// simple route
app.get("/", (req, res) => {
  res.json({ message: "Welcome to bezkoder application." });
});

// set port, listen for requests
const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}.`);
});
```

### 3.2 模块作用

- Express 用于构建 Rest api
- [body-parser](https://www.npmjs.com/package/body-parser)帮助解析请求并创建`req.body`对象
- [cors](https://www.npmjs.com/package/cors)提供 Express 中间件来启用具有各种选项的 CORS

### 3.3 运行程序

​	创建一个 Express 应用程序，然后添加添加`body-parser`和`cors`中间件，通过`app.use()`方法使用它们。（注意：设置了origin: `http://localhost:8081`用于接收跨域请求）。

​	运行命令`node server.js`可以再`http://localhost:8080/`看到页面

## 4. 配置 MySQL 数据库 & Sequelize

新建一个`app`文件夹，在文件夹中创建一个单独的`config`文件夹，用于使用`db.config.js`文件进行配置

```js
module.exports = {
  HOST: "localhost",
  USER: "root",
  PASSWORD: "123456",
  DB: "testdb",
  dialect: "mysql",
  pool: {
    max: 5,
    min: 0,
    acquire: 30000,
    idle: 10000
  }
};
```

前五个参数用于 MySQL 连接。
`pool`是可选的，它将用于 Sequelize 连接池配置：

- `max`: 池中的最大连接数
- `min`: 池中的最小连接数
- `idle`: 连接在被释放之前可以空闲的最长时间，以毫秒为单位
- `acquire`: 最长时间，以毫秒为单位，该池将在抛出错误之前尝试获取连接

## 5. 初始化Sequelize

在**app** / **models**文件夹中初始化 Sequelize，该文件夹包含之后数据库的模型

### 5.1 创建**app** / **models** / *index.js*

```js
const dbConfig = require("../config/db.config.js");

const Sequelize = require("sequelize");
const sequelize = new Sequelize(dbConfig.DB, dbConfig.USER, dbConfig.PASSWORD, {
  host: dbConfig.HOST,
  dialect: dbConfig.dialect,
  operatorsAliases: false,

  pool: {
    max: dbConfig.pool.max,
    min: dbConfig.pool.min,
    acquire: dbConfig.pool.acquire,
    idle: dbConfig.pool.idle
  }
});

const db = {};

db.Sequelize = Sequelize;
db.sequelize = sequelize;

db.tutorials = require("./tutorial.model.js")(sequelize, Sequelize);

module.exports = db;
```

### 5.2 在server.js 中调用sync()方法

```js
...
const app = express();
app.use(...);

const db = require("./app/models");
db.sequelize.sync();

...
```

### 5.3 删除现有表并重新同步数据库（一般不用）

```js
db.sequelize.sync({ force: true }).then(() => {
  console.log("Drop and re-sync db.");
});
```

## 6. 定义 Sequelize 模型

在`model`文件夹中创建`tutorial.model.js`文件：

这个Sequelize 模型代表MySQL 数据库中的**教程**表。这些列会自动生成：*ID*，*标题*，*描述*，*发布*，*createdAt*，*updatedAt*。

```js
module.exports = (sequelize, Sequelize) => {
  const Tutorial = sequelize.define("tutorial", {
    title: {
      type: Sequelize.STRING
    },
    description: {
      type: Sequelize.STRING
    },
    published: {
      type: Sequelize.BOOLEAN
    }
  });

  return Tutorial;
};
```

> 初始化 Sequelize 后，我们不需要编写 CRUD 函数，Sequelize 支持所有这些函数：
>
> - 创建一个新教程： `create(object)`
> - 通过 id 查找教程： `findByPk(id)`
> - 获取所有教程： `findAll()`
> - 通过 id 更新教程： `update(data, where: { id: id })`
> - 删除教程： `destroy(where: { id: id })`
> - 删除所有教程： `destroy(where: {})`
> - 按标题查找所有教程： `findAll({ where: { title: ... } })`

## 7.创建控制器

新建`app/ controllers`文件夹，在文件夹中创建文件`tutorial.controller.js`

```js
const db = require("../models");
const Tutorial = db.tutorials;
const Op = db.Sequelize.Op;

// Create and Save a new Tutorial
exports.create = (req, res) => {
  
};

// Retrieve all Tutorials from the database.
exports.findAll = (req, res) => {
  
};

// Find a single Tutorial with an id
exports.findOne = (req, res) => {
  
};

// Update a Tutorial by the id in the request
exports.update = (req, res) => {
  
};

// Delete a Tutorial with the specified id in the request
exports.delete = (req, res) => {
  
};

// Delete all Tutorials from the database.
exports.deleteAll = (req, res) => {
  
};

// Find all published Tutorials
exports.findAllPublished = (req, res) => {
  
};
```

### 7.1 创建一条新数据

```js
exports.create = (req, res) => {
  // Validate request
  if (!req.body.title) {
    res.status(400).send({
      message: "Content can not be empty!"
    });
    return;
  }

  // Create a Tutorial
  const tutorial = {
    title: req.body.title,
    description: req.body.description,
    published: req.body.published ? req.body.published : false
  };

  // Save Tutorial in the database
  Tutorial.create(tutorial)
    .then(data => {
      res.send(data);
    })
    .catch(err => {
      res.status(500).send({
        message:
          err.message || "Some error occurred while creating the Tutorial."
      });
    });
};
```

### 7.2 检索数据（有条件）

从数据库中检索所有教程/按标题查找

使用`req.query.title`从请求中获取查询字符串并将其视为`findAll()`方法的条件

```js
exports.findAll = (req, res) => {
  const title = req.query.title;
  var condition = title ? { title: { [Op.like]: `%${title}%` } } : null;

  Tutorial.findAll({ where: condition })
    .then(data => {
      res.send(data);
    })
    .catch(err => {
      res.status(500).send({
        message:
          err.message || "Some error occurred while retrieving tutorials."
      });
    });
};
```

### 7.3 检索单条数据

通过`id`查询数据

```js
exports.findOne = (req, res) => {
  const id = req.params.id;

  Tutorial.findByPk(id)
    .then(data => {
      res.send(data);
    })
    .catch(err => {
      res.status(500).send({
        message: "Error retrieving Tutorial with id=" + id
      });
    });
};
```

### 7.4 更新一条数据

以`id`作为依据

```js
exports.update = (req, res) => {
  const id = req.params.id;

  Tutorial.update(req.body, {
    where: { id: id }
  })
    .then(num => {
      if (num == 1) {
        res.send({
          message: "Tutorial was updated successfully."
        });
      } else {
        res.send({
          message: `Cannot update Tutorial with id=${id}. Maybe Tutorial was not found or req.body is empty!`
        });
      }
    })
    .catch(err => {
      res.status(500).send({
        message: "Error updating Tutorial with id=" + id
      });
    });
};
```

### 7.4 删除数据

根据`id`删除数据：

```js
exports.delete = (req, res) => {
  const id = req.params.id;

  Tutorial.destroy({
    where: { id: id }
  })
    .then(num => {
      if (num == 1) {
        res.send({
          message: "Tutorial was deleted successfully!"
        });
      } else {
        res.send({
          message: `Cannot delete Tutorial with id=${id}. Maybe Tutorial was not found!`
        });
      }
    })
    .catch(err => {
      res.status(500).send({
        message: "Could not delete Tutorial with id=" + id
      });
    });
};
```

删除所有数据：

```js
exports.deleteAll = (req, res) => {
  Tutorial.destroy({
    where: {},
    truncate: false
  })
    .then(nums => {
      res.send({ message: `${nums} Tutorials were deleted successfully!` });
    })
    .catch(err => {
      res.status(500).send({
        message:
          err.message || "Some error occurred while removing all tutorials."
      });
    });
};
```

### 7.5 按条件查找数据

查找属性`published = true`的数据

```js
exports.findAllPublished = (req, res) => {
  Tutorial.findAll({ where: { published: true } })
    .then(data => {
      res.send(data);
    })
    .catch(err => {
      res.status(500).send({
        message:
          err.message || "Some error occurred while retrieving tutorials."
      });
    });
};
```

## 8. 自定义路由

​	当客户端使用 HTTP 请求（GET、POST、PUT、DELETE）向端点发送请求时，需要通过设置路由来确定服务器将如何响应。

在`app/routes`文件夹中创建`turorial.routes.js`文件

```js
module.exports = app => {
  const tutorials = require("../controllers/tutorial.controller.js");

  var router = require("express").Router();

  // Create a new Tutorial
  router.post("/", tutorials.create);

  // Retrieve all Tutorials
  router.get("/", tutorials.findAll);

  // Retrieve all published Tutorials
  router.get("/published", tutorials.findAllPublished);

  // Retrieve a single Tutorial with id
  router.get("/:id", tutorials.findOne);

  // Update a Tutorial with id
  router.put("/:id", tutorials.update);

  // Delete a Tutorial with id
  router.delete("/:id", tutorials.delete);

  // Delete all Tutorials
  router.delete("/", tutorials.deleteAll);

  app.use('/api/tutorials', router);
};
```

在`server.js`中引入路由（在`app.listen()`前）

```js
...

require("./app/routes/turorial.routes")(app);

// set port, listen for requests
const PORT = ...;
app.listen(...);
```

## 9. 参考文档

[Node.js Rest APIs example with Express, Sequelize & MySQL](https://bezkoder.com/node-js-express-sequelize-mysql/#Nodejs_Rest_CRUD_API_overview)



