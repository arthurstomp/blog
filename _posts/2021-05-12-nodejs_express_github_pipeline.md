---
layout: post
title:  "NodeJS + Typescript + Express + Jest + Github CI"
date:   2021-05-18 19:00:00 -0300
categories: programming
---
> Project source code: [https://github.com/arthurstomp/ts_express_template](https://github.com/arthurstomp/ts_express_template)

> This post is heavily copied from [Jhon O. Paul's post](https://medium.com/the-andela-way/how-to-set-up-an-express-api-using-webpack-and-typescript-69d18c8c4f52). Thanks Jhon. I just extend it to add Jest and Github CI for it.

## Create project directory

```shell
$ mkdir api
$ cd api
$ yarn init -y # start package.json with default options. `-y` to say 'yes' to all options.
$ mkdir src
$ touch src/index.ts # create entrypoint file.
$ touch webpack.config.js # create file for webpacker configuration.
$ touch tsconfig.json # create file for typescript configuration.
$ yarn add ts-loader typescript webpack --dev # install minimal dev dependencies
```

Configure source and destination for webpacker builds.

```javascript
// webpack.config.js

const path = require('path');

const {
  NODE_ENV = 'production',
} = process.env;

module.exports = {
  entry: './src/index.ts', // entrypoint file
  mode: NODE_ENV,
  target: 'node',
  output: {
    path: path.resolve(__dirname, 'build'),
    filename: 'index.js' // bundled output file
  },
  resolve: {
    extensions: ['.ts', '.js'],
  },
  module: {
    rules: [
      {
        test: /\.ts$/, // load files with .ts extension with ts-loader
        use: [
          'ts-loader',
        ]
      }
    ]
  }
}
```

Configure Typescript

```javascript
// tsconfig.json

{
  "compilerOptions": {
    "sourceMap": true
  }
}
```

Configure ESlint. Always good

```shell
$ yarn add --dev babel-eslint
```

```javascript
// .eslintrc.json

{
    "extends": "eslint:recommended",
    "parser": "babel-eslint",
    "env": {
        "jest": true,
        "es6": true,
        "node": true
    }
}
```

## "Hello world" Express app

```shell
$ yarn add express @types/express # install express and express types
```

```javascript
// src/app.ts

import * as express from 'express';
import { Request, Response } from 'express';

const app = express();

app.get('/ping', (req: Request, res: Response) => {
  res.json({
    ping: 'pong',
  });
});

export default app
```

```javascript
// src/index.ts

import app from './app'

const {
  PORT = 3000,
} = process.env;

app.listen(PORT, () => {
  console.log('server started at http://localhost:'+PORT);
});
```

## Reducing bundle size

This simple "Hello World" project, when bundle up takes 1MB of space. That is a lot for such a simple application.

For Backend application it's not necessary to have all modules bundle up in a single file. To remove those modules from the bundle use `webpack-node-externals`

> When bundling with Webpack for the backend - you usually don't want to bundle its node_modules dependencies. This library creates an externals function that ignores node_modules when bundling in Webpack.
> [NPM: webpack-node-externals](https://www.npmjs.com/package/webpack-node-externals)

```shell
$ yarn add webpack-node-externals --dev
```

Add to `webpack.config.js`

```javascript
// webpack.config.js

const nodeExternals = require('webpack-node-externals');
module.exports = {
  ...
  externals: [ nodeExternals() ]
}
```

After setting up `webpack-node-externals` the bundle size dropped to 2.7KB :)

## Setting hot-reload

```javascript
// webpack.config.js

module.exports = {
  ...
  watch: NODE_ENV === 'development',
}
```

```javascript
// package.json

{
  ...
  "scripts": {
    "start": "NODE_ENV=development webpack"
  }
}
```

## Set up Jest testing framework

```shell
$ yarn add --dev jest ts-jest @types/jest
$ yarn ts-jest config:init
```

## Testing Express endpoint - Integration Testing

To test Express' endpoints without having the server running, use [`supertest`](https://github.com/visionmedia/supertest#readme)

```shell
$ yarn add --dev supertest
```

Testing `pong` response

```javascript
// src/app.test.js

const request = require('supertest')
const app = require('./app').default

test('test ping === pong', async done => {
  request(app)
    .get('/ping')
    .expect(200)
    .then(res => {
      expect(res.body.ping).toBe('pong')
      done()
    })
    .catch(err => {
      console.error(err)
      done()
    })
});
```

To see the test passing `yarn test` :)

## Setting up Github CI

Github now allows to have [`self-hosted`](https://docs.github.com/en/actions/hosting-your-own-runners/adding-self-hosted-runners) runners for it actions. That means you can use your own machine as the engine running your github actions, you don't need to pay a third-party service to have you CI script ready to production.

To set your machine as one of your `runners` go to your project *Settings* > (on the sidebar) *Actions* > (on the sidebar) *Runners* > *Add new Runner*. Follow the instructions and you should be good to go.

## [Github Actions](https://docs.github.com/en/actions/learn-github-actions/introduction-to-github-actions)

> Use [Act](https://github.com/nektos/act) to debug github actions locally.

This project has 1 workflow with 2 jobs: `lint`, and `test`. This workflow is set to be executed on every `git push`.

```yaml
# ./.github/workflows/ci.yml

name: Node.js CI

on:
  push # Execute this workflow on every push

jobs:
  lint:
    runs-on: self-hosted

    steps:
      - uses: actions/checkout@v2
      - name: Use Node.js 15.x
        uses: actions/setup-node@v2
        with:
          node-version: 15.x
      - run: yarn install
      - run: yarn run eslint > eslint-output
      - name: Save eslint-output
        uses: actions/upload-artifact@v2
        with:
          name: eslint-result
          path: ./eslint-output

```

The first line of the `lint` job is to set where this job will be executed. For this project it will be using a `self-hosted runner`.

Then comes the `steps`. The `steps` are a yaml object that frequently contains the `uses`, `name`, `run`, `with`. `uses` uses(duh) a github action, `name` describes the step, `run` executes a shell command and so on. Learn more at [Quickstart for github actions](https://docs.github.com/en/actions/quickstart)

`actions` are custom commands that can encapsulate a bunch of behavior. The `lint` workflow is using a few of those: `actions/checkout@v2`, `actions/setup-node@v2`, `actions/upload-artifact@v2`

So, step by step:

- `- uses: actions/checkout@v2`: Checkout changes from push event;
- `- name: Use Node.js 15.x`: Setup job use Node.js 15.x;
- `- run: yarn install`: Install dependencies with yarn;
- `- run: yarn run eslint > eslint-output`: Run yarn's `lint` script and saves the output in `eslint-output`
- `- name: Save eslint-output`: Upload `eslint-output` to github as artifact. For later use, is always good :)


```yaml
# ./.github/workflows/ci.yml

name: Node.js CI

on:
  push # Execute this workflow on every push

jobs:
  test:
    needs: lint
    runs-on: self-hosted

    steps:
      - uses: actions/checkout@v2
      - name: Use Node.js 15.x
        uses: actions/setup-node@v2
        with:
          node-version: 15.x
      - run: yarn install
      - run: yarn build
      - run: yarn test > jest-output
      - name: Save coverage
        uses: actions/upload-artifact@v2
        with:
          name: coverage
          path: ./coverage
      - name: Save jest-result
        uses: actions/upload-artifact@v2
        with:
          name: jest-result
          path: ./jest-output
```

The `test` job states by declaring a dependency with the `lint` job. That means, for this workflow first the `lint` job will be executed then, if `lint` is successful, it will execute the `test` job.

The `steps` have similar boilerplate from `lint`, but where they differ the steps are:

- `- run: yarn build`: Build application;
- `- run: yarn test > jest-output`: Run yarn's test script and save its output to `jest-output`;
- `- name: Save coverage`: Upload `coverage` folder to github as artifact for later use;
- `- name: Save jest-result`: Upload `jest-result` folder to github as artifact for later use;

And that is it.

The source code of the project can be found @ [https://github.com/arthurstomp/ts_express_template](https://github.com/arthurstomp/ts_express_template)

Until next time.
