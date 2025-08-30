## Palantir Unit Tests Guide

Here is the short version of how Foundry teams unit test Python transforms, with the Stack Overflow example you shared as the concrete pattern.

The common pattern in Foundry
	1.	Put tests in transforms-python/src/test/test_*.py, run with pytest. Foundry surfaces results in the Checks tab.  ￼
	2.	Build a tiny Pipeline in your test, add your transform, and run it with a lightweight runner. The easiest is an in-memory datastore so you do not touch real datasets.  ￼
	3.	Provide synthetic inputs, execute the transform, and assert on the output DataFrame. You can either:

	•	use TransformRunner with InMemoryDatastore (integration-like, very close to how Foundry runs), or
	•	call your compute directly and mock the inputs and outputs (pure unit style).  ￼ ￼

Option A: TransformRunner + in-memory inputs

This is exactly what the accepted Stack Overflow answer shows. Key ideas:
	•	Create input Spark DataFrames in the test.
	•	Store them into an InMemoryDatastore under the exact aliases of your Input objects.
	•	Run TransformRunner.build_dataset(...) on your Output alias and compare results.  ￼

Minimal template you can drop in src/test/test_simple_transform.py:

```python
from transforms.api import Pipeline
from transforms.verbs.testing.TransformRunner import TransformRunner
from transforms.verbs.testing.datastores import InMemoryDatastore
from myproject.datasets.simple_transform import compute  # your @transform_df


def test_simple_transform(spark_session):
    races = spark_session.createDataFrame(
        [(1, 2013, 10, "A"), (2, 2012, 11, "B")],
        ["raceId", "year", "circuitId", "name"]
    )
    circuits = spark_session.createDataFrame(
        [(10, "X", "Loc1"), (11, "Y", "Loc2")],
        ["circuitId", "name", "location"]
    )

    # Create a transform pipeline using the transform function used in the actual transform
    pipeline = Pipeline()
    pipeline.add_transforms(compute)
    # Create an in memory datastore to store the toy datasets
    store = InMemoryDatastore()

    t = pipeline.transforms[0]
    # Add the toy datasets into the a dictionary
    inputs_by_name = {"races": races, "circuits": circuits}
    # Store the datasets into the in memory datastore
    for name, inp in t.inputs.items():
        store.store_dataframe(inp.alias, inputs_by_name[name])

    # Pass the in memory datastore and the transform pipeline to the TransformRunner
    runner = TransformRunner(pipeline, datastore=store)
    out_alias = list(t.outputs.values())[0].alias
    df_out = runner.build_dataset(spark_session, out_alias)

    # Create an expected output dataframe for comparison
    expected = spark_session.createDataFrame(
        [(1, "A", "X", "Loc1", 2013),
         (2, "B", "Y", "Loc2", 2012)],
        ["raceId", "race_name", "circuit_name", "location", "year"]
    )
    assert df_out.exceptAll(expected).count() == 0
    assert expected.exceptAll(df_out).count() == 0
    assert df_out.schema == expected.schema
```

This mirrors the SO approach and avoids hard coded paths by reading inp.alias and the output alias from the pipeline.  ￼

Option B: call compute directly with mocks

Palantir’s docs show a pure unit style where you:
	•	create small CSV or in-memory frames,
	•	wrap or mock the Input and Output,
	•	call compute(...),
	•	capture the write_dataframe call to assert on the DataFrame content.  ￼

This keeps the test fast and isolated. It also works for @transform functions that write to an Output instead of returning a DataFrame.  ￼

Where these pieces come from
	•	The exact TransformRunner + InMemoryDatastore recipe and alias mapping are in the accepted Stack Overflow answer.  ￼
	•	Palantir’s official docs show CSV-fixture tests, mocking inputs and outputs, pytest discovery rules, and how results appear in Checks. They also show Spark-specific assertions and enabling testing plugins.  ￼


My Learnings:

To create a unit test for a transform operation in Palantir Foundry, I will have to create toy datasets and perform the exact operations which are being performed in the transform pipeline such that the output schema of the test should match the output schema for the pipeline. 