# Reproduction for an OutOfMemoryError in Apache Solr when trying to index multi-valued RPT fields

## Problem statement

- I'm using Drupal 10 with the following modules:
  - `search_api` `8.x-1.34`
  - `search_api_solr` `4.3.3`
- The `search_api_solr` module generates the schema for the Solr core, which is copied into the `conf` directory
  after creating the core.
- I need a custom dynamic field and field type, so I append some lines to the following files.
  - My custom type to `schema_extra_types.xml`:
    ```
    <!-- custom fields -->
    <fieldType name="custom" class="solr.SpatialRecursivePrefixTreeFieldType" geo="false" distanceUnits="kilometers" maxDistErr="1" worldBounds="ENVELOPE(0,48000000,1,0)" />
    ```
  - My custom dynamic fields to `schema_extra_fields.xml` (in keeping with the
    Drupal module's convention of using `<type>m_*` for multi-value fields and `<type>s_*` for single-value fields):
    ```
    <!-- custom fields -->
    <dynamicField name="customs_*" type="custom" indexed="true" stored="true" multiValued="false" />
    <dynamicField name="customm_*" type="custom" indexed="true" stored="true" multiValued="true" />
    ```
- When I index two values to a multi-value field, it crashes the Solr server with an `OutOfMemoryError`:
  ```json
  {
    "add": {
        "doc": {
            "id": "test-multi",
            "customm_test": [
                "ENVELOPE(29054482, 29054482, 1, 0)",
                "ENVELOPE(27609016, 27609016, 1, 0)"
            ]
        }
    }
  }
  ```
- When I index a single value to a multi-value field, it works in my dev environment, but it also crashes with an
  `OutOfMemoryError` when following the [steps below](#steps-to-reproduce). I have not figured out the difference
  between the two, yet.
  ```json
  {
    "add": {
        "doc": {
            "id": "test-single",
            "customm_test": "ENVELOPE(3678644, 3686246, 1, 0)"
        }
    }
  }
  ```

## Steps to reproduce

Assumptions:

- You are using a Linux machine.
- You have installed Docker.

### Using Docker image `solr:8`

1. In a terminal, start the server, create the core, and see the logs:
   ```shell
   mkdir -p ./tmp/solr-8
   sudo chown -R 8983:8983 ./tmp/solr-8
   docker run --rm -p 8983:8983 -v ./tmp/solr-8:/var/solr solr:8 solr-precreate vv
   ```
2. In another terminal, update the core config:
   ```shell
   sudo rm -rf ./tmp/solr-8/data/vv/conf
   sudo cp -r ./solr-8-conf ./tmp/solr-8/data/vv/conf
   sudo chown -R 8983:8983 ./tmp/solr-8/data/vv/conf
   ```
3. In a browser, open the "Core Admin" page at http://localhost:8983/solr/#/~cores/vv
4. Click "Reload".
5. Try to index a single value to a multi-value field. It crashes the solr container with an `OutOfMemoryError`.
   ```shell
   curl -X POST -H "Content-Type: application/json" --data-binary @./test-single.json 'http://localhost:8983/solr/vv/update?omitHeader=false&wt=json&json.nl=flat&commitWithin=1000'
   ```
6. Try to index two values to a multi-value field. It crashes the solr container with an `OutOfMemoryError`.
   ```shell
   curl -X POST -H "Content-Type: application/json" --data-binary @./test-multiple.json 'http://localhost:8983/solr/vv/update?omitHeader=false&wt=json&json.nl=flat&commitWithin=1000'
   ```

The same happens when using the `solr:8-slim` image.

### Using Docker image `solr:9`

1. In a terminal, start the server, create the core, and see the logs:
   ```shell
   mkdir -p ./tmp/solr-9
   sudo chown -R 8983:8983 ./tmp/solr-9
   docker run --rm -p 8983:8983 -v ./tmp/solr-9:/var/solr solr:9 solr-precreate vv
   ```
2. In another terminal, update the core config:
   ```shell
   sudo rm -rf ./tmp/solr-9/data/vv/conf
   sudo cp -r ./solr-9-conf ./tmp/solr-9/data/vv/conf
   sudo chown -R 8983:8983 ./tmp/solr-9/data/vv/conf
   ```
3. Same as steps 3–6 above.

The same happens when using the `solr:9-slim` image.

### Clean up
   ```shell
   sudo rm -rf ./tmp
   ```
