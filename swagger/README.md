## Swagger for REST APIs

### Continuous deployment

The specs are continuously deployed and available in Swagger UI:

- [couchbase-lite](http://developer.couchbase.com/mobile/swagger/couchbase-lite/)
- [sync-gateway-admin](http://developer.couchbase.com/mobile/swagger/sync-gateway-admin/)
- [sync-gateway-public](http://developer.couchbase.com/mobile/swagger/sync-gateway-public/)

A Jenkins job called SwaggerContinuousDelivery on http://docs.couchbasemobile.com:8080 is notified when a new commit is detected on the master branch. The job resolves the yaml files into one and outputs the json specs using the `resolve.js` script. This Jenkins server is for internal use.

### Working locally on swagger specs

The Swagger specs are located in the `swagger` folder in YAML format.

1. clone the repository: `git clone https://github.com/couchbaselabs/couchbase-mobile-portal.git`
2. cd in this directory: `cd swagger`
3. Install gulp: `npm install -g gulp`
4. Install local dependencies: `npm install`
5. Start the resolver and validator script: `gulp watch`

There are two gulp tasks available.

- build: only builds and outputs to `./tmp`
- watch: build and keep monitoring for changes in `.yaml` files

In the `parameters` and `definitions` folders you will find 3 files.

- `common.yaml`: Functionality that's common to CBL and SG.
- `cbl.yaml`: Functionality specific to CBL.
- `sg.yaml`: Functionality specific to SG.

When running `gulp watch` or `gulp build`, the definitions and parameters are pulled from the correct files depending on the target (lite, public or admin).

### Swagger libraries

This section describes how to generate the Swagger specs locally to use them with code gen tools whether it be with SwaggerJS or Swagger Code Gen.

- Install gulp: `npm install -g gulp`
- Install local dependencies: `npm install`
- Start the resolver and validator script: `gulp watch`
- Start a web server: `python -m SimpleHTTPServer 9000`

#### SwaggerJS

- Follow the instructions on [swagger-js](https://github.com/swagger-api/swagger-js) to use the specs served locally (e.g for the Sync Gateway Public REST API it's `http://localhost:9000/tmp/public.json`).

#### Swagger Code Gen

- Follow the instructions on [swagger-codegen](https://github.com/swagger-api/swagger-codegen) to install it.
- Generate the client by specifying the URL to the swagger spec over localhost. For the SG Public REST API for example.

    ```bash
    java -jar modules/swagger-codegen-cli/target/swagger-codegen-cli.jar generate \
          -i http://localhost:9000/tmp/public.json -l java -o ./lib
    ```

### Custom HTML output

Not currently supported, but there is an option to use the open source bootprint-openapi repo (submodule in `swagger`) to generate HTML that can be pushed to IngestStage.
