# bcgov/gcpe-lup-api

Minimal API for the GCPE LUP [Public](https://github.com/bcgov/gcpe-lup-public) and [Admin](https://github.com/bcgov/gcpe-lup-admin) apps -->

## How to run this

Before running the api, you must set some environment variables:
1) MINIO_HOST='foo.pathfinder.gov.bc.ca'
2) MINIO_ACCESS_KEY='xxxx'
3) MINIO_SECRET_KEY='xxxx'
4) KEYCLOAK_ENABLED=true
5) MONGODB_DATABASE='landuseplanning'

One way to do this is to edit your ~/.bash_profile file to contain:

```
export MONGODB_DATABASE="landuseplanning"
export MINIO_HOST="foo.pathfinder.gov.bc.ca"
export MINIO_ACCESS_KEY="xxxx"
export MINIO_SECRET_KEY="xxxx"
export KEYCLOAK_ENABLED=true
```

Please note that these values are case sensitive so don't use upper-case TRUE for example.

Don't forget to reload your .bash_profile file so that your terminal environment is up to date with the correct values
```
source ~/.bash_profile
env
```

The above env command will show you your environment variables and allow you to check that the correct values are present.

Start the server by running `npm start`

# Prerequisites

| Technology | Version | Website                                     | Description                               |
|------------|---------|---------------------------------------------|-------------------------------------------|
| node       | 8.x.x   | https://nodejs.org/en/                      | JavaScript Runtime                        |
| npm        | 6.x.x   | https://www.npmjs.com/                      | Node Package Manager                      |
| yarn       | latest  | https://yarnpkg.com/en/                     | Package Manager (more efficient than npm) |
| mongodb    | 3.2     | https://docs.mongodb.com/v3.2/installation/ | NoSQL database                            |

## Install [Node + NPM](https://nodejs.org/en/)

_Note: Windows users can use [NVM Windows](https://github.com/coreybutler/nvm-windows) to install and manage multiple versions of Node+Npm._

## Install [Yarn](https://yarnpkg.com/lang/en/docs/install/#alternatives-tab)

```
npm install -g yarn
```

## Install [MongoDB](https://docs.mongodb.com/v3.2/installation/)

# Build and Run

1. Download dependencies
```
yarn install
```
2. Run the app
```
npm start
```
3. Go to http://localhost:3000/api/docs to verify that the application is running.

    _Note: To change the default port edit `swagger.yaml`._

4. POST `http://localhost:3000/api/login/token` with the following body
```
{
"username": #{username},
"password": #{password}
}

# API Specification

The API is defined in `swagger.yaml`.

If the this gcpe-lup-api is running locally, you can view the api docs at: `http://localhost:3000/api/docs/`

This project uses npm package `swagger-tools` via `./app.js` to automatically generate the express server and its routes.

Recommend reviewing the [Open API Specification](https://swagger.io/docs/specification/about/) before making any changes to the `swagger.yaml` file.

# Initial Setup

We use a version manager so as to allow concurrent versions of node and other software.  [asdf](https://github.com/asdf-vm/asdf) is recommended.  asdf uses a config file called .tool-versions that the reshim command picks up so that all collaborators are using the same versions.

Run the following commands:
```
asdf plugin-add nodejs https://github.com/asdf-vm/asdf-nodejs.git
```

Next open .tool-versions and take the node version there and use it in the following command so that that specific version of node is present on your local machine for asdf to switch to.  Replace the "x" characters in the following command with what's in .tool-versions.

```
asdf install nodejs x.xx.x
```

Then run the following commands every time you need to switch npm versions in a project.

```
asdf reshim nodejs
npm i -g yarn
yarn install
```


Acquire a dump of the database from one of the live environments.  

Make sure you don't have an old copy (careful, this is destructive):

```
mongo
use landuseplanning
db.dropDatabase()
```

Restore the database dump you have from the old epic system.

```
mongorestore -d epic dump/[old_database_name_most_likely_esm]
```

Then run the contents of [dataload](prod-load-db/esm_prod_april_1/dataload.sh) against that database.  You may need to edit the commands slightly to match your db name or to remove the ".gz --gzip" portion if your dump unpacks as straight ".bson" files.

Seed local database as described in [seed README](seed/README.md)

Start server by running `npm start` in root

## Developing

See [Code Reuse Strategy](code_reuse_strategy.md)

# Testing

This project is using [jest](http://jestjs.io/) as a testing framework. You can run tests with
`yarn test` or `jest`. Running either command with the `--watch` flag will re-run the tests every time a file is changed.

To run the tests in one file, simply pass the path of the file name e.g. `jest api/test/search.test.js --watch`. To run only one test in that file, chain the `.only` command e.g. `test.only("Search returns results", () => {})`.

The **_MOST IMPORTANT_** thing to know about this project's test environment is the router setup. At the time of writing this, it wasn't possible to get [swagger-tools](https://github.com/apigee-127/swagger-tools) router working in the test environment. As a result, all tests **_COMPLETELY bypass_ the real life swagger-tools router**. Instead, a middleware router called [supertest](https://github.com/visionmedia/supertest) is used to map routes to controller actions. In each controller test, you will need to add code like the following:

```javascript
const test_helper = require('./test_helper');
const app = test_helper.app;
const featureController = require('../controllers/feature.js');
const fieldNames = ['tags', 'properties', 'applicationID'];

app.get('/api/feature/:id', function(req, res) {
  let params = test_helper.buildParams({'featureId': req.params.id});
  let paramsWithFeatureId = test_helper.createPublicSwaggerParams(fieldNames, params);
  return featureController.protectedGet(paramsWithFeatureId, res);
});

test("GET /api/feature/:id  returns 200", done => {
  request(app)
    .get('/api/feature/AAABBB')
    .expect(200)
    .then(done)
});
```

This code will stand in for the swagger-tools router, and help build the objects that swagger-tools magically generates when HTTP calls go through it's router. The above code will send an object like below to the `api/controllers/feature.js` controller `protectedGet` function as the first parameter (typically called `args`).

```javascript
{
  swagger: {
    params: {
      auth_payload: {
        scopes: ['sysadmin', 'public'],
        userID: null
      },
      fields: {
        value: ['tags', 'properties', 'applicationID']
      },
      featureId: {
        value: 'AAABBB'
      }
    }
  }
}
```

Unfortunately, this results in a lot of boilerplate code in each of the controller tests. There are some helpers to reduce the amount you need to write, but you will still need to check the parameter field names sent by your middleware router match what the controller(and swagger router) expect. However, this method results in  pretty effective integration tests as they exercise the controller code and save objects in the database.


## Test Database
The tests run on an in-memory MongoDB server, using the [mongodb-memory-server](https://github.com/nodkz/mongodb-memory-server) package. The setup can be viewed at [test_helper.js](api/test/test_helper.js), and additional config in [config/mongoose_options.js]. It is currently configured to wipe out the database after each test run to prevent database pollution.

[Factory-Girl](https://github.com/aexmachina/factory-girl) is used to easily create models(persisted to db) for testing purposes.

## Mocking http requests
External http calls (such as GETs to BCGW) are mocked with a tool called [nock](https://github.com/nock/nock). Currently sample JSON responses are stored in the [test/fixtures](test/fixtures) directory. This allows you to intercept a call to an external service such as bcgw, and respond with your own sample data.

```javascript
  const bcgwDomain = 'https://openmaps.gov.bc.ca';
  const searchPath = '/geo/pub/FOOO';
  const crownlandsResponse = require('./fixtures/crownlands_response.json');
  var bcgw = nock(bcgwDomain);
  let dispositionId = 666666;

  beforeEach(() => {
    bcgw.get(searchPath + urlEncodedDispositionId)
      .reply(200, crownlandsResponse);
  });

  test('returns the features data from bcgw', done => {
    request(app).get('/api/public/search/bcgw/dispositionTransactionId/' + dispositionId)
      .expect(200)
      .then(response => {
        let firstFeature = response.body.features[0];
        expect(firstFeature).toHaveProperty('properties');
        expect(firstFeature.properties).toHaveProperty('DISPOSITION_TRANSACTION_SID');
        done();
      });
  });
```

## Configuring Environment Variables

Recall the environment variables we need for local dev:
1) MINIO_HOST='foo.pathfinder.gov.bc.ca'
2) MINIO_ACCESS_KEY='xxxx'
3) MINIO_SECRET_KEY='xxxx'
4) KEYCLOAK_ENABLED=true
5) MONGODB_DATABASE='epic'

To get actual values for the above fields in the deployed environments, examine the openshift environment you wish to target:

```
oc project [projectname]
oc get routes | grep 'minio'
oc get secrets | grep 'minio'
```

You will not be able to see the above value of the secret if you try examine it.  You will only see the encrypted values.  Approach your team member with admin access in the openshift project in order to get the access key and secret key values for the secret name you got from the above command.  Make sure to ask for the correct environment (dev, test, prod) for the appropriate values.
