# Apache-Beam-to-Datastore
This is an example of how to transform a dictionary to a google cloud storage entity to afterwards store it into google cloud datastore using Dataflow/apache beam in python.

Our example will transform a bigquery dataset to gcs entity to datastore:

> **Step 1:** Load in the bigquery dataset with a query

```Python
with beam.Pipeline(options=pipeline_options) as pipeline:
    _ = (pipeline
        | 'BigQuery source' >> ReadFromBigQuery(  # nosec
                        query=('SELECT id, text FROM `bigquery-public-data.stackoverflow.comments`' \
                                'WHERE text LIKE "%Golang%" LIMIT 1000')
        )
```

> 
> **Step 2:** We want to change the dictionary in our dataflow pipeline to the correct format for datastore. We need to transform a dictionary to a GCS entity. This can be done by adding the following code to your pipeline and Calling it within a beam.ParDo(CreateEntities(args.project))
>
```Python
class CreateEntities(beam.DoFn):
    """Create GCS entity from dict"""

    def __init__(self, project_id) -> None:
        super().__init__()
        self.project_id = project_id

    def to_runner_api_parameter(self, placeholder):
        pass

    def process(self, element: Any, *args, **kwargs) -> Any:
        """
        Process the input element and yield a GCS entity.

        Args:
            element (Any): The input element containing the comment id and text.

        Yields:
            Any: A GCS entity with the comment id and text as properties.
        """
        comment_id, comment_text = element["id"], element["text"]
        key = Key(["id", comment_id], project=self.project_id)
        entity = Entity(key)
        entity.set_properties({"text": comment_text})

        yield entity
```

> Step 3: You will add the previously mentioned transformation to the dataflow pipeline and finally store the data to datastore. This can be done by importing the right function below:

```Python
from apache_beam.io.gcp.datastore.v1new.datastoreio import WriteToDatastore
```
> And calling the WriteToDatastore with the project args.

```Python
with beam.Pipeline(options=pipeline_options) as pipeline:
        _ = (pipeline
             | 'BigQuery source' >> ReadFromBigQuery(  # nosec
                 query=('SELECT id, text FROM `bigquery-public-data.stackoverflow.comments`' \
                        'WHERE text LIKE "%Golang%" LIMIT 1000'),
                 use_standard_sql=True)
             | 'create entity' >> beam.ParDo(CreateEntities(args.project))
             | 'write to datastore' >> WriteToDatastore(args.project)
             )
```
