# Metadata-Driven Ingestion with Azure Data Factory

## Overview

The insurance claims data platform uses a metadata-driven ingestion approach in Azure Data Factory (ADF).

Instead of manually running a separate pipeline for every dataset, a master pipeline uses metadata and a ForEach activity to automatically process multiple datasets.

Currently, the framework processes:

- Customers
- Hospitals

The same design can later be extended to Claims, Policies, Payments, Providers, and other insurance datasets.

---

## Architecture

The ingestion flow is:

    Ingestion Metadata
            |
            v
    pl_master_ingestion
            |
            v
         ForEach
            |
            v
     Execute Pipeline
            |
            v
      pl_ingest_file
            |
            v
       Copy Activity
            |
            v
    Landing --> Bronze

The master pipeline determines which datasets need to be processed, while the generic ingestion pipeline handles the actual data movement.

---

## ADLS Structure

The current storage structure is:

    insurance/
    |
    |-- landing/
    |   |-- customers/
    |   |   `-- customers.csv
    |   |
    |   `-- hospitals/
    |       `-- hospitals.csv
    |
    `-- bronze/
        |-- customers/
        |   `-- customers.csv
        |
        `-- hospitals/
            `-- hospitals.csv

The Landing layer represents files delivered by source systems.

The Bronze layer contains the raw data successfully ingested by the data platform.

---

## Generic Ingestion Pipeline

Pipeline:

    pl_ingest_file

The purpose of this pipeline is to ingest one file from the Landing layer into the Bronze layer.

It accepts two parameters:

    directory_name
    file_name

Example for Customers:

    directory_name = customers
    file_name = customers.csv

This resolves to:

    Source:
    insurance/landing/customers/customers.csv

    Destination:
    insurance/bronze/customers/customers.csv

Example for Hospitals:

    directory_name = hospitals
    file_name = hospitals.csv

This resolves to:

    Source:
    insurance/landing/hospitals/hospitals.csv

    Destination:
    insurance/bronze/hospitals/hospitals.csv

---

## Generic Datasets

Two reusable datasets are used by the ingestion pipeline.

### Source Dataset

    ds_landing_csv

The source path is dynamically constructed using dataset parameters.

Directory:

    @concat('landing/', dataset().directory_name)

File name:

    @dataset().file_name

This allows the same dataset to represent different Landing files.

---

### Bronze Dataset

    ds_bronze_csv

The destination path is also dynamically constructed.

Directory:

    @concat('bronze/', dataset().directory_name)

File name:

    @dataset().file_name

This allows the same dataset to write different files into the Bronze layer.

---

## Master Ingestion Pipeline

Pipeline:

    pl_master_ingestion

The purpose of this pipeline is to orchestrate ingestion for multiple datasets.

Instead of manually executing `pl_ingest_file` for every dataset, the master pipeline loops through ingestion metadata and invokes the generic ingestion pipeline.

---

## Ingestion Metadata

The master pipeline contains an Array parameter:

    ingestion_metadata

Current metadata:

    [
        {
            "directory_name": "customers",
            "file_name": "customers.csv"
        },
        {
            "directory_name": "hospitals",
            "file_name": "hospitals.csv"
        }
    ]

Each object represents one dataset that needs to be ingested.

The metadata describes:

- Source directory
- Source file name

It does not contain the actual Customer or Hospital records.

---

## ForEach Activity

The master pipeline contains a ForEach activity.

The ForEach Items expression is:

    @pipeline().parameters.ingestion_metadata

The ForEach activity loops through every object in the metadata array.

For the current configuration, it performs two iterations:

    Iteration 1 -> Customers
    Iteration 2 -> Hospitals

Sequential execution is currently used to make pipeline execution and troubleshooting easier to understand.

---

## item() Expression

Inside the ForEach activity, `item()` represents the metadata object currently being processed.

For the Customers iteration:

    @item().directory_name

returns:

    customers

and:

    @item().file_name

returns:

    customers.csv

For the Hospitals iteration, the same expressions return:

    hospitals
    hospitals.csv

This allows the same pipeline logic to work for different datasets.

---

## Execute Pipeline Activity

Inside the ForEach activity, an Execute Pipeline activity invokes:

    pl_ingest_file

The parameters are passed dynamically.

Directory:

    @item().directory_name

File name:

    @item().file_name

Therefore, the execution becomes:

    pl_master_ingestion
            |
            v
         ForEach
            |
            |-- Customers
            |       |
            |       v
            |  pl_ingest_file
            |
            `-- Hospitals
                    |
                    v
               pl_ingest_file

The ingestion logic is reused instead of duplicated.

---

## Why Metadata-Driven Ingestion?

Without this design, separate pipelines might be created for every dataset:

    pl_ingest_customers
    pl_ingest_hospitals
    pl_ingest_claims
    pl_ingest_policies
    pl_ingest_payments

As the number of datasets increases, this becomes difficult to maintain.

With the metadata-driven approach:

    Metadata
        |
        v
    Master Pipeline
        |
        v
      ForEach
        |
        v
    Generic Pipeline

the same ingestion logic can be reused for many datasets.

Benefits include:

- Reusability
- Scalability
- Reduced code duplication
- Easier maintenance
- Consistent ingestion logic
- Easier onboarding of new datasets

---

## Current Execution Flow

When `pl_master_ingestion` is executed:

1. ADF reads the `ingestion_metadata` array.
2. The ForEach activity selects the first metadata object.
3. `item()` provides the current directory and file name.
4. Execute Pipeline invokes `pl_ingest_file`.
5. The generic pipeline copies the file from Landing to Bronze.
6. ForEach moves to the next metadata object.
7. The process repeats until all metadata entries are processed.

For the current implementation:

    customers.csv
         |
         v
    Landing
         |
         v
    pl_ingest_file
         |
         v
    Bronze

followed by:

    hospitals.csv
         |
         v
    Landing
         |
         v
    pl_ingest_file
         |
         v
    Bronze

---

## Testing

The master pipeline was tested using ADF Debug mode.

A single execution of:

    pl_master_ingestion

processes both:

    customers.csv
    hospitals.csv

This verifies that multiple datasets can be ingested without manually executing the worker pipeline separately for each dataset.

A failure scenario was also tested using an invalid file name to understand pipeline failure behaviour and troubleshooting.

---

## Current Limitations

The ingestion metadata is currently stored directly inside the ADF pipeline as an Array parameter.

This is suitable for learning and small numbers of datasets but is not ideal for a large production platform.

For example, maintaining metadata for 100 datasets directly inside the pipeline would become difficult.

---

## Future Improvements

The ingestion framework can be extended with:

- External metadata/configuration storage
- Lookup activity
- Parallel ingestion
- Retry policies
- Error handling
- Audit logging
- Pipeline monitoring
- Data quality validation
- Incremental ingestion
- Dynamic partitioning
- Failure notifications
- Rerun support
- CI/CD deployment

The long-term architecture will move toward:

    External Metadata Store
             |
             v
          Lookup
             |
             v
    pl_master_ingestion
             |
             v
          ForEach
             |
             v
       pl_ingest_file
             |
             v
    Landing --> Bronze

This allows new datasets to be onboarded mainly through configuration rather than creating new ingestion pipelines.