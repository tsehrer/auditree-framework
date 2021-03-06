.. -*- mode:rst; coding:utf-8 -*-

.. _evidence-partitioning:

Partitioned Evidence
====================

Depending on the repository hosting service being used, it may be necessary to
manage evidence more effectively.  For example Github has limits on individual
file sizes.  In cases like this it is possible to partition evidence into
smaller, more manageable chunks if that evidence satisfies the following
conditions:

* Evidence is of the type :py:class:`~compliance.evidence.RawEvidence` or a
  sub-class of :py:class:`~compliance.evidence.RawEvidence`.

* The evidence content is JSON and the evidence file has a ``.json`` file
  extension.

* A partition key exists, that can logically separate the evidence into parts.

  * A key can be a single field or multiple fields in the evidence JSON
    document structure.  If multiple fields are used, the partitioning of the
    evidence will be handled in the order of the fields provided.

  * The path to a key can be represented using ``dot`` notation if necessary.

* A single partition root exists where partitioning can be applied.

  * The partition root must be a list field that serves as the origin of
    evidence partitioning based on the partition key provided.

  * The partition root can be the entire evidence JSON document or a nested
    field with the evidence JSON document.

Partition By Configuration
--------------------------

Turning partitioning on for a given evidence can be accomplished by setting the
appropriate ``locker.partitions`` parameters in your configuration file.
Parameters include the relative repository path to the raw evidence (sans the
*raw*), the partition key(s) as a list, and optionally the partition root
where the evidence will be partitioned.  For example::

  {
    "locker": {
      "repo_url": "https://github.com/my-org/my-evidence-repo",
      "partitions": {
        "foo/evidence_bar.json": {
          "fields": ["location.country", "location.region"],
          "root": "path.to.list"
        }
      }
    }
  }

or::

  {
    "locker": {
      "repo_url": "https://github.com/my-org/my-evidence-repo",
      "partitions": {
        "foo/evidence_bar.json": {
          "fields": ["location.country", "location.region"]
        }
      }
    }
  }

Partition By Evidence
---------------------

Turning partitioning on for a given evidence can also be accomplished by
directly constructing your raw evidence object in your fetcher with the
appropriate partitioning parameters.  Parameters include the partition key(s)
as a list, and optionally the partition root where the evidence will be
partitioned.  For example::

  RawEvidence(
      'evidence_bar.json',
      'foo',
      DAY,
      'My partitioned evidence',
      partition={
          'fields': ['location.country', 'location.region'],
          'root': 'path.to.list'
      }
  )

or::

  RawEvidence(
      'evidence_bar.json',
      'foo',
      DAY,
      'My partitioned evidence',
      partition={'fields': ['location.country', 'location.region']}
  )

**NOTE:** Constructing a partitioned :py:class:`~compliance.evidence.RawEvidence`
object overrides partitioning by configuration.

Examples
--------

Default Partition Root
~~~~~~~~~~~~~~~~~~~~~~

* Unpartitioned evidence content::

    [
      {"foo": "...", "bar": "...", "country": "US", "region": "IL"},
      {"foo": "...", "bar": "...", "country": "US", "region": "IL"},
      {"foo": "...", "bar": "...", "country": "US", "region": "NY"},
      {"foo": "...", "bar": "...", "country": "UK", "region": "Essex"},
      {"foo": "...", "bar": "...", "country": "UK", "region": "Essex"}
    ]

* Partitioned configuration::

    {
      "locker": {
        "repo_url": "https://github.com/my-org/my-evidence-repo",
        "partitions": {
          "foo/evidence_bar.json": {
            "fields": ["country", "region"]
          }
        }
      }
    }

* Partitions yielded

  * US/IL partition - ``foo/<us_il_hash>_evidence_bar.json``::

      [
        {"foo": "...", "bar": "...", "country": "US", "region": "IL"},
        {"foo": "...", "bar": "...", "country": "US", "region": "IL"}
      ]

  * US/NY partition - ``foo/<us_ny_hash>_evidence_bar.json``::

      [
        {"foo": "...", "bar": "...", "country": "US", "region": "NY"}
      ]

  * UK/Essex partition - ``foo/<uk_essex_hash>_evidence_bar.json``::

      [
        {"foo": "...", "bar": "...", "country": "UK", "region": "Essex"},
        {"foo": "...", "bar": "...", "country": "UK", "region": "Essex"}
      ]

Explicit Partition Root
~~~~~~~~~~~~~~~~~~~~~~~

* Unpartitioned evidence content::

    {
      "nested": {
        "good_stuff": [
          {"foo": "...", "bar": "...", "country": "US", "region": "IL"},
          {"foo": "...", "bar": "...", "country": "US", "region": "IL"},
          {"foo": "...", "bar": "...", "country": "US", "region": "NY"},
          {"foo": "...", "bar": "...", "country": "UK", "region": "Essex"},
          {"foo": "...", "bar": "...", "country": "UK", "region": "Essex"}
        ],
        "nested_other_stuff": "nested meh"
      },
      "other_stuff": "other meh"
    }

* Partitioned configuration::

    {
      "locker": {
        "repo_url": "https://github.com/my-org/my-evidence-repo",
        "partitions": {
          "foo/evidence_bar.json": {
            "fields": ["country", "region"],
            "root": "nested.good_stuff"
          }
        }
      }
    }

* Partitions yielded

  * US/IL partition - ``foo/<us_il_hash>_evidence.bar.json``::

      {
        "nested": {
          "good_stuff": [
            {"foo": "...", "bar": "...", "country": "US", "region": "IL"},
            {"foo": "...", "bar": "...", "country": "US", "region": "IL"}
          ],
          "nested_other_stuff": "nested meh"
        },
        "other_stuff": "other meh"
      }

  * US/NY partition - ``foo/<us_ny_hash>_evidence.bar.json``::

      {
        "nested": {
          "good_stuff": [
            {"foo": "...", "bar": "...", "country": "US", "region": "NY"}
          ],
          "nested_other_stuff": "nested meh"
        },
        "other_stuff": "other meh"
      }

  * UK/Essex partition - ``foo/<uk_essex_hash>_evidence.bar.json``::

      {
        "nested": {
          "good_stuff": [
            {"foo": "...", "bar": "...", "country": "UK", "region": "Essex"},
            {"foo": "...", "bar": "...", "country": "UK", "region": "Essex"}
          ],
          "nested_other_stuff": "nested meh"
        },
        "other_stuff": "other meh"
      }
