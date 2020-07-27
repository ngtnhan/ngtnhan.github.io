---
title: "Using middleware in Expressjs"
date: 2020-07-23T14:00:00
categories:
    - programming
tags:
    - nodejs
    - express
---

## Introduction

Middleware function is a task we put in the middle of the Request lifecycle. The middleware can access HTTP Request and Response. Each middleware function will receive three or four parameters. It depends on our needs (I will talk about this later). That four parameters are:

- request
- response
- next function
- error

The "next" function is called when the current middleware is finished. And it can be successful or failure. When the function next is called with a parameter, that means the current middleware is failed. And we usually pass the error to the next function. And if the middleware is successful, we will call the next function without parameter.

## Using middleware

As I said before, the middleware function is a task we put in the Request lifecycle. So two things we should care about middleware is what the task does and where does it put. Let's walk through some examples below for more detail.

### Using middleware for checking authentication

If your server serves some protected API, the middleware function can be an option. Usually, every API will have a controller to handle the request. So to protect that API we can handle it inside the controller by checking the token header or something like that. But this way will add more logic to the controller and make it more complexer. And violate the Single Responsibility principles. And middleware is a option to seperate this logic out of the controller.
Let say we need to protect the API that lets users get their information. But the request must has a valid header "token"

```javascript
// router
app.get('/api/user/me', (req, res, next) => {
  // do something to get the user information
  res.json(user);
});
```

We define the route and the controller to handle that route. Currently, the API is not protected. In the next step, we will define the middleware function and put it in front of the controller. So before the HTTP request reaches the controller, it will be caught by middleware function.

```javascript
// authentication middleware
const ensureAuthenticationMdw = (req, res, next) => {
  // check header exists
  if (!req.headers['token']) return next('NotAuthentication');
  // validation token
  if (!isTokenValid(req.headers['token'])) return next('NotAuthentication');
  // end this middleware by call next fuction
  next();
};

// put mniddleware infront of handler
app.get('/api/user/me',
  ensureAuthenticationMdw,
  // handler
  (req, res, next) => {
    // do something to get the user information
    res.json(user);
  }
)
```

Now our API is protected by the `ensureAuthenticationMdw`. And we can reuse this middleware for other API. In reality, some API need to do some tasks before reach handler. Let say we have an API to update an asset of user. And that API must check authentication, check permission, an asset exists. etc... In this case, we can define another middleware to handle check each condition. And chain it like this.

```javascript
app.patch('/api/user/:userId/assets/:assetId',
  ensureAuthenticationMdw,
  ensurePermission,
  ensureAsset,
  // handler
  (req, res, next) => {
    // do something to get the user information
    res.json(user);
  }
)
```

### Using middleware to validate Request

Let say our system serves an API to update an asset. And before the handler handles that payload update data, we should validate that Request body or params. We can define the validation middleware for this kind of task. And protect our handler from handle the bad data request. There are some library for supporting schema validator like [@hapi/joi](https://hapi.dev/module/joi/) or [express-validator](https://express-validator.github.io/docs/). You can use one of them for this and wrap it by a middleware function.

```javascript
// define middleware to validate the data object
/**
 * @param {JoiSchema} schema
 * @param {string} checkingProp
 */
const validateSchemaMdw = (schema, checkingProp) => (req, res, next) => {
  const { error } = schema.validate(req[checkingProp]);
  // if invalid request
  if (error) return next(error);
  // else we finish this middleware
  next();
}

// schema body for updating asset
// below we define a schema for updating assets and using this to validate the request body
const schema = Joi.object({ username: Joi.string() })

// update asset api
app.patch('/api/user/:userId/assets/:assetId',
  ensureAuthenticationMdw,
  ensurePermission,
  validateSchemaMdw(schema, 'body'),
  ensureAsset,
  // handler
  (req, res, next) => {
    // do something to get the user information
    res.json(user);
  }
)
```


